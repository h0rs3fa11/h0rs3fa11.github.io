---
title: Hyperledger Fabric世界状态的篡改
date: 2020-12-13 15:55:53
tags:
---
### 0x01 前言

Fabric的账本数据存储分为不可变的历史数据存储，也就是区块链形式，以文件形式保存；另外是可变的世界状态，用nosql存储当前状态的key-value，fabric支持的世界状态存储数据库有LevelDB和CouchDB。

实验使用fabric first-network测试环境

本文所有查询与调用合约都使用的SDK

### 0x02 测试：修改世界状态

测试所用链码：fabcar

代码位置：[https://github.com/hyperledger/fabric-samples/blob/release/chaincode/fabcar/fabcar.go](https://github.com/hyperledger/fabric-samples/blob/release/chaincode/fabcar/fabcar.go)

测试链码的背书策略都是1-of Org1MSP，即满足Org1的任意节点背书即可

修改账本节点：peer0.org1.example.com

leveldb路径为Mountpoint下的ledgersData/stateLeveldb/，然后通过leveldb数据库查询账本数据(leveldb提供了API)

```plain
CAR0 {"make":"Toyota","model":"Prius","colour":"blue","owner":"Tomoko"}
```
更改CAR0的colour为yellow，更改leveldb数据库时需要关闭当前节点，完毕后再启动
![图片](/image/fabric01.PNG)

用API调用链码fabcar的queryCar函数查询peer0的CAR0信息

![图片](/image/fabric02.PNG)

查询未篡改节点的CAR0信息

![图片](/image/fabric03.PNG)

在peer0上调用fabcar的changeCarOwner函数，接着查询peer0的CAR0，owner已经修改

![图片](/image/fabric04.PNG)

再查询peer1的CAR0，可以看到在随着owner更改的同时，colour变为了peer0的篡改数据

![图片](/image/fabric05.png)

使用一个复杂一点的链码，实现了简单的注册、充值和转账

注册一个新用户test1，初始余额为0

![图片](/image/fabric06.png)

修改peer0的leveldb账本，使得test1的Amount为10

![图片](/image/fabric07.png)

调用链码函数查询peer1的test1余额!

[图片](/image/fabric08.png)

接下来测试被篡改节点peer0发起的交易能否修改其他账本的数据

调用链码paycode函数recharge给账号test1充值20，然后查询各个节点的test1余额；peer0节点毫无悬念的是篡改后的值+20=30，通过查询可以看到peer1节点也是这个值

![图片](/image/fabric09.png)

通过以上测试可以发现，在节点的世界状态被篡改并且背书策略只设置为少量节点背书的话，可以很轻易的修改正常节点的账本。这涉及到Fabric中背书-验证的机制。

### 0x03 交易流程

当客户端要调用链码testcode，需要向多少个节点请求取决于链码的背书策略，在安装-实例化链码的实例化时指定（在2.x中，链码生命周期有所修改，链码的背书策略需要组织认可）。示例背书策略如下：

```plain
policy = {
    'identities': [
        {'role': {'name': 'member', 'mspId': 'Org1MSP'}},
    ],
    'policy': {
        '1-of': [
            {'signed-by': 0}
        ]
    }
}
```
该背书策略仅需要Org1MSP成员的背书
接下来客户端需要向peer0.org1发送背书请求(proposal)，节点peer0.org1接收到请求后会验证交易格式、签名和权限、尚未提交过背书proposal(重放攻击保护)。然后针对peer0.org1当前的世界状态模拟执行交易，产生响应值和读写集。

客户端收到背书响应和读写集后验证peer的签名，并检查是否满足了背书策略（客户端行为可以自定义，但是交易流程中的验证阶段也会进行检查），然后将其提交给orderer，如果proposal仅是查询账本，那么到这一步后将不会再提交给orderer上链，直接返回peer0.org1的查询结果。

排序服务中不需要检查任何交易，仅仅是各个排序节点达成共识将交易排好序后打包成块，然后分发给通道内各个组织的(leader)节点，leader节点将块再广播给其他节点验证。

验证过程中节点验证块内的交易以确保背书策略满足，并确保读集在模拟交易执行时生成以来，读集变量的账本状态没有变化，区块中的交易被标记为有效或无效。然后区块将被写入数据库，并更新状态。

在0x02测试中，产生这种安全漏洞的原因主要在于背书策略的不完善，容错性很差，在仅有一个恶意节点时就可能产生所有节点账本被修改的后果，Fabric的机制让其安全性依赖于背书策略，包括世界状态和读写集，以及背书和验证过程。

### 0x04 Hyperledger Fabric 2.0改进

Fabric 2.0引入了去中心化的chaincode生命周期管理， chaincode需要获得一定数量组织成员的批准才能实例化，与1.x中的单个节点就可将chaincode实例化在通道不一样；并且chaincode的默认背书策略是通道中的大多数组织，提高了本文所述数据篡改的利用难度。


