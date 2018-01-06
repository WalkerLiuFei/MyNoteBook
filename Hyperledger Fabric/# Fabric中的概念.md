# Fabric中的名词解释

+ 成员 (Member) : 在fabric 网络中含有一个唯一根认证文件的实体,网络组件，比如一个 个人节点(peer node) 和 客户端程序会连接到一个 Member上面
+ Ledger (账本) ： 账本记录着当前channel的状态，channel 每个Peer都维护着一个账本
+ Peer : 一个网络实体维护着一个账本且运行一个链码容器，用来执行read / write 账本等操作，Peers 是Member用来维护的
+ A network entity that maintains a ledger and runs chaincode containers in order to perform read/write operations to the ledger. Peers are owned and maintained by members.