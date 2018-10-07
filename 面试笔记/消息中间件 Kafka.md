---
title: 中间件
---
# 消息队列框架 kafka

+ 关键词： `Producer`,`Consumer`,`Broker`,`Topic`,`Offset`
 + **Topic :** 主题，由用户定义并配置在Kafka服务器上，用于建立生产者和消费者之间的订阅关系。生产者生产消息到这个Topic下，消费者从这个Topic下取消息进行消费
 + **Broker :** Kafka的服务器，用于保存消息队列，Kafka集群中一台服务器就相当于一台broker
 + **Paritions :** Topic 物理上的分组，一个 Topic可以有多个Parition，每个parition是一个有序数列，并且其值必须要从0开始,相当于一个文件	
 + **Offset :** 相当于Parition中的消息的“键”
 + **Segment :** Partion 保存数据文件的最小单位，Partion 对其管理的时候都是以Segment 作为单位进行管理的。比如对Older 消息进行删除等操作  


![](http://img.blog.csdn.net/20160421172632084?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

# From Interview

+ **Offset的角色是什么：**
 + 偏移量：
+ **ZooKeeper在Kafkaserver中的角色是什么**
 + 管理消费者和生产者
+ **Kafka集群，Replicat**
 + 为保证Kafka Server的高可用性，需要为Kafka添加Replicat 从机。
+ **当插入一个过长数据导致`InvalidMessageSizeException` 时怎么办**
	+ kafka-topics.sh --zookeeper localhost:13003 --alter --topic MyTopic --config retention.ms=1000
+ **Kafka中消息被消费后会不会立即删除？**
 + Kafka中消息被消费后不会被立即删除，
+ **Partitions的作用**
 + 通过分区，将日志内容分散保存到不同的server上。避免文件尺寸达到磁盘上限
+ **AsyProducer 和sync producer 的区别**
 + 区别在于 sync producer 的send 方法会阻塞线程，sync 不会。
+ **安全，SSL加密**
+ **Log数据写入磁盘的策略**
 + 通过消息数量 
 + 通过时间间隔

# From StackOverflow

+ **怎么删除一个 Topic** ： 怎么删除一个 :`kafka-topics.sh --delete --zookeeper localhost:2181 --topic your_topic_name
` 
+ **如果我有多个Broker，我是否需要手动来负载均衡，为什么消费者只需要一个Zookeeper的end point 而不是一个Broker的 end piont**
 + 

# <a href = "https://cwiki.apache.org/confluence/display/KAFKA/FAQ#FAQ-Isitpossibletodeleteatopic?">From FAQ</a>

+ **什么时候你的Record 会包含一个时间戳**
+ **当出现一个QueueFullException时，你又不想数据丢失。应该怎么做？**
 + 将生产的enqueue Timeout 时间设置成 -1，这样会导致producer 被堵塞，但是不会丢失丢失数据
+ **当我的kafka服务器没有接收到生产者发送的数据，会是什么问题？**
 + 问题多了去了。
+ **Partion 是什么 ？ Partition分区策略**
 + 利用 Hash作为分区策略
+ **删除Topic 之前需要做的操作**
 + 需要将 Topic mark为 delete
+ **如果消费者一直没有接收到数据，会是什么问题**
 + Consumer 只会接收在启动之后 发送到broker上的数据，也可以通过设置`auto.offset.reset`来设置
+ **生产者出现invalidMessageException会是什么问题**  参考这个<a href ="http://stackoverflow.com/questions/21020347/kafka-sending-a-15mb-message"> stackover flow </a>
 + 可能的原因是 fetch size 太小、然而数据太大，合理的配置 好 consumer 的 fetch.max.byte的大小，或者注意生产者发送数据量的大小
 + 为了避免这个问题 ： 最好就在 broker 端将 message.max.bytes 的大小 设置成大于consumer 端的 message.max.bytes。这样的话broker 接收的数据都能被 consumer 正确的消费
 + 在集群中，只有那些 被正确 replicat的消息会被正确到达 consumer 端，所以你也搞保证 broker `replica.fetch.max.bytes`的设置正确。、
 + 保证 consumer端 `fetch.message.max.bytes` ， broker 端 `replica.fetch.max.bytes`,`message.max.bytes`都设置正确！
+ **Consumer的group id 是属于整个集群的**
 + 稳定状态下每一个Consumer实例只会消费某一个或多个特定 Partition的数据，而某个Partition的数据只会被某一个特定的Consumer实例所消费
+ **Kafka读取特定**
+ **Kafka 在传输大文件时是怎么做的 ？**
 + 通过对较大数据进行压缩，Producer实现压缩，Consumer实现解压缩，这样可以解决网络传输带来的瓶颈，但是会对cpu造成负载
+ **Prducer 发布消息的流程**
 1. producer 先从 zookeeper 的 "/brokers/.../state" 节点找到该 partition 的 leader
 2. producer 将消息发送给该 leader
 3. leader 将消息写入本地 log
 4. followers 从 leader pull 消息，写入本地 log 后 leader 发送 ACK
 5. leader 收到所有 ISR 中的 replica 的 ACK 后，增加 HW（high watermark，最后 commit 的 offset） 并向 producer 发送 ACK
+ **Producer 发送一条消息到Broker会不会被重复发送多次？**
 + 由于网络原因，会发送多次。这是Kafka的一个机制，也就是说保证消息至少发送一次
+ **Consumer消费消息时有那两套机制？** 
 + **High-level Consumer Api ：** 这个API 具有Consumer Group 语义，一个消息只能被group内的一个Consumer所消费，且Consumer消费消息时 不关注 **Offset**。
 + **The SimpleConsumer API:** 需要自己跟踪Offset，从而决定下一个消费的消息在哪里，Consumber需要通过程序获知每个Parition的leader是谁，需要处理Leader的变更
 + **最终就是 一个获取消息时需要指定Offset 一个不需要指定 Offset**
+ **Consumber Group：** 每个Consumer 都属于一个Consumer group ，一个 parition只能被同一个Group内的一个Consumer 所消费（保证一个消息只能被group 内的一个Consumer 消费 （消息都是以Topic区分的）！）。  
+ **如果某些group 中的 Consumer没有接收到消息，会是什么原因？**
 + 可能是Parition 不太够的原因，因为Consumer Group 对应关系是 Topic，Group中每个Consumer 对应一个Parition,如果有的Consumer没有接收到消息，说明应该增加Parition的数量了
+ **如果你的Consumer 忽然挂了，下次重新启动以后，Consumer 会发生什么？**
 + Consumer 会接着从挂的地方的offset接着消费，因为从哪里开始的Offset并没有消费，Consumer 自己！！！！
+ **怎样避免重复数据**
 + **在生产者这边**
  +  在每次得到网络错误的时候，检查最后写入的消息是不是现在正要发送的消息  
  + 在消息中包含UUID作为header ，在消费者这边检查这个header，避免重复
 + **在消费者这边**
+ **ZooKeeper在Kafka中的作用？**
 + 服务宕机检测
 + 数据的分区
 + 数据备份
+ **Customer 查询消息的时间复杂度为什么是常量级的**
 + 我认为查询的时间复杂度并不是常量级的应该是 O logN, 在传输中 topic 每个Partion将消息保存Segment。Segment的文件名称为保存消息开始的那个Offset。当Customer通过一个Offset查找消息时，通过二分法首先定位到这个Segment文件，又因为Segment文件的大小是固定的 所以在Segment中查询消息的时间复杂度是常量级的。所以我感觉应该是常量级的
+ **如果一个Producer 要发送一个大文件，你要怎么做？**
 + 先将文件进行压缩，另外保证broker能接受的最大字节数和 Replication最大复制字节数，Consumer能接收的最大字节数大于这个值

# 重要的参数配置

## Broker 端

+ **log.flush.scheduler.interval.ms**	检查数据是否要写入到硬盘的时间间隔。
+ **message.max.bytes**		一个socket 请求的最大字节数、
+ **fetch.message.max.bytes**		控制在一个请求中获取的消息的最大字节数。


 


