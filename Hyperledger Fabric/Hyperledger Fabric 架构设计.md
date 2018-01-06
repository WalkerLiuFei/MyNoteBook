# Hyperledger Fabric 架构设计

1. **链码信任和灵活性** ：该架构分离了链码（区块链应用）的信任假设和排序的信任假设。换句话说，排序服务可以由一组节点（排序者）提供，可以容忍一些失败节点或恶意节点，以及**每个链码的背书者可以不同**。
2. **可扩展性** ： 作为特定链码的背书节点和排序者是垂直交叉关系，这可以使系统性能比这些功能都在同一节点上完成要好。特别是，当不同的链码指定不同的背书者时更是如此，为此在背书者之间引入了链码分区技术，允许链码并行执行（和背书）。此外，链码的执行，可能比较耗费计算资源，所以把它从排序服务的关键路径移除。
3. **机密性** ：该架构利于有保密性要求的链码部署，能满足交易内容和状态更新的部署。
4. **共识模块化**： 该架构是模块化的，允许可插拔的共识（即排序服务）实现。

### Nodes

#### Client

client连接一个peer节点，client可以发起/唤起一个交易。client和peers和order service节点都保持着链接

#### Peer

Peer节点负责维护一个账本的拷贝，当Peer节点收到Client的一个交易请求。

Peer节点能附加一个特殊的**背书节点角色**，或**背书者**。*背书节点*的特殊功能是关于特殊链码，存在于提交之前*背书*一个交易。每个链码可以指定一个*背书策略*，可以引用一组背书节点。策略定义一个有效交易背书的必要和充分条件（典型的是一组背书者签名），在后面的第2节和第3节描述。在部署交易的特殊情况下，安装链码（部署）背书策略是由系统链码的背书策略指定。



#### Orderer

 *排序者*产生*排序服务*，即，一个提供交付保证的通信架构。排序服务能以不同的方式实现：从集中服务排序（例如，开发和测试）的分布式协议，指向不同的网络和节点故障模型。

排序服务为客户端和peer节点提供共享的*通信信道*，为包含交易的消息提供广播服务。客户端连接到信道，可以在信道上广播消息，信道随后传递消息给所有peer节点。信道支持所有消息的*原子*传递，意思是，全部排序交付的消息通信和（具体实施）可靠性。换句话说，信道输出同样的消息给所有连接的peer节点并且输出的消息具有同样的逻辑顺序。这个原子通信保证也称为*全部排序广播*，*原子广播*，或是分布式系统中的*共识*。通信消息是包含在区块链状态中的申请交易。

**分隔（排序服务信道）。**排序服务可以支持多个*信道*，类似发布/订阅*主题*消息系统。客户端能够连接到一个给定的信道，然后能够发送消息和获得到达的消息。信道能够被认为是分区-客户端连接到一个信道而没有察觉到其它信道的存在，但客户端可以连接到多个信道。尽管一些排序服务实现包括Hyperledger Fabric v1将支持多信道，为了阐述简单，在本文档的剩余部分，我们假定排序服务包含一个单独的信道/主题。



**排序服务API。**peer节点通过排序服务提供的接口连接到排序服务提供的信道。排序服务API包含两个基本操作（更多是*异步事件*）：

- broadcast(blob): 客户端调用此函数来广播任意消息blob在全信道散播。这在BFT环境下也称为request(blob)，当发送一个请求到服务器时。
- deliver(seqno, prevhash, blob):排序服务在peer节点传送带有非负整型序列号（seqno）和最近广播（blob），也是hash值 的hash值。换言之，它是从排序服务产生的输出事件。deliver()有时在发布/订阅系统也称为notify() ，或在BFT系统中称为commit()。

**账本和块构成。** 账本（见1.2.2）包含了排序服务输出的所有数据。概括地说，它是一系列deliver(seqno, prevhash, blob)事件，根据之前描述的prevhash计算形成的一个哈希链。大多数情况下，出于效率的原因，代替输出单个交易（blobs），排序服务会批量输出blobs，也就是块，而且输出块在一个单个交付事件中。在这种情况下，排序服务必须在每个块内实施和传递blobs的确定顺序。**块内blobs的数量可以由排序服务实现动态选择**。



#### 排序服务特性

排序服务的保证（或原子广播信道）规定了广播消息的发生和交付消息之间存在什么关系。这些保证如下：

1. **安全性（一致性保证）**：只要peer节点连接到信道足够长的时间（它们能够断开或奔溃，但会重启和重新连接），它们会看到交付（seqno,prevhash,blob）消息的*同等*序列。这意味着向所有peer节点输出（deliver()events）*相同排序*，以及根据序列号和为相同序列号携带*同等内容*（blob和prevhash）。注意这仅是一个逻辑顺序，在一个peer节点上的deliver(seqno,prevhash,blob)是不需要与另外一个peer节点输出的deliver(seqno,prevhash,blob)发生任何实时关系。换句话说，给定一个特定的seqno，没有两个正确的peer节点交付不同的prehash或blob值。而且，没有值blob交付除非一些客户端（peer节点）实际是广播（blob），以及更好的，每个广播blob只交付一次。

   此外，deliver()事件包含之前deliver()事件（prevhash）的数据加密哈希。当排序服务实现原子广播保证，prevhash是从序列号为seqno-1的deliver()事件得到的参数的加密哈希，这在第4节和第5节讨论。对于第一个deliver()事件特例，prevhash有一个缺省值。 -----> ` 和比特币的原理基本一样的`

2. 2、**活跃度（交付保证）**：排序服务的活跃度保证由排序服务实现确定。准确的保证可以依赖于网络和节点故障模型。

   原则上，如果提交客户端没有失败，排序服务应该保证每个连接到排序服务的正确peer节点终究交付每个提交交易。

   概括地说，排序服务确保以下特性：

   - *一致性：*对于任何两个具有相同seqno的正确peer节点的事件deliver(seqno, prevhash0, blob0)和deliver(seqno, prevhash1, blob1) , 则prevhash0 == prevhash1，以及 blob0 == blob1;
   - **哈希链完整性。**对于任何在正确peer节点的两个事件deliver(seqno-1, prevhash0, blob0)和deliver(seqno, prevhash, blob), prevhash = HASH(seqno-1||prevhash0||blob0).
   - *没有跳过.* 如果排序服务在正确peer节点p输出deliver(seqno, prevhash, blob) , 这样的话seqno>0, 然后p已经交付事件deliver(seqno-1, prevhash0, blob0).
   - *没有创造.* 任何在正确peer节点上的事件deliver(seqno, prevhash, blob)必须之前一定有一个broadcast(blob)事件在一些(可能是不同的)peer节点上;
   - *没有重复 (可选,但可取).* 对于任何两个事件broadcast(blob)和broadcast(blob’), 当两个事件deliver(seqno0, prevhash0, blob) 和 deliver(seqno1, prevhash1, blob’) 发生在正确的节点 和 blob == blob’, 那么 seqno0== seqno1 和 prevhash0 == prevhash1.
   - *活跃性。*如果正确的客户端调用事件broadcast(blob)那么每个正确的peer节点“最终”发出事件deliver(*, *, blob)，其中*表示任意值。

## 交易背书的基本工作流程

客户端发送一个Propose请求消息到他选择的一组背书节点（可能不是同一时间），给定chaincodeID的背书peer节点的设置由客户端通过peer节点实现，通过背书策略 知道背书peer节点的设置。例如，交易能被发送给所有给定 Channle ID 背书节点。例如交易会通过channel ID被发送到所有的背书节点，那就是说，一些背书者能够离线，其它人可能反对和选择不为交易背书。提交客户端尝试满足背书者可用的背书策略表达。

### PROPOSE消息格式

一个PROPOSE消息的格式是，其中tx是强制的，anchor可选参数在下面列出。

- tx 表示来自于哪里
- clientID 是提交客户端的身份，
- chaincodeID 引用交易相关的链码
- txPayload 是提交交易自身的载体,
- timestamp 是由客户端维护的一个单独递增(为每一笔交易)整型值,
- clientSig 是tx的其它域客户端签名.

txPayload的细节会在调用交易和部署交易之间有所不同（即，调用交易引用部署指定的系统链码）。

对于**调用交易**，txPayload会包含两个域

- txPayload = , 其中
- operation 表示链码操作（函数）和参数,
- metadata 表示调用相关的属性.

对于**部署交易**，txPayload会包含三个域

- txPayload = , 其中
- source 表示链码的源码
- metadata 表示链码和应用的相关属性

policies 包含所有peer节点可访问的链码的相关策略，像背书策略。注意背书策略在部署交易中不支持txPayload，但部署的txPayload包含背书策略ID和它的参数（见第3节）。、

anchor包含*读版本依赖*，或更具体地说，键-版本对（即，anchor是KxN的一个子集），它捆绑或“锚”PROPOSE请求到指定KVS中key的版本（第1.2节）。如果客户端指定anchor参数，背书者背书交易的情况是，只基于读它本地KVS匹配anchor中的相应KEY的版本号（更详细内容见第2.2节）。

tx加密哈希被所有node节点用作唯一的交易标识tid（即，tid=HASH(tx)）。客户端保存tid在内存中，等待背书peer节点的响应。

### 消息模式

客户端决定与背书者互动的顺序。例如，客户端通常会发送（即，没有anchor参数）到一个单独的背书者，背书者随后产生版本依赖（anchor）,客户端可以在晚些时候使用这个版本依赖（anchor）作为它的PROPOSE消息参数，发送给其它背书者。另外的例子，客户端能直接发送（没有anchor）到它选择的所有背书者。不同的通信模式都有可能，客户端在这方面是自由的（也见第2.3节）。

### 背书peer节点模拟交易和产生背书签名

在从客户端接收消息时，背书peer节点epID首先校验客户端签名clientSig，然后模拟一个交易。如果客户端指定了anchor，那么背书peer节点模拟交易只基于在它本地KVS匹配的由anchor指定的版本号对应的key读版本号（即，下面定义的readset）。

模拟一个交易涉及背书节点尝试执行一个交易(txPayload), 通过调用链码到交易引用（chaincodeID）和背书peer节点本地持有的状态拷贝。

作为执行的结果，背书peer节点计算读版本依赖（readset）和状态更新（writeset），也在DB语言中称为MVCC+postimage info。

回顾状态包含键/值对。所有键/值对实体都是版本化的，那就是说，每个实体包含排序版本信息，它是在每次键的值更新时增加的。解释交易的peer节点记录了所有的被链码访问的键/值对，不管读或是写，peer节点不会更新它的状态。更具体地说：

- 在背书节点执行一个交易前给定状态s，被交易读取的每个键k，键/值对(k,s(k).version)被添加到readset。
- 此外，对于每一个被交易编辑的键k到值v’，键/值对(k,v’)被添加到writeset。或者，v’能成为新值与前值(s(k).value)的增量。

如果客户端在PROPOSE消息中指定了anchor，那么客户端指定的anchor在模拟交易时必须等于背书peer节点产生的readset.

它的逻辑部分来背书交易，称为背书逻辑。缺省时，一个peer节点的背书逻辑接受交易提案并简单签署。无论如何，背书逻辑可以执行任意功能，到，例如，与原有系统交互交易提案和tx作为输入来得知是否背书交易。



如果背书逻辑决定背书一个交易，它发送 消息到提交客户端（tx.clientId），其中:

- 交易提案 ：=tran-proposal := (epID,tid,chaincodeID,txContentBlob,readset,writeset), 其中 txContentBlob 是链码/交易专用信息。目的是让txContentBlob 用作tx的一些陈述 (例如, txContentBlob=tx.txPayload).
- epSig 是背书peer节点的交易提案签名。

假使背书逻辑拒绝背书交易，背书者*可以*发送消息(TRANSACTION-INVALID, tid, REJECTED)到提交客户端。

注意背书者在这一步不能改变它的状态，在背书没有影响状态的情况下交易模拟产生状态更新。

### 提交客户端收集交易背书并通过排序服务广播它



提交客户端一直等待直到它在(TRANSACTION-ENDORSED, tid, *, *)上收集到“足够”的消息和签名来推断出交易提案已背书。 “足够”的准确数字取决于链码背书策略（也见第3节）。如果背书策略是安全的，交易已经背书；注意它还没提交。签署TRANSACTION-ENDORSED消息的收集从背书peer节点来，背书peer节点建立了交易是背书的称为背书并以背书为名称。

如果提交客户端没有设法为交易提案收集背书，则放弃这个交易，稍后再试。

对于一个具有有效背书的交易，我们现在开始使用排序服务。提交客户端使用broadcast(blob)调用排序服务，其中blob=endorsement.如果客户端没有能力直接调用排序服务，它可以通过它选择的peer节点代理广播。这样的peer节点必须被客户端信任不会从背书移除任何消息或其它可能被无效的交易。注意一点，无论如何，代理peer节点不可能制造有效背书。



### 排序服务向peer节点提交交易

当一个事件(seqno, prevhash, blob)发生并且一个peer节点已为所有序列号低于seqno的blosbs更新状态，peer节点执行如下流程：

- 它检查blob.endorsement是有效的，根据的是它引用的链码(blob.tran-proposal.chaincodeID)。
- 在典型情况下，它也验证了依赖(blob.endorsement.tran-proposal.readset)在期间没有被违反。在更复杂的用例中，背书中的交易提案域可能不同，在这种情况下，背书策略（第3节）指定状态如何形成。

依赖的验证能以不同的方式实现，根据一致性属性或为状态更新选择的“孤立保证”。**Serializability**是一个缺省的孤立保证，除非链码背书策略指定一个不同的。Serializability能够通过在readset中的每个key关联的版本被提供，相当于key在状态中的版本，并拒绝不满足这个要求的交易。

- 如果所有这些检查通过，交易被视为*有效*或*承诺*。在这种情况下，peer节点在PeerLedger用1标记交易，适用于blob.endorsement.tran-proposal.writeset区块链状态（如果交易提案是相同的，其它背书策略逻辑定义了函数处理blob.endorsement）。
- 如果blob.endorsement背书策略验证失败，交易无效，并且peer节点在PeerLedger的位掩码用0标记交易。重要的是要注意无效交易不会改变状态。

注意，这里有足够的让所有（正确）peer节点在处理一个给定序列号的deliver事件（块）之后具有同样的状态。即，通过排序服务的保证，所有正确的peer节点会收到相同的deliver(seqno, prevhash, blob)事件序列。当背书策略的评估和readset中版本依赖的评估是确定的，所有正确的peer节点也会得出相同的结论，关于包含在blob中的交易是否有效。因此，所有peer节点提交和应用同样交易序列并用同样的方式更新它们的状态。![](https://hyperledger-fabric.readthedocs.io/en/release/_images/flow-4.png)



## 背书策略

### 背书策略规范

**背书策略**，是背书一个交易的条件。区块链peer节点有一组预先确定的背书策略，它被安装特定链码的部署交易引用。背书策略能参数化，这些参数能被部署交易指定。

为了保证区块链和安全特性，背书策略组**应该是一组验证过的策略**，具有有限功能，为了保证有限的执行时间（终止），决定、性能和安全保证。

背书策略的动态添加（即，在链码部署时间由部署交易添加）是对背书评估时间限制（终止）、决定、性能和安全保证非常敏感的。因此，**动态添加背书策略是不允许的，但将来能支持**。

### 针对背书策略的交易评估

交易只有经过根据背书策略的背书才会宣布有效。对于链码的调用交易首先需要的到一个满足链码策略的背书不然的话这个调用交易的操作就是无效的。这通过在提交客户端和背书peer节点之间的互动发生。

正式的背书策略是以背书为基础，以及潜在的进一步评估为真假状态。对于部署交易，获得背书的依据是系统系统范围策略（例如，来自系统链码）。

背书策略断言引用一定的变量。潜在可能引用的是：

1. 与链码有关的键值或ID（存放在在链码元数据），例如，一组背书者；
2. 更多的链码元数据；
3. endorsement and endorsement.tran-proposal的元素；
4. 其他更多变量

`背书策略断言`的评估必须是确定的。背书在每个peer节点的本地进行评估，这样这个peer节点就不需要和其它peer节点进行交互来确定背书，所有正确的peer节点都以相同的方式评估背书策略。

### 背书策略例子

`背书策略断言`可以包含逻辑表达式和评估真假。通常情况会对背书节点为链码发出的交易请求使用数字签名。

假定链码指定背书者集E = {Alice, Bob, Charlie, Dave, Eve, Frank, George}.一些例子策略如下：

1. 来自E的所有成员的同一个提案中的有效签名
2. 一个有效签名来自E的任一单个成员。
3. 从背书peer节点来的同一交易提案的有效签名条件是：(Alice OR Bob) AND (any two of: Charlie, Dave, Eve, Frank, George).
4. 同一提案的有效签名为7名背书者的任意5名。（更常用的，链码n>3f背书者，n名背书者有任意2f+1有效签名，或任意大于(n+f)/2背书者小组有效签名）
5. 假定背书者有一个“股份”或“权重”的任务，像{Alice=49, Bob=15, Charlie=15, Dave=10, Eve=7, Frank=3, George=1}, 其中全部股份是100：策略需要一组占大多数股份的有效签名（即，一组合并股份完全超过50），像{Alice, X}，X只要不是George的任何人，或{除去Alice以外的所有人}，等等。
6. 假定前面例子中的股权条件是静态的（固定在链码的元数据中）或动态的（例如，取决于链码的状态和在执行中修改）。
7. 交易提案1的有效签名来自(Alice OR Bob) 和交易提案2有效签名来自（Charlie, Dave, Eve, Frank, George中的任何两个），其中交易提案1和交易提案2的不同只在它们的背书peer节点和状态更新。

##  证实账本和节点账本检查



为了维护一个只包含有有效交易账本的抽象，peer节点可以处理维护状态和账本外，还可以维护一个有效的账本。这个所谓的账本其实就是区块链。利用这个区块链来排除非法的交易提交

证实账本块的生成按如下顺序。当节点账本块可能包含无效交易（即，交易的背书无效或版本依赖无效），这样的交易被peer节点在交易从块变为证实块之前过滤掉。每个peer节点自身实现这点（例如，使用节点账本关联的位掩码）。证实块被定义为没有无效交易的块，是进过过滤的块。这样证实块在大小上是动态的也可能是空的。证实块生成的说明在下图中给出。

所以Peer的的参与要保证是经过验证允许的节点，保证这个Peer不会作弊！

![从节点账本块变为有效账本块](https://hyperledgercn.github.io/hyperledgerDocs/img/blocks-3.png)

证实块被每个peer节点链接在一起形成一个哈希链。更具体地，证实账本的每个块包含

- 前证实块的哈希。
- 证实块编号。
- 从上一个证实块被计算出以来所有peer节点提交交易的排序列表（即，在相应块中的有效交易列表）。
- 相应块的哈希（在节点账本中），来自得出的当前证实块。

所有这些信息都被peer节点级联和哈希，产生证实账本中证实块的哈希。

### 节点账本检查

账本包含的无效交易，没有必要永久记录。然而，一旦建立相应的证实块，peer节点不能简单地丢弃节点账本块从而修剪节点账本。即，在这种情况下，如果新的peer节点加入了网络，其它peer节点不能转移丢弃块（与节点账本有关的）到新加入的节点，也不能使新加入的peer节点承认它们的证实块。

为了便于节点账本修剪，这个文档描述一个检查点机制。这个机制建立了证实块的有效性，贯穿节点网络，允许检查点证实块替换丢弃的节点账本块。这，反过来，减少了存储空间，因为没有必要存储无效交易。它也减少了新加入的peer节点重构状态的工作量（当通过重演节点账本重构状态时，因为他们不需要建立有效的单个交易，但可以简单重演包含在节点账本中的状态更新。）

检查点是由peer节点每个CHK块周期性地形成，这里CHK是一个可配置参数。开辟一个检查点，peer节点广播（例如，传播）给其它peer节点 , 其中，blockno是当前块编号，blocknohash是各自的哈希，stateHash是最新状态的哈希（产生于，例如Merkle hash），基于确认的块编号，peerSig 是peer(CHECKPOINT,blocknohash,blockno,stateHash)的签名，引用了证实账本。

peer节点收集CHECKPOINT消息直到它得到匹配blockno, blocknohash 和 stateHash 的足够正确的签名消息来建立一个有效的检查点。（见4.2.2节）

在为块编号blockno 和 blocknohash建立了有效的检查点的基础上，peer节点： - 如果 blockno>latestValidCheckpoint.blockno, 那么peer节点分配 latestValidCheckpoint=(blocknohash,blockno), - 存储各peer节点的签名集，它构成了有效的检查点到集合latestValidCheckpointProof, - 存储状态相应的stateHash 到 latestValidCheckpointedState, - （可选的）修剪它的节点账本到块编blockno (包含).



### 4.2.2. 有效检查点(Valid checkpoints)

显然，检查点协议增加了下面的问题：peer节点什么时候能修剪它的节点账本？多少检查点消息是足够多的？这由检查点有效策略定义，要有（至少）两种可能的方法且也能合并：

- Local (peer-specific) checkpoint validity policy (LCVP). A local policy at a given peer p may specify a set of peers which peer p trusts and whose CHECKPOINT messages are sufficient to establish a valid checkpoint. For example, LCVP at peer Alice may define that Alice needs to receive CHECKPOINT message from Bob, or from both Charlie and Dave.
- Local (peer-specific) checkpoint validity policy (LCVP).给定peer节点p上的本地策略可以确定一组peer节点，这一组peer节点是p信任的且它的CHECKPOINT消息是足够建立一个有效的检查点。例如，在peer节点Alice上的LCVP可以定义本地（peer确定）检查点有效性策略（LCVP）。
- Global checkpoint validity policy (GCVP).检查点有效策略可以确定为全局的。这类似于本地节点策略，除非在系统链间隔上规定，好于节点间隔。例如，GCVP可以指定：
  - 每个peer节点可以信任一个由11各不同peer节点确认的检查点。
  - 在具体部署中每个排序者与peer节点配置在同一台机器上（即，信任域），多达f个排序者可以是（拜占庭）错误，每个peer节点可以信任一个检查点，如果经过f+1个排序者配置的不同的节点确认。