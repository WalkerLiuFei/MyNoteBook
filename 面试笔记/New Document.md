---
title: 要准备的东西
---

# Java 
+ 牢记JVM的平台无关性
+ java 8 的新语法
+ 线程池实现原理和调度原理
+ Java 线程安全同步代码块和lock的区别，原子性 和 可见性  
+ ClassLoader 双亲委派？ClassLoader 深入理解
+ JVM 编译优化
+ LinkedHashMap 与HashMap的区别在哪里，ConcurrentHashMap哪。Concurrent 在处理Hash竞争时的处理
+ java Nio的概念以及使用，底部实现原理（通道，缓冲区，选择器）
+ java内存模型的理解（深入理解JVM虚拟机）
+ 守护线程
+ 深入反射机制，是怎么 实现的，起码能大概说出其原理
+ Concurrent 包的整体架构，CAS和volatile可见性的变量的读写等等
+ 常用设计模式的深入理解
+ HashTable 和 HashMap的区别，H
+ 六大设计模式原则！
+ 在java下面怎么获取一个dump文件
+ 线程局部变量
+ 熟悉 JVM --xx 几种优化命令
+ java中的序列化与反序列化
+ 为什么Java的hash 会用 31 ? java中 这样计算hash code 出现的hash碰撞的几率会有多大？
+ HashMap中的负载因子是干嘛的？ List中貌似也有
# Mybatis
+ Mybatis 怎么控制事务
+ 
# MYSQL
+ MySQL数据库的几种引擎，不同数据库间的比较
+ 数据库的封锁协议，各种读写锁,乐观锁和悲观锁
+ OUTTER join 和INNER JOIN的区别
# Spring
+ 动态代理的几种方式
+ 为什么CGlib方式可以对接口实现代理？ 
+ Spring中编码统一要如何做（Spring Boot）
+ 依赖倒置，控制反转
+ 怎么做一个Spring 的测试
# Redis
+ Redis面试会问到的问题，CAS操作
+ Redis集群中，如何将一个对象映射到对应的缓存服务器
+ 缓存集群中如果新增一台服务器，怎么才能不影响大部分缓存数据的命中？
+ Redis的优化等
+ 项目中具体是怎样使用Redis的
+ Redis怎么做持久化？
# MongoDB
+ MapReduce在MongoDB中应用
+ MongoDB 的缓存和Redis缓存的区别
# 数据结构与算法
+ <a href="http://blog.csdn.net/cywosp/article/details/23397179">一致性Hash算法</a>
+ 深入了解下B-Tree
+ 几个常用的排序算法 好好复习下
# netty

# 其他
+ Maven的使用
+ 一个服务从发布到消费的过程
+ TCP/IP协议 常问的问题。TCP协议栈？
+ TCP和UDP协议的却别
+ Socket的连接在建立TCP/UDP 连接时的区别
+ RMI与代理模式






+ HashMap(包括但是不限于 Collections的其他数据结构实现)的底层数据结构，以及并发的处理



+ MapReduce的原理
+ 消息中间件


+ Redis缓存失效了怎么办？

+ **注重对基础和编程能力的考察，以及对分布式系统设计和架构的理解。**


+ Spring 中的设计模式
+ Spring bean的生命周期，CGLIB <a href="https://premaseem.wordpress.com/2013/02/09/spring-design-patterns-used-in-java-spring-framework/">Spring中的设计模式</a>
+ Spring IOC 和AOP 概念以及基本使用，实现原理
+ Git的操作
+ Thrift通信协议
+ ，Mysql 锁。
+ <a href="http://blog.csdn.net/hguisu/article/details/6155636">深入理解 java异常机制</a>

<a href = "http://www.cnblogs.com/xrq730/p/5260294.html"> 面试感悟----一名3年工作经验的程序员应该具备的技能</a>
### 基础能力 

基础知识必须要扎实，包括语言基础，计算机基础，算法和基本的Linux运维等

针对Java语言，需要对集合类，并发包，IO/NIO，JVM，内存模型，泛型，异常，反射等都有比较深入的了解，最好是学习过部分源码。

这些知识点都是相通的，在面试中也可以体现，比如集合类的HashMap，

从源码的角度，可以深入到哈希表的实现，拉链法以外的哈希碰撞解决方法，如何平衡内部数组保证哈希表的性能不会下降等；
从线程安全的角度可以扩展到HashTable、ConcurrentHashMap等其他的数据结构，可以比较两种不同的加锁方式，RetreenLock的实现和应用，
继续深入可以考察Java内存模型，Volitale原语，内存栅栏等；
横向扩展可以考察有序的Map结构如TreeMap、LinkedHashMap，继而考察红黑树，LRU缓存，HashMap的排序等知识。

Java方向的中高级职位，会比较重视对虚拟机的掌握，诸如类加载机制，内存模型等，这些在程序的优化和并发编程中都非常重要。

算法方面，基本的排序和查找算法，对递归，分治等思想的掌握。如果算法基础不太好，推荐《编程珠玑》等，每一章都很经典。

另外计算机基础，比如TCP/IP协议和操作系统的知识也是必备的，这些都是大学计算机专业的基础课，也是做开发基本的素养。

### 设计模式

设计模式，造轮子的能力，各种缓存和数据库应用，缓存，中间件技术，高并发和高可用的分布式系统设计

大型互联网公司每天要面对海量的请求，都会考察分布式系统的架构和设计，如何构建高并发高可用的系统，

另外因为用户基数比较大，一个细微的优化可能会给带来很大的收益，所以对一些技术栈的掌握要求都比较深入。

比如对MySQL数据库，需要知道相关的配置和优化，业务上来以后如何分库分表，如何合理的配置缓存，一个经验丰富的服务端开发人员，也应该是一个称职的DBA。

对常用的开发组件，比如中间件，RPC框架等都要有一定的了解，虽然工作中可能用不到我们自己造轮子，但是掌握原理才会得心应手。

这部分知识主要靠工作积累，推荐**《大型网站技术架构与Java中间件实践》**，还有曾贤杰的**《大型网站系统架构与实践》**，

里面对大型网站的演变，服务治理和中间件的使用做了很详细的阐述。

作为业务开发人员，有必要了解压力测试相关的指标，比如QPS，用户平均等待时间等，可以帮助你更好的了解自己的系统。