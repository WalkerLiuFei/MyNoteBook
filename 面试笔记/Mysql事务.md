---
title: 事务
---

# Transactional注解

+ 利用`@Transaction`注解来进行事务管理
	<pre>
	   @Transactional
    public void insertBouncePeron(){
        SqlSession sqlSession = MybatisUtils.openSqlSession();
        PersonMapper personMapper = sqlSession.getMapper(PersonMapper.class);
            for (int count = 0 ;count < 20;count ++) {
                PersonBean bean = new PersonBean(count + "","Fake Email" + count);
                if (count > 10){
                    sqlSession.rollback();
                }
                personMapper.insertTable(bean); 
            }
            sqlSession.commit();  //只需要利用Mybatis的SqlSession进行commit 管理即可
    }
	</pre>
+ value： 指定使用的事务管理器
+ propagation: 可选的事务传播行为设置
+ isolation:可选的事务隔离级别
+ readonly ： 读写或只读事务？默认读写 也就是说默认为true
+ rollbackfor:导致事务回滚的异常类数组
+ rollbackforClassName:导致事务回滚 的异常类名字数组
+ noRollbackFor:不会导致事务回滚的异常类数组
+ noRollbackForClassName:不会导致事务回滚的异常类名字数组	

+ 事务传播的 行为**propagation	enum: Propagation	可选的事务传播行为设置**
 + TransactionDefinition.PROPAGATION_REQUIRED：如果当前存在事务，则加入该事务；如果当前没有事务，则创建一个新的事务。这是默认值。
 + TransactionDefinition.PROPAGATION_REQUIRES_NEW：创建一个新的事务，如果当前存在事务，则把当前事务挂起。
 + TransactionDefinition.PROPAGATION_SUPPORTS：如果当前存在事务，则加入该事务；如果当前没有事务，则以非事务的方式继续运行。
 + TransactionDefinition.PROPAGATION_NOT_SUPPORTED：以非事务方式运行，如果当前存在事务，则把当前事务挂起。
 + TransactionDefinition.PROPAGATION_NEVER：以非事务方式运行，如果当前存在事务，则抛出异常。
 + TransactionDefinition.PROPAGATION_MANDATORY：如果当前存在事务，则加入该事务；如果当前没有事务，则抛出异常。
 + TransactionDefinition.PROPAGATION_NESTED：如果当前存在事务，则创建一个事务作为当前事务的嵌套事务来运行；如果当前没有事务，则该取值等价于TransactionDefinition.PROPAGATION_REQUIRED。
# Mysql的事务控制和锁定语句

+ Innodb 支持对数据级别 （row）的锁定，Myisam支持对表级别的锁定，锁定表
<pre>
lock tables table_name [as alias] {read [local]|[low_priority]write} ，table_name [as alias] {read [local]|[low_priority]write}...
</pre>

+ 事务管理：注意，默认情况下，Mysql是自动提交的，如果需要通过明确的commit和rollback来提交和回滚事务，那么就需要通过明确的事务控制命令来开始事务。
 <pre>
	start tracsaction | begin[work] commit [work][and[no] chain][[no]release] 

	rollback [work][and[no] chain][[no]release]
	set autocommit={0|1}
 </pre>

+ chain 和release 子句分别用来定义在事务结束以后，chain会立即开始一个新事务，并且和上一个事务有相同的隔离级别。release 会和客户端断开连接
+ 如果实在锁表期间开始开启一个事务，`start transaction`命令会隐含一个unlock tables的命令被执行
+ **在同一个事务中最好不要查询不同存储引擎的表，否则Rollback时需要对非事务类型的表进行特别的处理**

## 分布式事务

+ Mysql  用于分布式的应用程序涉及一个或多个**资源管理器（RM）**，和一个**事务管理器(TM)**。
 + 资源管理器（RM）：用于提供通向事务资源的途径，数据库服务器是一种资源管理器，该管理器必须可以提交或回滚由RM管理的事务。
 + 事务管理器（TM）：用于协调作为一个分布式事务一部分的事务。TM与管理每个事务的RMS进行通信。在一个分布式事务中，各个单个的事务均是分布式事务的“分支事务”。分布式事务和各分支通过一种命名方法进行标识。与Mysql连接的客户端相当于事务管理器
+ 每个分布式事务（XA 事务）都有一个特有的XID
+ `XA {START|BEGIN} XID [JOIN | RESUME]` 启动一个给定XID值的XA事务。
+ 存在的问题

