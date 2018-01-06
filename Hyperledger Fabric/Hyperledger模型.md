# Hyperledger模型

### Channel (通道)
一个通道是一个私有的块链覆盖，允许数据隔离和保密。通道对应的ledger在参与者之间可以共享，另外，每个channel对应的参与者必须充分认证才能参与到channel的交易中，channel的定义在[configuration-block](https://hyperledger-fabric.readthedocs.io/en/latest/glossary.html#configuration-block)中

### Assets（资产）
资产范围从有形（房地产和贵金属）到无形（合同和知识产权）。 Hyperledger Fabric提供使用链码交易来处理资产的能力。

资产在Hyperledger Fabric中表示为键值对的集合，状态更改在channle ledger 上记录为事务。 资产可以用二进制和/或JSON形式表示。

您可以使用Hyperledger Composer工具轻松地定义和使用Hyperledger Fabric应用程序中的资源。

### 交易

交易可以有两种类型，

1. 部署交易创建新的链码，并设置一个程序作为参数，当一个部署交易成功，表明链码已经被安装到了区块上。

2. *调用交易* 是在之前已部署链码的情况下执行一个操作。调用交易引用链码提供的一个函数。当成功时，链码执行特定的函数-它可能涉及修改相应的状态，并返回一个输出。

   ​

### Chaincode  （链码）

链码是定义资产或资产的软件，以及用于修改资产的交易指令。 换句话说，这是业务逻辑。 链码强制读取或更改键值对或其他状态数据库信息的规则。 链码功能针对分类帐当前状态数据库执行，并通过事务提议启动。 链码执行导致一组可以提交到网络并应用于所有对等体上的分类帐的键值写入（写入集）。链码通过Go或者Node.js书写，并运行在一个安全的Docker容器中，通过给予适当的权限，chaincode 可以唤起/访问 同一个网络下的另一个chaincode的状态 [chaincode go api](https://godoc.org/github.com/hyperledger/fabric/core/chaincode/shim#Chaincode)

### Ledger 

账本提供了在系统运行过程中发声的可验证的历史，包含所有成功的状态更改和不成功的状态更改尝试，它由排序服务构建的一个全部有序的交易哈希链块，哈希链强制将全部排序块置入账本，每个块包含一批全部排序交易。这个强制全部排序覆盖所有交易。账本保存在所有peer节点，可选地，保存在排序者的一个子集。在谈论排序时我们说的账本是排序账本，而谈论peer节点时我们说的账本是peer账本。peer账本与排序账本的区别是，**peer节点本地维护一个位掩码来表明隔离有效交易和无效交易**

### Ledger Features (账本特性)

ledger 是一串连续的，不可篡改的在fabric上的所有交易状态记录，状态转换是参与方提交的链码调用（“交易”）的结果。每个事务都会产生一组资产键值对，这些对象将作为创建，更新或删除而提交给分类帐。

分类帐由一个块（'chain'）组成，用于存储块中不变的，有序的记录，以及状态数据库以保持当前的结构状态。每个channel 都会有一个ledger，并且每个成员都会维护一个channel的 Ledger 拷贝 


+ 使用基于键的查询，范围查询和复合关键字查询的查询和更新分类帐
+ 使用丰富的查询语言（如果使用CouchDB作为状态数据库）的只读查询
+ 只读历史查询 - 关键字的查询分类帐历史记录，实现数据来源场景
+ 事务包括在链码（读集）中读取的键/值的版本以及以链码（写集）编写的键/值，
+ 交易包含每个认可同行的签名，并提交订购服务
+ 交易被排序成块，并从订单服务“传送”到通道上的对等体
+ 对等人根据认可政策验证交易，并执行政策
+ 在添加块之前，将执行版本控制，以确保从链码执行时间开始读取的资产状态没有更改
+ 一旦事务被验证提交，就不可更改
+ 通道的分类帐包含定义策略，访问控制列表和其他相关信息的配置块
+ 通道包含会员服务提供商实例，允许从不同的证书颁发机构派生加密资料

### Privacy through Channels（通道的隐私性）
Hyperledger Fabric在每个频道的基础上采用不可变的分类帐，以及可以操纵和修改当前资产状态（即更新键值对）的链码。分类帐存在于通道的范围内 - 可以在整个网络中共享（假设每个参与者都在一个公共通道上运行） - 或者可以将其私有化，只包括一组特定的参与者。

在后一种情况下，这些参与者将创建一个单独的渠道，从而隔离/隔离其交易和分类帐。为了解决希望弥合完整透明度和隐私之间差距的场景，可以仅在需要访问资产状态的对等体上安装链码才能执行读写操作（换句话说，如果链码未安装在对等体上，它将无法正确地与分类帐连接）。为了进一步模糊数据，链码中的值可以使用常用的加密算法（如AES）加密（部分或全部），然后追加到分类帐。



## Security & Membership Services （安全和会员服务）

Hyperledger Fabric支撑着一个交易网络，所有参与者都具有已知的身份。 公共密钥用于生成 参与者，网络组件,以及最终用户或客户端应用程序的相关联的加密证书。 因此，数据可以在更广泛的网络和通道级别上操纵和管理数据访问控制。 Hyperledger Fabric的这种“许可”概念，加上渠道的存在和功能，有助于解决隐私和机密性至关重要的情况。

请参阅会员服务提供商（MSP）主题，以更好地了解加密实现，以及Hyperledger Fabric中使用的签名，验证，验证方法。

### Consensus （共识）

在分布式分类帐技术中，共识最近已成为单一功能中特定算法的代名词。然而，共识不仅仅是简单地同意交易顺序，而是通过其在整个交易流程中的基本角色，从提案和认可到订购，验证和承诺，在Hyperledger Fabric中强调了这种差异化。简而言之，共识被定义为对包含块的一组交易的正确性的全面验证。

当块的交易的顺序和结果符合明确的政策标准检查时，意味着共识已经达成。这些检查和权衡是在交易的生命周期内进行的，并且包括使用认可政策来规定哪些特定成员必须认可一定的交易类，以及系统链式代码，以确保这些政策得到执行和维护。在承诺之前，链上的其他参与者将采用这些系统链码来确保有足够的认可，并且它们是从适当的实体派生出来的。此外，将在分类帐之前将任何包含交易的块包含在交易账单中的当前状态同意或同意之前进行版本控制。此最终检查提供了针对双重支出操作和可能危及数据完整性的其他威胁的保护，并允许对非静态变量执行函数。

除了发生的众多认可，有效性和版本检查之外，还有在交易流程的所有方向上发生的持续身份验证。访问控制列表在网络的层次层次上实现（从订单服务到通道），并且随着事务提议通过不同架构组件，有效负载被重复签名，验证和验证。总而言之，协商一致不仅仅局限于一批交易的商定订单，而是作为在交易从提案到承诺的过程中进行的正在进行的验证的副产品而实现的总体表征。

## Endorsement - 背书

Endorsement 是指一个peer执行一个交易并返回`YES-NO`给生成交易proposal的client app 的过程。chaincode具有相应的endorsement policies，其中指定了endorsing peer。

## Endorsement policy - 背书策略

Defines the peer nodes on a channel that must execute transactions attached to a specific chaincode application, and the required combination of responses (endorsements). A policy could require that a transaction be endorsed by a minimum number of endorsing peers, a minimum percentage of endorsing peers, or by all endorsing peers that are assigned to a specific chaincode application. Policies can be curated based on the application and the desired level of resilience against misbehavior (deliberate or not) by the endorsing peers. A distinct endorsement policy for install and instantiate transactions is also required.

Endorsement policy定义了依赖于特定chaincode执行交易的channel上的peer和响应结果（endorsements）的必要组合条件（即返回Yes或No的条件）。Endorsement policy可指定对于某一chaincode，可以对交易背书的最小背书节点数或者最小背书节点百分比。背书策略由背书节点基于应用程序和对抵御不良行为的期望水平来组织管理。在install和instantiate Chaincode（deploy tx）时需要指定背书策略。

## Gossip Protocol - Gossip协议

Gossip数据传输协议有三项功能：

 	1. 管理peer发现和channel成员；
 	2. channel上的所有peer间广播账本数据；
 	3. channel上的所有peer间同步账本数据。

## Fabric-ca

Fabric-ca是默认的证书管理组件，它向网络成员及其用户颁发基于PKI的证书。CA为每个成员颁发一个根证书（rootCert），为每个授权用户颁发一个注册证书（eCert），为每个注册证书颁发大量交易证书（tCerts）。



