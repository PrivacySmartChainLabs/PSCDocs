# PrivachSmartChain节点搭建教程

## bootstrap 引导节点搭建

```bash
# 开启相关端口
ufw allow 26656  # 开放p2p端口
ufw allow 26657  # 开放rpc端口

# 创建存放sgx远程证明证书的目录
# 这个证书有因特尔签名的报告
mkdir -p /opt/privacy/.sgx_privacys

# 创建环境变量
# /usr/lib 下存放了三个重要的.so 动态库，与sgx相关
export PRIVACY_ENCLAVE_DIR=/usr/lib
export PRIVACY_SGX_STORAGE=/opt/privacy/.sgx_privacys

# 初始化enclava环境
# 这个命令会生成一个英特尔签名的远程证明证书
pscd init-enclave

# 检查是否有生成证书
ls -h /opt/privacy/.sgx_privacys/attestation_cert.der

# 设置链的id
pscd config chain-id pscdev
# 设置key模式
pscd config keyring-backend test
# 初始化链 banana 是节点别名，可以换
pscd init banana --chain-id pscdev
# 修改 app 中 gas 代币的名字
# 修改 创世文件中的代币名字
# stake -> upsc
# 需要自己修改

# 添加一个账号,助记词会打印在屏幕
pscd keys add a
# 备份助记词
echo "助记词" > a.txt

# 将 a 账号的信息写入创世块配置
pscd  add-genesis-account "$(pscd keys show -a a)"  1000000000000000000upsc

# 生成一笔交易： a 委托第一个验证器
# 注意要加上gas-prive
pscd gentx a 1000000upsc --chain-id pscdev --gas-prices 0.0125upsc

# 将这比交易收集进入genesis.json 中
pscd collect-gentxs
# 验证是genesis.json 是否有效
pscd validate-genesis

# 初始化引导节点
pscd init-bootstrap
pscd validate-genesis

# 创建一个logs文件夹
mkdir logs
# 后台运行
nohup pscd start --rpc.laddr tcp://0.0.0.0:26657 --bootstrap
# orlogs
nohup pscd start --rpc.laddr tcp://0.0.0.0:26657 --bootstrap >./logs/nohup.out 2>&1 & 
# 或者直接运行
pscd start --rpc.laddr tcp://0.0.0.0:26657 --bootstrap
```

## node 一般节点搭建

加入已有的网络

```bash
# ufw 开放防火墙
ufw allow 26656 #p2p
# 创建存放sgx远程证明证书的目录
# 这个证书有因特尔签名的报告
mkdir -p /opt/privacy/.sgx_privacys

# 创建环境变量
# /usr/lib 下存放了三个重要的.so 动态库，与sgx相关
export PRIVACY_ENCLAVE_DIR=/usr/lib
export PRIVACY_SGX_STORAGE=/opt/privacy/.sgx_privacys

# 初始化enclava环境
# 这个命令会生成一个英特尔签名的远程证明证书
pscd init-enclave

# 检查是否有生成证书
ls -h /opt/privacy/.sgx_privacys/attestation_cert.der

# 指向引导节点
pscd config node tcp://167.179.118.118:26657
# 初始化链信息,  hostname 自己取
pacd init $hostname --chain-id pscdev

# 引导节点的p2p种子
PERSISTENT_PEERS="d0c7b5064d67c48c26cc77a1e8a95ad032fe1461@167.179.118.118:26656"
# 将种子写入文件
sed -i 's/persistent_peers = ""/persistent_peers = "'$PERSISTENT_PEERS'"/g' ~/.pscd/config/config.toml

# 打印种子
echo "Set persistent_peers: $PERSISTENT_PEERS"


# 创建一个账号，用户管理这个节点, dafni 是key的名称
pscd keys add dafni

# 有代币的账户给 dafni 转账
pscd tx bank send psc1jvtdv8674llygwn8s0d7z47uctyjf9uxu9p8k4
psc17tuk77p0eqs8nqevwhqs9lrsqm54e56uef6rey 1050000upsc --gas-prices 0.0125upsc

# 注册节点
pscd tx register auth /opt/privacy/.sgx_privacys/attestation_cert.der -y --from dafni --gas-prices 0.25upsc

# 注册成功则可以拿到共享种子
SEED=$(pscd q register seed "$PUBLIC_KEY" 2> /dev/null | cut -c 3-)

# 打印种子信息，看是否为空
echo "SEED: $SEED"

# 设置网络证书，这个命令会生产两个证书
pscd q register secret-network-params 2> /dev/null

pscd configure-secret node-master-cert.der "$SEED"

# 下载创世文件，替换
xxxxx

# 校验创世文建是否有效
pscd  validate-genesis

# 完成注册，将配置指向自己
pscd config node tcp://localhost:26657

# 运行节点
nohup pscd start >./logs/nohup.out 2>&1 & 
```

### node 成为验证人

```bash
# 注意 1 psc = 1000000upsc
  pscd tx staking create-validator \
  --amount=1000000upsc \
  --pubkey=$(pscd tendermint show-validator) \
  --identity=dafni-boxi \
  --details="To infinity and beyond!" \
  --commission-rate="0.10" \
  --commission-max-rate="0.20" \
  --commission-max-change-rate="0.01" \
  --min-self-delegation="1" \
  --moniker=dafni \
  --from=dafni
  --gas-prices 0.25upsc
```

