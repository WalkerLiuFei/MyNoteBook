---
title: 常用Sql语句笔记
---



# 基础

## 查询的高级用法

+ 查询结果通过columName 进行降序排序：**SELECT columnName(s) FROM tableName ORDER BY column_name DESC**
+ `as new`语法可以将查询结果 作为一个临时的新表
`select min(salary) from (select salary from employee order by salary DESC limit 2)  as new limit 2;` 

## 插入

+ 批量插入： **INSERT INTO table (field1,field2,field3) VALUES ('a',"b","c"), ('a',"b","c"),('a',"b","c")**
+ 批量插入时，利用事务可以提高插入效率

## 改变表的信息
+ 改变列名 ；    **alter table table_name change srcColumnName newColumnName**
+ 改变列的类型： **alter table table_name motify srcColumnName newColumnType**  
+ 改变表的名字:  **alter table table_name rename newTableName**
+ index(...列名）创建索引，组合索引，在建表的时候用
+ 改变某个字段的信息：**alter table t modify id int auto_increment;**

## 修饰符

+ zerofill :对于指定的int长度，插入的数据没有充满前面自动fill zero

## 函数
<a href = "http://dev.mysql.com/doc/refman/5.7/en/func-op-summary-ref.html"> MySql处理函数，来自官方文档</a>

+ 字符串处理函数
 + Concat

## 运算符

+ "="运算符不能比较null,要比较null用 "<=>",而"<>"表示两侧不相等 ： **0 <> 1**


## 数据格式

+ float(5,3),double(5,3),以及decima。 decimal默认是decimal(10,0).在Mysql中默认以字符串的形式存储
+ DESC 降序
+ 时间类型
 + DATE:4字节 1000-01-01 9999-12-31
 + DARATIME:8 ,1000-01-01:0000 9999-12-31:23:59:59
 + TIMESTAMP
+ char会删掉尾部的空格，varchar则不会
+ ENUM 枚举类型


## 拷贝

+  将一个表的数据拷贝到另一张表：**INSERT INTO 目标表 SELECT * FROM 来源表;**
+  copy之前，或许需要另一个表：**create table newTable like oldTable**

## 主键 外键约束

外键取值规则：空值或者参照的主键值

+ 插入非空值时，如果主键表中没有这个值，则不能插入。
+ 更新时，不能改为主键表中没有的值
+ 删除主键表记录时，你可以在建外表时选定外键记录一起级联删除还是拒绝删除
+ 更新主键记录时，同样有级联更新和拒绝执行的选择

主键，外键，索引

+ 主键
	+ 标示一条记录，不允许为空，不允许重复
	+ 用来保证数据的完整性
+ 外键
	+ 一个表的外键是另一个表的主键，外键 可以有重复的，也可以是空值
	+ 用来和其他表建立联系使用
	+ 一个表可以有多个外键
+ 索引
	+ 该字段没有重复值，但可以有一个空值
	+ 用来提高索引的速度
	+ 一个表可以有多个唯一的索引

为一个表添加外键

<pre>
alter table 表名
add constraint FK_字段名--"FK"为外键的缩写
foreign key (字段名) references 关联的表名(关联的字段名) --注意'关联的表名'和'关联的字段名'
</pre>

在建表的时候添加指定外键**注意语法合适（逗号）**

<pre>
create table customer(customer_id varchar(50) NOT NULL primary key,
ticket_id varchar(50) not null ,foreign key (ticket_id) references ticket(ticket_id))
 default character set utf8 ;
</pre>
**创建外键关联时注意，reference 的外键必须要类型&&长度一致，两个表的存储引擎必须一致，编码方式一致**
