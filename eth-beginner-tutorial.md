# 建立基于以太坊的私有网络和智能合约

> 本文欢迎转载，转载请标明出处
>
> freewolf 资深IT从业者，关注`微服务`、`区块链`、`敏捷开发`、`前端技术`等，不是大神，只是出于热爱。有问题可以到 https://github.com/freew01f/blog 进行交流。
>
> 发布时间 `BiTCoin #480499`


## 写在前面

最近一段时间一直关注区块链的相关的领域和知识，今天本来想帮助小伙伴建立一个基于以太坊的智能合约Demo，发现很多过去的文档都已经过时了，无法正常工作。那就只能自己造个轮子，弄个版本新一些帮助大家入门。

> 本文以流程`tutorial`为主，不过多去讲技术原理，原理文章网络大把。

## 目标

本文目标如下：
- 建立私有以太坊，设置第一个节点，挖矿
- 完成一笔转账交易
- 建立简单的智能合约
- 建立第二个网络节点

## 环境介绍

无论什么开发都离不开相应的环境，我尽可能将所有软件都升级到最新版本，以下是本文内容相关的环境：
- 操作系统 MacOS 10.12.6
- Geth 以太坊 CLI https://github.com/ethereum/go-ethereum v1.67
- Solidity 智能合约编译器 Version: 0.4.15+commit.8b45bddb.Darwin.appleclang

## 安装

安装`Node.js`，这里不阐述了，源代码自己编译吧。
安装`Geth`，这里直接去官方网站下载最新的可执行程序，复制到`/usr/local/bin`，就OK。
最后安装`Solidity`，本地要先有`brew`，才能进行安装：
```text
brew tap ethereum/ethereum
brew install solidity
```

## 创建区块链

创建自己的以太坊私有链很简单，新建一个目录，在目录中先建立自己的创世区块描述`genesis.json`文件。

文件内容如下

```
{
  "config": {
    "chainId": 2017,
    "homesteadBlock": 0,
    "eip155Block": 0,
    "eip158Block": 0
  },
  "difficulty": "100",
  "gasLimit": "2000000",
  "alloc": {}
}
```

> 为什么自己创建创世区块描述，如果使用默认值，`difficulty`值非常高，这样挖矿要急死人的。

首先创建两个账户，本文后面需要用到这两个账户，创建账户需要输入两次密码。

```
🔥 > geth --datadir node1 account new
🔥 > geth --datadir node1 account new
```

用我们刚刚创建的描述文件，创建创世区块。

```
🔥 > geth --datadir node1 --networkid 27027 init genesis.json
```

使用console连接节点1并且记录`log`

```text
🔥 > geth --datadir node1 --networkid 27027 console 2>>geth.log
```

## 使用geth完成挖矿和交易

连接成功后，看看有几个账户

```
> eth.accounts
["0x65070d1d224114fd3c8358e9614fd948daecc428", "0xf11167054eb5fb91dd7b46726380f0f0cb09a6d8"]
```

查询下账户余额

```
> eth.getBalance(eth.accounts[0])
0
```

第一个账户没有余额，`accounts[0]`默认情况是`coinbase`账户，也就是挖矿的收益归集账户，现在我们就来挖矿，赚取奖励，由于`difficulty`值很低，挖矿秒出基本。

```
> miner.start(2);admin.sleepBlocks(1);miner.stop();
true
```

- `miner.start(2)` 开始挖矿，参数是开启挖矿的计算的线程数
- `admin.sleepBlocks(1)` 挖到1个区块就停止
- `miner.stop()` 挖矿停止

第一次会创建`DAG` ，这里会花费一些时间，关于`DAG`，详情见底部参考。出现`true`说明挖矿完毕，挖完查询余额。

```
> eth.getBalance(eth.accounts[0])
5000000000000000000
```

需要注意，挖一个区块，获得5个`以太币`作为奖励，这里的显示的单位是`wei`，并不是以太币，下面转换一下

```
> web3.fromWei(eth.getBalance(eth.accounts[0]), 'ether')
5
```

5个以太币在`accounts[0]`，现在转2个给`accounts[1]`，转账时候，单位是`wei`，但是注意，既然转账，别忘先解锁账户`accounts[0]`，这里要输入账户密码。

```
> personal.unlockAccount(eth.accounts[0])
Unlock account 0x65070d1d224114fd3c8358e9614fd948daecc428
Passphrase:
true
> eth.sendTransaction({from: eth.accounts[0], to: eth.accounts[1], value: web3.toWei(2, "ether")})
"0x3f190d2af25dff4e6b6fac9537f8f6152fdb193ca9afe228fbdc9301bbff5645"
```

最后出现的是这个交易的`hash`，查一下有没有待处理的交易

```
> txpool.status
{
  pending: 1,
  queued: 0
}
> web3.fromWei(eth.getBalance(eth.accounts[1]), 'ether')
0
```

果然有一笔`pending`，查询账户`accounts[1]`，并没有发现以太币，这里需要旷工来挖矿，打包这个交易到最新区块。交易才能生效，继续挖

```
> miner.start(2);admin.sleepBlocks(1);miner.stop();
true
```

查下余额

```
> web3.fromWei(eth.getBalance(eth.accounts[1]), 'ether')
2
```

已经到账，再看下刚才交易的详情

```
> eth.getTransaction("0x3f190d2af25dff4e6b6fac9537f8f6152fdb193ca9afe228fbdc9301bbff5645")
{
  blockHash: "0xd30fbefb48de05a458a909d9486402bfa4d1459619226a3f8b95aaf407669bb7",
  blockNumber: 2,
  from: "0x65070d1d224114fd3c8358e9614fd948daecc428",
  gas: 90000,
  gasPrice: 18000000000,
  hash: "0x3f190d2af25dff4e6b6fac9537f8f6152fdb193ca9afe228fbdc9301bbff5645",
  input: "0x",
  nonce: 0,
  r: "0xd9b7c4830b9a7ae8ac922179c4e73e6bf2a52178ee0c01250bd940586334d412",
  s: "0xa1b0058b63e1c0360eae6073791b1d63d4a737c71c5932b4b203e853a8185cd",
  to: "0xf11167054eb5fb91dd7b46726380f0f0cb09a6d8",
  transactionIndex: 0,
  v: "0xfe5",
  value: 2000000000000000000
}
```
到这里整个交易就完成了

## 简单的智能合约

下面我们来创建一个极简单的智能合约，`geth 1.6`变化蛮大的，以前编译智能合约的方法都有一些问题，没什么简单的办法，`browser-solidity`是个不错的在线编译选择，我们还是选择在本地进行操作，前面已经通过`brew`安装了`solidity`，创建一个`contract`文件夹，在文件夹中创建一个`hello.sol`智能合约文件

```
pragma solidity ^0.4.13;

contract Hello {
  function sum(uint _a, uint _b) returns (uint o_sum, string o_author) {
    o_sum = _a + _b;
    o_author = "freewolf";
  }
}
```

然后我们来编译，完成后，会多出两个文件，abi文件就是智能合约相关的接口，bin文件就是智能合约编译代码。

> 这里是Mac命令行环境，不是geth，`🔥 > `开头的都是命令行

```
🔥 > solc -o . --bin --abi hello.sol
🔥 > ls
Hello.abi	Hello.bin	hello.sol
```

在`geth`中加载这些文件很复杂，这里我们修改下刚生成的文件

Hello.abi 文件内容修改成

```
var HelloContract = eth.contract([{"constant":false,"inputs":[{"name":"_a","type":"uint256"},{"name":"_b","type":"uint256"}],"name":"sum","outputs":[{"name":"o_sum","type":"uint256"},{"name":"o_author","type":"string"}],"payable":false,"type":"function"}])
```

Hello.bin  文件内容修改成

```
personal.unlockAccount(eth.accounts[0])

var hello = HelloContract.new({
  from: eth.accounts[0],
  data: "0x6060604052341561000f57600080fd5b5b61017a8061001f6000396000f30060606040526000357c0100000000000000000000000000000000000000000000000000000000900463ffffffff168063cad0899b1461003e575b600080fd5b341561004957600080fd5b61006860048080359060200190919080359060200190919050506100eb565b6040518083815260200180602001828103825283818151815260200191508051906020019080838360005b838110156100af5780820151818401525b602081019050610093565b50505050905090810190601f1680156100dc5780820380516001836020036101000a031916815260200191505b50935050505060405180910390f35b60006100f561013a565b82840191506040805190810160405280600881526020017f66726565776f6c6600000000000000000000000000000000000000000000000081525090505b9250929050565b6020604051908101604052806000815250905600a165627a7a72305820063cb95e17166637bd4ab62eae6b0e6c4e1fcd85a9c2e3be29aa75a272280b830029",
  gas: 500000
})
```

> 注意别忘了data必须`0x`开头，New合约就得解锁自己的账户，这里解锁也写在这里了

回到`geth`，加载刚修改的文件，加载`bin`文件需要输入账户密码

> 文件夹`contract`就在运行`geth`命令的目录

```
> loadScript("contract/Hello.abi")
true
> loadScript("contract/Hello.bin")
Unlock account 0x65070d1d224114fd3c8358e9614fd948daecc428
Passphrase:
true
```

现在智能合约已经部署到区块链上了，但是要挖矿才能生效，挖完就可以尽情玩耍了。

```
> hello
{
  abi: [{
      constant: false,
      inputs: [{...}, {...}],
      name: "sum",
      outputs: [{...}, {...}],
      payable: false,
      type: "function"
  }],
  address: undefined,
  transactionHash: "0x783f5cae1f9b40f25da1260267d5e6f801d1746541b5f28f84684883723807b8"
}
> hello.sum
undefined
> miner.start(1);admin.sleepBlocks(1);miner.stop();
true
> hello.sum
function()
> hello.sum.call(1,2)
[3, "freewolf"]
```

## 追加 - 如何建立其他节点

追加一段，如何创建其他的`P2P`节点，首先在原先节点执行下面代码，查看当前节点信息

```
> admin.nodeInfo
{
  enode: "enode://bbd3d0f2afad68c3e4b2e79a6daddc6e8498b9266cc3e953bbb121bae40fe44b5e0377c0768b03e47ac04cead52235a12931bfb96528a8225be5808fc8c174b3@192.168.1.2:30303",
  id: "bbd3d0f2afad68c3e4b2e79a6daddc6e8498b9266cc3e953bbb121bae40fe44b5e0377c0768b03e47ac04cead52235a12931bfb96528a8225be5808fc8c174b3",
  ip: "192.168.1.2",
  listenAddr: "[::]:30303",
  name: "Geth/v1.6.7-stable-ab5646c5/darwin-amd64/go1.8.3",
  ports: {
    discovery: 30303,
    listener: 30303
  },
  protocols: {
    eth: {
      difficulty: 655652,
      genesis: "0x7b0286b147e6b5b8710b8acff38053fdf1991a980da8ca73b4b359c28c7144fc",
      head: "0xd545cee3b9247b67c5d43728eddcbcfe9315dcf18cbc12187a7a178220829153",
      network: 27027
    }
  }
}
```

拿到node相关信息，就可以创建新节点了，这里就不过多解释了，前面基本都介绍过了

```
🔥 > mkdir node2
🔥 > geth --datadir node2 account new
🔥 > geth --datadir node2 --networkid 27027 init genesis.json
🔥 > geth --datadir node2 --networkid 27027 --port 30304 --bootnodes "enode://bbd3d0f2afad68c3e4b2e79a6daddc6e8498b9266cc3e953bbb121bae40fe44b5e0377c0768b03e47ac04cead52235a12931bfb96528a8225be5808fc8c174b3@192.168.1.2:30303" console
```

需要注意的也就是最后一行，写入你自己的node信息，其他的也没什么了，进入`geth`后，数据同步后，可以使用下面命令

```
> eth.getBlock('latest')
> admin.peers
```

以上就可以在第二个节点看到之前节点1的信息了。

## 参考 & 资源
- 关于DAG http://www.hashcash.org/papers/dagger.html
- 智能合约 http://solidity.readthedocs.io/en/develop/solidity-in-depth.html
- 智能合约开发IDE https://ethereum.github.io/browser-solidity
