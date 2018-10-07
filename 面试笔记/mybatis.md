---
title: Mybatis 
---

+ Mybatis 查询的的结果类必须要有相应结果属性的构造函数，并且还不能有原生的属性，比如int 需要是 Integer的！

##  Mybatis 中 “#” 和“$”的区别

+ `#{ }` 解析为一个 JDBC 预编译语句（prepared statement）的参数标记符。
 <pre>
	select * from user where name = #{name};
	解析为：
	select * from user where name = ?;
 </pre>
 一个 `#{}` 被解析为一个参数占位符 ？ ，而`${}` 仅仅作为一个**String**的替换出现。在动态sql中会进行变量替换。
也就是说 `${}` 的变量替换阶段是在动态	SQL解析阶段，而 `#{}` 的变量替换是在DBMS中

#### `#{}` 和 `${}`用法 Tips
 + 能使用 `#{}`的地方尽量使用 `#{}`,为了性能考虑，相同的预编译Sql可以重复使用。

 + 避免使用 `${}` ,`${}`在预编译之前已经被变量替换了，这会导致Sql注入的问题。

 <pre>
	select * from ${tableName} where name = #{name}
	假如，我们的参数 tableName 为 user; delete user; #，
	
	那么 SQL 动态解析阶段之后，预编译之前的 sql 将变为
	select * from user; delete user; -- where name = ?;
 </pre>

 上面 #之后的就不会再被执行，这说明，本来好好的一个查询语句，变成了一个删除语句，这只是一个例子，还有更严重的！。。。。。
 
 + 表名作为变量时，必须使用 `${}`,这是因为，表名是字符串，使用sql占位符替换字符串会加上单引号' '，这会导致语法错误
 + 
 

##  Mybatis 中的SQL 预编译

+ Mybatis 的预编译其实就是JDBC的预编译。
 + JDBC的预编译利用的是Preparestament 对象。
 + PreparedStatement 能够提升代码的可读性和可维护性
 + PreparedStatement 尽最大可能提高性能
 + **提高安全性	 ??**
+ JDBC中使用对象 PrepareedStatement来抽象编译语句，使用预编译
 + 预编译可以优化Sql的执行，DBMS不需要再次编译，越复杂的sql，编译的复杂度越大，预编译阶段可以合并多次操作为一个操作
 + 预编译语句对象可以重复使用；**把一个预编译后的preparedStatement对象缓存下来，下次对于同一个sql，可以直接使用这个缓存的PreparedStatement对象。在默认情况下，对所有的Sql进行预编译**
	 
	![](https://segmentfault.com/img/bVtwuY)

### JDBC的预编译是怎样 防止 SQL 注入的？

JDBC的预编译，其实就是在SQL执行之前，在Mysql进行了一次预编译。将查询的参数等，利用占位符填充的形式进行注入。计算出现上面的额那种类型的sql注入攻击。JDBC也能进行过滤。

**简单的讲，就是将参数 和 语句进行了分离，这样就可以避免吗SQL 注入！**

	String sql="update cz_zj_directpayment dp"+"set dp.projectid = ? where dp.payid= ?";
	try {
	    PreparedStatement pset_f = conn.prepareStatement(sql);
	    pset_f.setString(1,inds[j]);
	    pset_f.setString(2,id);
	    pset_f.executeUpdate(sql_update);
	}catch(Exception e){
	    //e.printStackTrace();
	    logger.error(e.message());
	}

**对于怎样提升的性能？ 个人理解就是将语语句进行了缓存。例如查询 `select * from table_name where id > 100`在第一次查询时将语句进行缓存，第二次查询时，使用同样的 查询语句 select * from table_name where id > ? 第二次主需要填充占位符即可！**

## Mybatis中的缓存
### 一级缓存

一级缓存是SqlSession级别的缓存。在操作数据库时需要构造sqlSession对象，在对象中有一个（内存区域）HashMap用于缓存数据。不同的sqlSession之间的缓存数据区域互相没有影响。Mybatis 默认开启一级缓存。当然，让SqlSession close之后一级缓存就没了。

这一级缓存很容易理解，没什么好说的

### 二级缓存

二级缓存是Mapper级别的缓存，多个SqlSession去操作同一个Mapper的Sql语句，多个SqlSession去操作数据库会存在二级缓存区域，多个SqlSession可以共享二级缓存。二级缓存是跨SqlSession的。二级缓存需要在Mybatis中的Setting 选项中设置.其映射区域为整个Mapper。

数据结构仍然为HashMap
### 三级缓存