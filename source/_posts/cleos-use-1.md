---
title: 【EOS开发入门】10分钟学会创建账户、转账及买卖RAM等常用命令
categories: cleos
---

### 搭建环境
> 本文主要介绍EOS命令的使用，搭建环境只作简单介绍，更多信息请查阅 [官方手册](https://developers.eos.io/eosio-nodeos/docs/getting-the-code)。

1、获取EOS代码
```
> git clone https://github.com/EOSIO/eos --recursive
```

2、使用脚本自动安装
```
> cd eos
> ./eosio_build.sh
```

### 配置命令指向主网
1、创建EOS操作命令cleos别名，便于后续命令均指向EOS主网
```
# 进入cleos目录
> cd ./build/programs/cleos

# 创建cleos别名
> alias cleos='./cleos  --wallet-url http://127.0.0.1:8900  -u http://mainnet.eoscalgary.io'
```

`cleos` 是用于与钱包程序和EOS网络进行交互的命令行工具
`--wallet-url` 指定钱包程序运行所在的URL
`-u` 指定EOS网络接入点的URL

> 更多EOS主网接入点URL可以进入 [https://eospark.com/](https://eospark.com/)，打开超级节点信息页面进行查询。

2、获取主网信息
```
> cleos get info
{
  "server_version": "90fefdd1",
  "chain_id": "aca376f206b8fc25a6ed44dbdc66547c36c6c33e3a119ffbeaef943642f0e906",
  "head_block_num": 4141109,
  "last_irreversible_block_num": 4140774,
  "last_irreversible_block_id": "003f2ee6381e87a4089c7cb77698d537d6740a6f1ad9b2f03041bec4e5271dda",
  "head_block_id": "003f303552d77a1dbc63550d5ffceeb7527644a05a86f779f0d9686f71208b1b",
  "head_block_time": "2018-07-04T15:37:41.000",
  "head_block_producer": "eoscanadacom",
  "virtual_block_cpu_limit": 200000000,
  "virtual_block_net_limit": 1048576000,
  "block_cpu_limit": 199900,
  "block_net_limit": 1048576
}
```

> 请确认主网 chain_id 为 
> aca376f206b8fc25a6ed44dbdc66547c36c6c33e3a119ffbeaef943642f0e906

### 创建钱包并导入私钥

1、创建钱包
```
> cleos wallet create -n mywallet
Creating wallet: mywallet
Save password to use in the future to unlock this wallet.
Without password imported keys will not be retrievable.
"<钱包密码>"
```

`-n` 指定钱包名称，如不指定，则默认名称为 `default`
> 钱包一段时间不操作会自动锁定，需要使用创建钱包时生成的钱包密码来解锁，所以请务必记录下钱包密码

2、解锁钱包
```
> cleos wallet unlock -n mywallet --password <钱包密码>
```

3、导入私钥
```
> cleos wallet import -n mywallet <你的私钥>
```

> 对于EOS主网上线前进行了ERC20代币映射的用户，可以导入当时映射的EOS公钥所对应的EOS私钥。对于没有操作映射的用户，可以创建新的密钥对。

4、创建密钥对
```
# 在钱包中创建密钥对
> cleos wallet create_key -n mywallet
Created new private key with a public key of: "<创建的公钥>"

# 查询钱包中的密钥对，过程中需要输入钱包密码
> cleos wallet private_keys -n mywallet
password: [[
    "<创建的公钥>",
    "<创建的私钥>"
  ]
]
```
> 请务必准确抄写并保存创建好的密钥对。

### 创建账户
> 对于EOS主网上线前经过EOS映射的公钥，主网会自动分配一个关联的账户。对于没有账户的用户，可以提供前面创建的公钥，让拥有EOS主网账户的用户或者通过支持创建账户的EOS钱包进行创建。

1、获取公钥所关联的账户
```
> cleos get accounts <你的公钥>
{
  "account_names": [
    "<你的账户名>"
  ]
}
```

2、创建新账户，账户名必须是12位字符（可用字符：12345abcdefghijklmnopqrstuvwxyz）
```
> cleos system newaccount --stake-net '0.01 EOS' --stake-cpu '0.02 EOS' --buy-ram-kbytes 3 <创建者的账户名> <新创建的账户名> <你的公钥>
```

`--stake-net` 换取网络资源所抵押的EOS
`--stake-cpu` 换取CPU资源所抵押的EOS
`--buy-ram-kbytes` 购买的RAM资源，单位为KB

> 创建新账户需要消耗2.93k的RAM，抵押EOS换取网络和CPU资源可以满足转账的需要。

3、获取账户信息
```
# 获取账户信息（权限、资源以及余额等）
> cleos get account <账户名> 

# 获取账户抵押信息
> cleos system listbw <账户名> 
```

### EOS转账
```
> cleos transfer <转出账户名>  <转入账户名>  '0.0001 EOS' '<备注信息>'
```

### 买卖RAM
> 买卖RAM的操作均需要支付0.5%的手续费，买卖的价格通过Bancor算法来决定，关于Bancor算法可以自行百度，这里不展开介绍

1、买入RAM
```
> cleos system buyram <支付方账户名> <获得RAM者账户名> "0.01 EOS"
```

2、卖出RAM
```
# 卖出 4000 bytes 的RAM
> cleos system sellram <账户名> 4000
```


> 需要了解更多的命令，可以查阅 [官方手册](https://developers.eos.io/eosio-cleos/reference) 


