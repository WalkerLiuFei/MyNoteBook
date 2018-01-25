# Bitcoin 名词

## 数据结构

### 区块头

nonce 存在于区块头中，是一个32位长的一个属性，它的值被设置为使得块的散列将包含前导零的运行。其余的领域可能不会变，因为他们有一个明确的含义。

比特币块中的“nonce”是一个32位（4字节）的字段，其值被设置为使得块的散列将包含前导零的运行。 其余的领域可能不会改变，因为他们有一个明确的含义。

块数据的任何改变（例如nonce）都会使块散列完全不同。 由于预测哪个比特组合将导致正确的散列是不可行的，所以尝试了许多不同的nonce值，并且为每个值重新计算散列，直到找到包含所需数量的零 的 的Hash值。 所需的零的位数 由难度设定。 由此产生的哈希必须是一个小于当前难度的值，因此必须具有一定数量的前导零比特。 由于这种迭代计算需要时间和资源，所以具有正确的现时值的块的呈现构成了工作证明。



### Merkle tree

```
d1 = dhash(a)
d2 = dhash(b)
d3 = dhash(c)
d4 = dhash(c)            # a, b, c are 3. that's an odd number, so we take the c twice

d5 = dhash(d1 concat d2)  
d6 = dhash(d3 concat d4)

d7 = dhash(d5 concat d6)
where

dhash(a) = sha256(sha256(a))
```



## P2P网络

参考：[Bitcoin Developer Gulide](https://bitcoin.org/en/developer-examples#p2p-network)

P2P网络的具体实现并不是bitcoin的共识协议的一部分，Bitcoin可以任意使用对应的网络协议和对应的实现，例如Miner使用的 [high-speed block relay network](https://www.mail-archive.com/bitcoin-development@lists.sourceforge.net/msg03189.html)  ，一些SPV 节点使用的  [dedicated transaction information servers](https://github.com/spesmilo/electrum-server) 

比特币网络中的 Full Nodes分为两类

1. Archival Nodes : 保存有所有的历史区块，拥有完整的区块链数据
2. Pruned Nodes ：也是Full node，只不过没有全部的历史区块不保存的全部的区块链数据，在交易验证时，需要Archival Nodes提供的服务
3. SPV Client : 也就是相当一个普通的客户节点

Bitcoin 通过DNS种子来发现 Full Node ，DNS 种子都是hardcode 到Bitcoin Client 软件中的。Bitcoin的正式网络的端口号是`8333`  , `18333` 是测试端口号。 比特币网络中的节点会偶尔失去和网络的链接/ 更换IP地址的等，所以像SPV Client 这样发送一个交易请求 / 验证交易 时，需要等待一段时间。保证能够连接上比特币网络，就像请求中的超时一样。

### 连接到Bitcoin网络

Bitcoin 节点通过发送[版本号信息](https://bitcoin.org/en/developer-reference#version)连接到Bitcoin网络中。对应的节点返回他自己的版本号信息。然后发送[`verack` message](https://bitcoin.org/en/developer-reference#verack) 来完成链接的建立。

### 初始化区块下载

**IBD （Initial Block Download） ** : 在Full Node 验证交易/ 和最近的区块之前，它必须从下载并且验证整个最佳的区块链。

Bitcoin Core 软件在本地保存的区块链的时间戳落后24小时会自动进行IBD 方法。

#### Block-First 

0.9.3版本之前的版本都是Block-First的方式同步的区块，这种最简单的方式进行区块同步



![Block First](https://bitcoin.org/img/dev/en-blocks-first-flowchart.svg)

创世区块是HardCode到 Bitcoin Core软件中的，在进行区块同步时，Peer选择任一个远程节点然后开始同步整个过程是通过[getblocks](https://bitcoin.org/en/developer-reference#getblocks) 命令完成

![](https://bitcoin.org/img/dev/en-ibd-getblocks.svg)

同步节点返回下一个区块的  [`inv` message](https://bitcoin.org/en/developer-reference#inv) 

![](https://bitcoin.org/img/dev/en-ibd-inv.svg)

Inventories 在bitcoin 网路中是独一无二的，每个Inventory都包含有自己的类型和对应的Hash值，对于Block来说，他就是Block的Header的Hash

完成Invetories的接受以后，IBD利用上面获取到的 Block的 Header 的hash 值，维护 一个128队列上进行每个区块的同步。节点通过下面[`getdata` message](https://bitcoin.org/en/developer-reference#getdata) 命令进行数据的获取

![](https://bitcoin.org/img/dev/en-ibd-getdata.svg)

在进行同步时，必须要确保以上步奏要按照区块的顺序来进行，原因你懂得，因为每个区块的区块头都包含有上个区块的Hash。。在IBD node接受到区块后，IBD区块会对区块进行验证，如果出现无法验证的情况，即当前的验证的这个节点的父区块还没有被同步，那么这个区块被称为[orphan blocks](https://bitcoin.org/en/glossary/orphan-block) （孤块）。。

Sync节点通过对上面的响应是：**这就是Sync 节点 响应的一个Block**

![](https://bitcoin.org/img/dev/en-ibd-block.svg)

在请求节点请求完毕之后，IBD 节点会通过继续进行同步，只不过这次扩大了

##### Block First 的总结：

| **Message** | [`getblocks`](https://bitcoin.org/en/developer-reference#getblocks) | [`inv`](https://bitcoin.org/en/developer-reference#inv) | [`getdata`](https://bitcoin.org/en/developer-reference#getdata) | [`block`](https://bitcoin.org/en/developer-reference#block) |
| ----------- | ---------------------------------------- | ---------------------------------------- | ---------------------------------------- | ---------------------------------------- |
| **From→To** | [IBD](https://bitcoin.org/en/glossary/initial-block-download)→Sync | Sync→[IBD](https://bitcoin.org/en/glossary/initial-block-download) | [IBD](https://bitcoin.org/en/glossary/initial-block-download)→Sync | Sync→[IBD](https://bitcoin.org/en/glossary/initial-block-download) |
| **Payload** | One or more [header](https://bitcoin.org/en/glossary/block-header) hashes | Up to 500 [block](https://bitcoin.org/en/glossary/block) [inventories](https://bitcoin.org/en/glossary/inventory) (unique identifiers) | One or more [block](https://bitcoin.org/en/glossary/block) [inventories](https://bitcoin.org/en/glossary/inventory) | One [serialized block](https://bitcoin.org/en/glossary/serialized-block) |

##### Block First 的缺陷

1. 速度限制: 单节点同步，受限于Sync 节点的网速
2. 重新下载：Sync可能并不是最优的Block Chain,这样的话，IBD节点就需要从另一个节点进行同步
3. 磁盘限制：SYNC Node的非最优Blcok Chain 会占用IBD节点的存储空间
4. 内存占用：孤块的存在会导致孤块保存在内存中，因为孤块要等待父节点到底是否存在，并进行验证

#### Headers-First

Bitcoin-core 从0.10.0之后使用header-first 的方式进行区块同步，这种方式的目的是download header chain ，部分验证其有效性，并并行的下载区块头对应的区块。

![Header First同步的概览](https://bitcoin.org/img/dev/en-headers-first-flowchart.svg)

1. 通过 [`getheaders` message](https://bitcoin.org/en/developer-reference#getheaders) 来从[best header chain](https://bitcoin.org/en/glossary/header-chain)进行区块头的同步

   ![区块图](https://bitcoin.org/img/dev/en-ibd-getheaders.svg)

   ​

   Sync 节点的响应：[`headers` message](https://bitcoin.org/en/developer-reference#headers) 最多可以携带2000个区块头的响应：

   ![](https://bitcoin.org/img/dev/en-ibd-headers.svg)

   ​

   在IBD节点下载完部分区块头并进行验证以后，IBD节点即开始同步的进行剩余区块头的同步和区块的同步

   1. 下载更多的区块头： IBD节点循环的使用一个长度为2000的队列进行Header 同步，在完成Header 同步后.IBD节点发送get header命令到其他节点请求Header 并比较已经请求的Header，来验证是否是Best Chain Header。 通过这种方式来验证当前的sync node 是否是一个诚实节点

   2. 下载区块： 在区块头被下载并且验证之后，区块用区块头作为payload 利用getdata命令下载区块，与Block-First不同的是，这在下载的时候是从不同的Full Node下载的。也就是说，IBD下载时不受但一个一个SNC节点的网络状态的影响。在进行区块拉取时，IBD节点同时从8个节点，每个节点维持一个长度为16的的请求队列进行同步。也就是说同时可以请求128个区块。

      ​

   | **Message** | [`getheaders`](https://bitcoin.org/en/developer-reference#getheaders) | [`headers`](https://bitcoin.org/en/developer-reference#headers) | [`getdata`](https://bitcoin.org/en/developer-reference#getdata) | [`block`](https://bitcoin.org/en/developer-reference#block) |
   | ----------- | ---------------------------------------- | ---------------------------------------- | ---------------------------------------- | ---------------------------------------- |
   | **From→To** | [IBD](https://bitcoin.org/en/glossary/initial-block-download)→Sync | Sync→[IBD](https://bitcoin.org/en/glossary/initial-block-download) | [IBD](https://bitcoin.org/en/glossary/initial-block-download)→*Many* | *Many*→[IBD](https://bitcoin.org/en/glossary/initial-block-download) |
   | **Payload** | One or more [header](https://bitcoin.org/en/glossary/block-header) hashes | Up to 2,000 [block headers](https://bitcoin.org/en/glossary/block-header) | One or more [block](https://bitcoin.org/en/glossary/block) [inventories](https://bitcoin.org/en/glossary/inventory) derived from [header](https://bitcoin.org/en/glossary/block-header) hashes | One [serialized block](https://bitcoin.org/en/glossary/serialized-block) |

   也就是说如果不考虑Block-Frist 通过单节点同步时的单节点限制，Block-First 和Header-First的同步速度是一样的

   ​

   ​

   ​

 ### 区块广播

1. 矿工在挖矿成功后，进行区块广播，区块广播分为下面方法：

   1. **Unsolicited Block Push**  ： 矿工直接将新发现的区块发送给Full Node,一般不会使用这种简单粗暴的方式，因为这个矿工并不确定其余节点是否发现新的区块

   2. **standard block relay 区块中转方式：** miner 通过 [`inv` message](https://bitcoin.org/en/developer-reference#inv) 消息将新的区块分发给所有的 Full Node和[SPV](https://bitcoin.org/en/glossary/simplified-payment-verification)节点,miner得到的回复有两种

      1. Each [blocks-first](https://bitcoin.org/en/glossary/blocks-first-sync) (BF) [peer](https://bitcoin.org/en/glossary/node) that wants the [block](https://bitcoin.org/en/glossary/block) replies with a [`getdata` message](https://bitcoin.org/en/developer-reference#getdata) requesting the full [block](https://bitcoin.org/en/glossary/block).
      2. Header-First（HF） **节点回复一个 [`getheaders` ](https://bitcoin.org/en/developer-reference#getheaders) message 来请求 miner的best header chain的最高区块头的Hash, 包括最高区块头的前几个区块头的信息。通过这样的方式来进行分叉检** 紧随其后的，HF节点发送[`getdata` message](https://bitcoin.org/en/developer-reference#getdata) 来获取完整的区块信息
      3. SPV 回复一个[`getdata`](https://bitcoin.org/en/developer-reference#getdata) 信息来获取[merkle block](https://bitcoin.org/en/glossary/merkle-block).

      miner 在收到这些回复后对应的发送相应的Block message 或者 headers message 或者是SPV client 需要的 [`tx` messages](https://bitcoin.org/en/developer-reference#tx) 

   3. 直接通过区块头进行广播：HF 节点在收到区块头广播后进行验证，之后紧接着进行请求区块头

在默认情况下，Bitcoin Core 利用  ` direct headers announcement` 来回复用 [`sendheaders`](https://bitcoin.org/en/developer-reference#sendheaders) 来标记请求的节点，standard block relay 来响应其他的



|             |                                          |                                          |                                          |                                          |
| ----------- | ---------------------------------------- | ---------------------------------------- | ---------------------------------------- | ---------------------------------------- |
| **Message** | [`inv`](https://bitcoin.org/en/developer-reference#inv) | [`getdata`](https://bitcoin.org/en/developer-reference#getdata) | [`getheaders`](https://bitcoin.org/en/developer-reference#getheaders) | [`headers`](https://bitcoin.org/en/developer-reference#headers) |
| **From→To** | Relay→*Any*                              | BF→Relay                                 | HF→Relay                                 | Relay→HF                                 |
| **Payload** | The [inventory](https://bitcoin.org/en/glossary/inventory) of the new [block](https://bitcoin.org/en/glossary/block) | The [inventory](https://bitcoin.org/en/glossary/inventory) of the new [block](https://bitcoin.org/en/glossary/block) | One or more [header](https://bitcoin.org/en/glossary/block-header) hashes on the HF [node’s](https://bitcoin.org/en/glossary/node) [best header chain](https://bitcoin.org/en/glossary/header-chain) (BHC) | Up to 2,000 [headers](https://bitcoin.org/en/glossary/block-header) connecting HF [node’s](https://bitcoin.org/en/glossary/node) BHC to relay [node’s](https://bitcoin.org/en/glossary/node) BHC |
| **Message** | [`block`](https://bitcoin.org/en/developer-reference#block) | [`merkleblock`](https://bitcoin.org/en/developer-reference#merkleblock) | [`tx`](https://bitcoin.org/en/developer-reference#tx) |                                          |
| **From→To** | Relay→BF/HF                              | Relay→[SPV](https://bitcoin.org/en/glossary/simplified-payment-verification) | Relay→[SPV](https://bitcoin.org/en/glossary/simplified-payment-verification) |                                          |
| **Payload** | The new [block](https://bitcoin.org/en/glossary/block) in [serialized format](https://bitcoin.org/en/developer-reference#serialized-blocks) | The new [block](https://bitcoin.org/en/glossary/block) filtered into a [merkle block](https://bitcoin.org/en/glossary/merkle-block) | [Serialized transactions](https://bitcoin.org/en/glossary/serialized-transaction) from the new [block](https://bitcoin.org/en/glossary/block) that match the [bloom filter](https://bitcoin.org/en/glossary/bloom-filter) |                                          |



### 孤块

孤块是指的没有父块的字块

![Difference Between Orphan And Stale Blocks](https://bitcoin.org/img/dev/en-orphan-stale-definition.svg)

节点在接收到孤块后，会到节点请求前面的区块并且验证，知道验证到孤块之前的那个区块



### 交易广播

发送这通过一个 [`inv` message](https://bitcoin.org/en/developer-reference#inv) 来开始交易的广播，当收到`getdata`  命令后，发送着利用`tx` 来发送这个交易。接受者介绍到这个交易后，会利用同样的方式继续进行广播

Full Node 会将这些未完全确认的交易存放在内存中，准备打包进下一个区块中。正是因为这样的原因，如果一笔交易如果一直没有被矿工打包进区块并添加到区块链中，那么这笔交易会慢慢的从整个bitcoin网络中消失, 当最近挖到的区块包含了这些交易，这些交易会从内存中抹去。

