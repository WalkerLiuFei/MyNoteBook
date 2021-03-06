---
title: MongoDB知识点
---

like alaways 这些东西应该实时更新，参照 stackoverflow 上面的问题！


## MongoDB中的一些知识点

+ mongoDB 中collection 的特点，mongoDB collection大小的限制，遇到存储大小限制时应该怎么办？
	+ capped collection 在对Document 数量做限制时，其限制的数量必须小于 2^32, 如果不做限制，就没有限制
+ monogoDB 的存储大小限制
	+ 	对于MMAPv1 存储引擎中，DataBase 中的文件数量不能超过16000个，也就是说一个以MMAPv1 作为存储引擎的data base的最大容量为32TB，如果设置了storage.mmapv1.smallFiles 为true,最大容量为8TB（设置这个bool 值后，单个data file 的最大容量会是 512MB）
	+ 	对于WiredTiger engine ,内部缓存分为两种，一般的RAM 存储，或者256MB（V .34）。
+ mongoDB 中的存储引擎有哪些种，各自有什么特点
	+ MMAPV1  
		+ 存储原理:文档在磁盘中连续存放，文档所占用的磁盘空间包括文档数据所占空间和文档填充，datafile 起始大小为64MB（pre allocate）,当大小超过 64MB后，data file 会通过padding 算法来叠加文档大小
		+ journal 刷新频率为 100ms，jorunal 用来用来恢复数据库的非正常关闭
		+ 很吃内存，因为数据文件会通过mmap 映射到内存中（基本上是有多少吃多少），所以增大机器内存是提高 mongoDB性能的手段之一
	+ WiredTiger engine v3.2 以后的默认存储引擎
	  +  支持文档级别的 lock （`在代码中怎么实现？`）
	  +  快照功能（check point）， 一分钟或者超过2GB的变更日志一次，在写最新快照时，上一个快照已然有效。当 当前快照落地后，会自动删除上一个快照。journal 文件的大小为100MB，当日志文件超过这个大小后，会自动新建一个日志文件
	  +  内部缓存 默认值是：1GB 或 60% of RAM - 1GB，取两值中的较大值
	+ InMemory : 基于内存的存储，不是重点
+ BSON Document 大小限制，有没有 level limit,是多少？
	+ 为避免内存溢出，Bson 最大为 16MB ，如果遇到size 瓶颈，你就需要 GridFS API 来解决
	+ BSON Document 的level 层级最多100.
    
+ mongoDB 是基于内存的还是基于硬盘的
 + 根据存储引擎不同，inmemory 是基于内存。otherwise 基于硬盘的持久化
+ mongoDB 中的GridFS机制，怎么使用
 + mongoDB 将db 分为 多个 chunk 来分块存储大文件。
 + 在使用时需要 基于database 新建 新建一个   `GridFSBucket`，上传文件时需要通过文件流，在java 中也可以使用`GridFSUploadStream ` 来批量上传文件。 上传成功后 会返回一个 fileID,利用这个FileID 实现CURD 等等操作
 + 
+ mongoDB 的数据文件为什么很大
 + mongoDB 为防止数据碎片采用存储空间预分配的策略
+ mongoDB 中chunk的概念，操作块（迁移，删除等等需要注意的事项）
 + chunk 用来存储协作一起来存储大的文件
 + chunk size 默认为64M,chunk 过小会导致频繁的迁移从而导致性能问题，chunk 过大会导致数据存储的不均衡
+ mongoDB 中分片的概念
 + 分片方式： 内存区域分片，hash分片(类似于 数组和hash表)
+ mongoDB 中的日志文件应该怎样存储
+ mongoDB 中的fsync 操作
 + fsync 命令允许我们刷新所有等待写入的操作刷新数据文件中，它提供了锁的机制，这样会使得备份更加简单。
 + fsync 机制并不适合建立一个 read-only 的模式，因为它的读操作也有可能会被block（因为新建立的 connection 需要被授予权限）
+ mongoDB 中**事务**？！ 如何对 collection ，database 进行加锁？
+ mongoDB 中的集群分为那些。各包括那些部分，有什么特点
+ 什么事oplog？ 有什么用
 + 操作日志
+ getlasterror是什么东西？ 有什么用
 + 
+ sharding 和replication 是怎样工作的。
+ 数据在什么时候才会扩展到多个分片里？
+ 当我在视图更新一个正在被迁移的块上的文档时会发生什么？
+ <a href = "https://docs.mongodb.com/manual/reference/limits/">mongoDB 的一些局限性</a>
+ <a href = "https://docs.mongodb.com/manual/reference/program/mongod/#bin.mongod">说出几个常用的mongod 命令</a>
+ mongoDB 中的RDBMS
+ dataBase 是怎样保存 collection的？ collection又是怎样保存document的？ 
+ chunk 默认的大小为 255KB
+ bulkwrite ()
+ 目前很多write operation是用返回值的。包括错误信息

+ Document Model 类型
 + one to one
 + one to many emb


+ read concern ,write concern
 


+ shard 集群，
 + <ul>


 <pre>
    mongod --port 2001 --shardsvr --dbpath shard1/
    mongod --port 2002 --shardsvr --dbpath shard2/
 </pre>


   + 配置服务器：保存集群的元数据，包含各个shard的路由规则\
 
  <pre>
	mongod --port 3001 --dbpath cfg1/
	mongod --port 3002 --dbpath cfg2/
	mongod --port 3003 --dbpath cfg3/
  </pre>

 + 路由服务器：启动查询路由mongos 服务器 -->
	 +  `mongos --port 5000 --configdb 127.0.0.1:3001,127.0.0.1:3002,127.0.0.1:3003`
 
 + 连接mongos，为集群添加数据分片节点。
 	
	<pre>
	mongo --port 5000 amdmin
	
	sh.addShard("127.0.0.1:2001")
	sh.addShard("127.0.0.1:2002")
	</pre>

+ 如果Shard是Replica Set，添加Shard的命令：
	<pre>
	sh.addShard("rsname/host1:port,host2:port,...")	
	rsname - 副本集的名字
	</pre>


 </ul>
 
 



----------------------------以上 需要在 2017/02/07 完成！---------------------------


## mongoDB 常用命令

+ use XXX (用户名，database)
+ db.addUser("name","password")  //添加用户
+ db.getUsers  //查询用户
+ db.`collectionName`.Command : 其中的command 包括 各种对collection 数据的操作。这些也是driver 里面最常用的一些
 + aggreagate  
 + distinct
 + group
 + mapReduce
 + find 
 + insert
 + update
 + delete
 + findAndModify 
 + getMore  // 
 + getLastError ----> db.getLastError
 + getPrevError ----> db.
 + resetError
 + eval // 从 mongo 3.0 开始已经deprecate
 + parallelCollection


## stackoverflow 上的问题

+ 怎样查询 like。 利用regex..find("key","regex 表达式")。。。。其实mongoDB的regex 要比mysql的like 强大的多
+ 利用 ` collection.find(Filters.where("this.stars - 20  > 20" )).forEach((Block<? super Document>) System.out::println );` 可以查询一些复杂的逻辑表达式 的数据
+ mapreduce 
	+ 
+ group
	+ 
+ aggreate

+ replica set 的rollback ，其实是主节点进行写操作，从节点没有同步成功。然后主节点出现宕机，或者其他情况退出了当前的cluster。当这个宕机的主节点从新加入到从节点的后，这个former primary 就成了从节点，从节点需要从当前 的主节点进行一次同步，这就是所谓的rollback。

## mongoDB 官网FAQ

+ mongoDB 保存有cache功能
+ mongoDB 是否支持事务? 
 + mongoDB 不支持在 多个文档上的事务，但是支持在单一文档的上的原子操作,利用$isolated 操作符
 + 原子操作保证在一次读写中，除非出现错误或者异常，其他的client是看不到 的。另外，原子操作
+ 当在建立一个collection上面的Index的时候，collection是不允许读写的。所以建立index的时候尽量将background的属性设置为 true
+ mongoDB的索引和mysql的索引有什么不同？ mysql只能建立一个索引的key，而mongodb 可以建立多个规则更flexable
+ 对于MMAPv1 存储引擎，如果对一个document进行修改导致它存储的空间超过了现有的分配空间。monogodb 会为他新分配一个空间并更新所有indexs.
+ mongodb 的锁机制的类型： Mongodb 使用一个readers-writer 锁，允许多个读操作访问数据库，但是只提供唯一写操作。写操作比读操作的优先级更高
+ 对于MMAPv1 存储引擎，在进行一个读操作的时候，如果当前的document没有在memory中存在，它会将锁ylied，在所要读要读的data被读入内存后，这个操作会再获取这个锁进行操作。
+ CURD ，Indexs,Map-Reduce，aggreate 都会获取锁
+ 那些操作会锁定数据库
 + db.collection.ensureIndex 。在不设置background 为true的时候
 + reindex
 + compact : 把所有的数据和索引重写并分块，在WiredTiger 存储引擎中，这个命令会将硬盘容量重新分配给操作系统
 + 再创建一个很大的collection的时候
 + db.collection.validate() 
 + db.copyDatabase()
+ 那些操作会锁定多个数据库
 + journing
 + uset.auth
+ sharding 中的每个分片都有自己独立的读写锁
+ 在复制集节点中，由于从节点是不允许用户进行写操作的，所以没有必要关心其写的锁。另外就算是对一个cluster 进行写操作，从节点也不是在每时每刻监听主节点的写操作并执行同步，从节点会每分钟(defalut)参考主节点的oplog进行同步
+ 对一个document的一个读操作在进行时，稍后一个对这个读数据的update操作几乎同时发起。虽然读操作能及时看到写造成的document的更新操作的update version，但是无法实时的数据snap shot.也就是说，读操作会错过读操作期间的数据的更新
+ replica Set 和shared 机制应该一起使用，但是replica set架构应该优先于shared机制
+ 一个复制集如果节点的数量是级数，他就不需要投票节点。
+ 在复制集架构中，你可以使用不同的存储引擎
+ 对于wiredTiger存储引擎，当一个机器上运行多个mongod实例 或者mongodb 运行在一个容器中（docker）时你需要降低其cacheSize
+ 工作复制集代表应用在正常操作进程中使用的所有数据，大部分情况下是所有数据大小的集合，但特殊情况下工作复制集的大小取决于当前正在使用的数据库
+ 对于unaccessed 的document，mongod 是不会将其映射到内存中，只要能映射到内存中的数据都是可以accesss的
+ monogdb 每一分钟从memory 写入到disk一次，对于wireTiger，当journal 容量大于2G时其也进行一次write disk 操作，wireTiger 每100ms 写入journal文件一次
+ 默认存储目录 /data/db 中的数据文件可能要比要插入的data set所占空间要打，原因是
	+ 第一：preallocated data file
	+ oplog 文件，oplog 文件大小大概是 disk存储的 5%
	+ journal文件
	
+ repairdatabase 和 compact命令都可以为monogoDB重新分配内存，但是需要注意这两个命令执行的时候都需要2GB 的临时硬盘容量（2GB --->单个datafile的最大容量）
+ 对于 mmapv1 存储引擎collection 没有所谓的容量大小限制，database是有的，database 最多只能包含16000个datafile，每个datafile 最大2GB
+ 内存不足会导致pagefault。原因是内存不足导致的从硬盘读取数据不到内存中，遇到这种情况最好就开始考虑sharding 分片了
+ 一般情况下mongodb是没有连接限制的，除非你对连接进行设置


## mongodb 性能优化

+ 通过打开db.getProfilingLevel(1) 来记录慢命令的日志。默认的慢命令阈值为100 ms
+ 建立索引
+ Nosql于sql的区别
 + Nosql可以水平扩展
 + Nosql 可以对于某些数据类型的存储/获取更加友好
 + Sql的功能更强大
 + Mysql更加稳定，且可以保证数据的原子性和一致性适合大量事务的应用程序。
 + Nosql不太适合复杂的查询
 + 支持缓存机制
+ MongoDB 对于单一的Document的操作总是原子性的

### MongoDB的事务

+ **MongoDB对于事务的支持：**
 + Atomicity:单行/文档级别的原子性，因为MongoDB的操作都是基于文档的
 + Consistency:强一致性，或者最终一致性
 + Isolations：提交读
 + Durability：日志以及复制
+ **对于原子性：**因为Mongo的操作都是基于文档的，如果Mongo的一个“事务操作了多个文档时，中间出了问题 ，它是没有办法回滚的。”
+ **对于一致性：**不同于传统数据的单机版，Mongo更多的应用到分布式的集群上面。对于事务的处理，保证一致性上面显得更加复杂。所以mongo提供了两种一致性： **强一致性和 弱一致性**，通常来讲。最终一致性是更好的选择！
 + **强一致性:**保证集群内所有的机器状态保持一致，
 + **弱一致性（最终一致性）：**允许数据短暂的不一致，但最终所有的数据会一致
+ **Isolation 隔离性：**MongoDB在3.2之后调整为 读已提交，以前你的版本是 读未提交，会出现脏读的情况
+ **持久化**