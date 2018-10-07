---
title: Mysql的性能优化（一）
---

# 概述

Mysql 的缩影存储支持 Hash索引和BTree索引，**Hash索引只支持  "=","<=>",不支持 > < >=  <= ，between 等比较运算符，Hash**hash索引的时间开销主要发生在建立Hashcode 上，另外，hash索引对于磁盘的的I/O占用会更大。一般是大于O（N）的，而Btree的索引占用的空间只是 O(Log n)
Mysql中每个表的至少支持16个索引，MyISAM和InnoDB的存储引擎表默认创建的数据格式是都BTree，支持前缀索引，不支持函数索引。其中MyIsam支持全文本索引。

索引，简单的可以类比于书的目录。Mysql的索引相当于一种模式，是一个数据库对象，在数据库中加速对表的查询，通过使用快速访问的方法定位数据，减少磁盘的I/O，索引通常与表独立存放，但是其存在依赖于表。

# 设计索引的原则

+ 最适合索引的列是出现在 WHERE 语句中列
+ 使用前缀索引的话，尽量缩短前缀长度。
+ 使用最左前缀
+ 不要过度索引，额外的索引会占用额外的磁盘空间，降低写操作的性能，在修改表的内容时，索引必须进行更新，有必要时要进行重构。
+ 对于InnoDB来讲，记录会按照一定的顺序保存。优先级是 主键 > 唯一索引 > 自动生成的内部列.通常来讲，按照主键和内部列进行访问的速度是最快的。所以尽量选择自己制定主键。当表中同时又多个列都是唯一的时，要选择最常访问键作为主键，提高检查效率。另外，Inn1oDB表的普通索引都会保存主键的键值，所以主键尽量选择较短的数据类型。可以有效的减少索引的磁盘占用，提高索引的缓存效果。 

# 查询语句的优化

1. 尽量避免在where 子句中使用 `!=`和`<>`操作符和null值判，否则存储引擎将放弃	使用索引而进行全表扫描
+ 对查询进行优化，应尽量避免全表扫描，首先应该考虑在where以及order by设计的列上建立索引
+ 应尽量避免在 where 子句中对字段进行 null 值判断，否则将导致引擎放弃使用索引而进行全表扫描，如：

	`select id from t where num is null`

	可以在num上设置默认值0，确保表中num列没有null值，然后这样查询：

	`select id from t where num=0`

+ 尽量避免在where子句中使用`or`来连接条件，否则将导致引擎放弃使用索引而进行全局扫描 如 
 
	 `select id from t where num=10 or num=20`
 	可以使用union 
	
	`select id from t where num = 10` 
	union all 
	`select id from t where num = 20`

 	**注意，前提是 你要在 num 上建立索引，不然两次查表明显没有一次查询来的快**

+ in 和 not in 也要慎用，否则会导致全表扫描，如： `select id from t where num in (1,2,3)` 低于连续的数，能用between 就不要用inselect from t where num betew
+ 避免在where 子句中使用常量，因为sql 只有在运行时才会解析局部变量，但优化程序的不能将访问计划的选择推迟到运行时，它必须在编译时进行选择。`select id from t where num = @num` 改成 `select id from t with(index(索引名)) where num=@num`
+ 尽量避免在where 子句中进行表达式操作

  	`select id from t where num/2 = 100` ----> `select id from t where num = 100*2`
+ 应尽量避免在where子句中对字段进行函数式操作，这将导致引擎放弃使用索引而进行全表扫描
 
	<pre>
	select id from t where substring(name,1,3)=’abc’–name以abc开头的id
	select id from t where datediff(day,createdate,’2005-11-30′)=0–’2005-11-30′生成的id
	</pre>

	----->
	<pre>
	select id from t where name like ‘abc%’
	select id from t where createdate>=’2005-11-30′ and createdate<’2005-12-1′
	</pre>
	更特别的，是否应该在将这些数据读到内存中 再去check,这样的话貌似内存可能不太够用
+ 不要在where 子句中的"="左边进行函数，算术运算或其他表达式运算，否则系统将可能无法正确使用索引
+ 在使用索引字段作为条件时，如果该索引是复合索引，那么必须使用到该索引中的第一个字段作为条件时才能保证系统使用该索引，否则该索引将不会被使用，并且应尽可能让字段顺序与字段顺序保持一致
+ 很多时候 利用exists 代替 in是一个不错的选择
	`select num from a where num in (select num from b)` ---> `select num from a where exists(select 1 from b where num = a.num)`
+ 并不是所有的索引对查询都有效，sql是根据表中数据来进行查询优化的，当索引列有大量数据重复时，sql查询可能不会去利用索引。例如 sex 列只有 female 和male，unknown三种类型，建立了索引，效率也不会有很大改变
+ 索引固然可以提高相应的select的效率，但是对于insert和update 的效率会有很大的影响。因为insert和update时可能会重建索引。索引建立索引数量一般**不要超过6个，为什么是6个**？
+ 如果对文本索引尽量使用短索引，查看B树数据额结构就会明白这样的好处	 
+ 尽量避免更新clustered 索引数据列，因为clustered索引数据列的顺序就是表记录的物理存储顺序，一旦该列值改变，将导致整个表记录的顺序的调整，会耗费相当大的资源。若应用系统需要频繁更新clustered索引数据列，那么需要考虑是否应将该索引建立为clustered索引。
+ 尽量使用数字型字段
+ 尽可能使用 vchar/ nvchar 代替 char / nchar。首先变长字段可以减小空间占用，其次对于查询来说，一个相对较小的字段搜索效率显然更高
+ 尽量不要使用 select * from t ,而用查询具体字段。
+ 尽量使用表变量来代替临时表，如果表变量含有大量数据，请注意索引有限
+ 尽量避免频繁的删除和创建临时表，以减少表资源的消耗，对于一次性时间尽量使用**导出表**
+ 对于创建的临时表，如果插入的数据两量很大，可以选择使用 select into 代替 create table.避免大量的log.以提高速度。
+ 如果使用到了临时表，在存储过程的最后务必将所有的临时表显式的删除，先turncate table，然后 drop table,这样可以避免系统表的较长时间锁定。
+ 尽量避免使用 **游标**，游标的效率较差，如果游标超过一万行，那么就应该考虑改写
+ 使用基于游标的方法或者临时表方法之前，应先寻找基于集的解决方案来解决问题，基于集的方法通常更有效
+ 在所有的存储过程和触发器的开始位置SET_COUNT_ON,在结束时SET_COUNT_OFF,无需在执行存储过程和触发器的每个语句后向客户端发送DONE_IN_PROC消息
+ 避免向客户端发送大量数据
+ 尽量避免大事务操作，提高系统并发能力


## 其他优化策略

+ 利用 explain 语句对查询进行勘查
+ 对于字符串的索引可以考虑前缀索引。就像匹配名字，一般名字很少会有超过10个字符的，那可以吧 一个表的`name`字段的索引设置成为 前十个字符的前缀索引
+ 注意 区别模糊搜索时是否走索引。
	 <pre>
 	SELECT * FROM `houdunwang` WHERE `uname` LIKE'后盾%' -- 走索引
	SELECT * FROM `houdunwang` WHERE `uname` LIKE "%后盾%" -- 不走索引
  	</pre>

+ 通过设置慢查询 的时间阈值来记录慢查询语句，从而进行调优操作
	+ show variables like '%log%'; //查看 log 文件的存储位置
	+ set global XXXX //设置变量 。。
+ Mysql 分区