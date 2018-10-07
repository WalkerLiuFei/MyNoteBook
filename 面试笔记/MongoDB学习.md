---
title: MongoDB学习笔记
---

# 基本的使用

+ mongodb 的MongoDBClient对象表示一个MongoDB的连接池，就算你有多个线程的操作，你也只需要一个对象
+ MongoClientOptions可以配置数据库连接的选项。其有类似一个静态的工厂类，可以指定，**应用名，最大连接数，最小连接数，服务选择的超时时间，解码器，编码器等等。**
+ MongoDB默认连接是通过SSL加密的（Java 7 以上！），如果你必须使用java 6，或者更低的java版本，你必须通过MongoDBClientOptions的`sslInvalidHostNameAllowed(true)`来disable ssl加密！
+ MongoDB的标准URI格式 : `   mongodb://[username:password@]host1[:port1][,host2[:port2],...[,hostN[:portN]]][/[database][?options]]`,
	+ 首先前缀 `mongodb://`  这是用来标示mongoDB URI的标示符。
	+ `[username:password@]` 是可选项
	+ host1 是唯一必须的部分。
	+ ：port 是可选项，默认为：27017
	+ `/database` 是登录的数据库名字。这个选项只有在 前面 `[username:password@]`使用时才会使用得到。如果没有的话，默认为admin database 会被使用。

+ 对于Capped Collections ，指的是指定容量的collections,类似于 java中 lrucache,根据插入的数据顺序来进行覆盖存储。Capped Collections 的插入速度更高，并且你无法从一个 capped collections 中移除一个document。collections 的最小cap 是4096 byte。mongodb 会确保 query 的 index 等同于 insert 的 index.
+  mongoDB的三种codec 类。
	+  Document since 3.0
	+  BasicDBObject
	+  BsonDocument
	
+ MongoDB 的Index是在collection 级别上创建的。其类似于Mysql 其默认的index 是"_id"。也就是说排序的标准是按照插入的顺序来的。
 
<pre>
	//创建一个 递增的key
  collection.createIndex(Indexes.ascending("stars"));
	//默认的query 的顺序依旧是按照 "_id"的值，也就是说插入顺序来的
   collection.find().forEach((Block<? super Document>) System.out::println);
	// 但是如果你加上 查找的限制字段。他就会按照字段"stars" 来进行排序显示！ 
   collection.find(Filters.gt("stars",0)).forEach((Block<? super Document>) System.out::println);
    // 或者
    collection.find().sort(Indexes.descending("stars")).forEach((Block<? super  Document>) System.out::println);
</pre>

+ Text Index ..一个collection 至多含有一个Text Index。

+ projections(Projections.XXX)。。。。 
` collection.find().projection(Projections.fields(Projections.include("stars"),Projections.exclude("_id"))).forEach((Block<? super Document>) System.out::println);`
 projections 内部的fileds 都是Projections 里面一些静态方法。
+ 同上，MongoDB里面的包`package com.mongodb.client.model;`里面都是一些类似工具类的类。可以用来进行相应的操作
+ 类似projections的还有sort。。

# MongoDB Cluster

+ **主从架构：**搭建一个主从数据库， slave 会同步master 所有 write的操作。下面是 shell 命令。。`注意在本机测试时 需要在不同的命令行窗口打开master 和 slave`
<pre>
#创建一个主从MongDB服务器
cd /

e:

mkdir monogoDBCluster
cd ./monogoDBCluster
mkdir Master
mkdir Slave

mongod --port 12002 --master --dbpath ./Master 
# 源master 来自于 127.0.0.1:2000 。
mongod --port 12003 --slave --source 127.0.0.1:12002 --dbpath  ./Slave


</pre>

+ **副本集架构**：显示启动三个 mongoDB 的实例，并同时制定为 replSet 的名称为 rs0; 

 <pre>
#
mkdir replicaSet_1
mkdir replicaSet_2
mkdir replicaSet_3

cmd /k start mongod --port 12004 --dbpath replicaSet_1 --replSet rs0
cmd /k start mongod --port 12005 --dbpath replicaSet_2 --replSet rs0
cmd /k start mongod --port 12006 --dbpath replicaSet_3 --replSet rs0
</pre>

然后选择其中一个实例进入 其shell命令界面。初始化replSet集群，并添加secondary。 即，当前这些命令的这个实例会被作为primary。

<pre>
mongo --port 12004

rs.initiate()
rs.add("<hostname>:2002")
rs.add("<hostname>:2003")
rs.conf()
</pre>

所有的连接什么的都会在这个上面。在默认情况下，secondary是不允许读和写的。如果你想在secondary 节点上进行读。需要进行相应的设置
在java下面三个等级的设置
<pre>
		//在client 级别。read 首先是从 secondary 上面进行读
  MongoClientOptions options = MongoClientOptions.builder().readPreference(
                                  ReadPreference.secondary()).build();
	//类似的！
  MongoClient mongoClient = new MongoClient(
    new MongoClientURI("mongodb://host1:27017,host2:27017/?readPreference=secondary"));
// database 级别的设置
MongoDatabase database = mongoClient.getDatabase("test")
                         .withReadPreference(ReadPreference.secondary());
//最后 是collection级别的设置
MongoCollection<Document> collection = database.getCollection("restaurants")
            .withReadPreference(ReadPreference.secondary());
</pre>
