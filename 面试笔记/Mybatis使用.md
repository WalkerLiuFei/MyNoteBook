---
title： Mybatis 基本使用
---


+ SQL 映射文件有很少的几个顶级元素（按照它们应该被定义的顺序）：
 + cache – 给定命名空间的缓存配置。
 + cache-ref – 其他命名空间缓存配置的引用。
 + resultMap – 是最复杂也是最强大的元素，用来描述如何从数据库结果集中来加载对象。
 + parameterMap – 已废弃！老式风格的参数映射。内联参数是首选,这个元素可能在将来被移除，这里不会记录。
 + sql – 可被其他语句引用的可重用语句块。
 + insert – 映射插入语句
 + update – 映射更新语句
 + delete – 映射删除语句
 + select – 映射查询语句


# Mapper 

+ Mapper中子语句中的id 标示是不需要 加 “.” 作为前缀的，其在SqlSession 中通过Mapper的NameSpace的前缀和id结合获取时候是默认加上“.”的
+ 作为Domain的bean， 一定要有geter/setter
+ **selectMap 和 selectList 的区别其实就是多一个mapKey，这个mapKey其实是查询结果集的一个 cloumn name！！！！利用这个作为 结果集中每一个结果额映射**

<pre>   
		//这个 id,其实就是查询时一个列 
		Map<String,Object> map =  sqlSession.selectMap(getClass().getName()+".selectUserContent",userID,"id");
</pre>



## 一个Mapper的配置， 


+ **Mapper的 NameSpace 是不能少的，这个表示对于这个Mapper的映射，通过这个映射可以顺利找到Mapper**

```XML 

		<?xml version="1.0" encoding="UTF-8" ?>
		<!DOCTYPE mapper
		        PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
		        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
		<mapper namespace="person.walker.dao.UserDAO">
		    <insert id="insertUser" parameterType="user">
		      INSERT INTO USERS VALUES (#{id},#{account})
		    </insert>
		</mapper>
```

### ReslutMap

ReslutMap中的选项,Type指定一个 **类**，
 
 + constructor - 类在实例化时,用到的构造函数
  + idArg - ID 参数;标记结果作为 ID 可以帮助提高整体效能
  + arg - 注入到构造方法的一个普通结果
  + 这其实就是 指定这个通过sql结果查询构造函数时的参考，包括类型和column 
 + id – 一个 ID 结果;标记结果作为 ID 可以帮助提高整体效能
 + result – 注入到字段或 JavaBean 属性的普通结果
 + association – 一个复杂的类型关联
  + 嵌入结果映射，其实就是ResultMap 属性 Type类中的一个 属性，只不过这个属性是一个java对象
 + collection – 复杂类型的集嵌入结果映射 
  + 嵌入结果映射到一个集合中，其实这个集合包含有Java对象
 +　discriminator – 使用结果值来决定使用哪个结果映射
  + case – 基于某些值的结果映射 嵌入结果映射
	
# Mybatis 的一个典型配置配置

+ **下面的这些配置都是不可以变的！！**

```XML

	<?xml version="1.0" encoding="UTF-8" ?>
	<!DOCTYPE configuration
	        PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
	        "http://mybatis.org/dtd/mybatis-3-config.dtd">
	<configuration>
	    <settings>
	        <setting name="cacheEnabled" value="true"/>
	        <!--延迟加载-->
	        <setting name="lazyLoadingEnabled" value="true"/>
	        <setting name="multipleResultSetsEnabled" value="true"/>
	        <setting name="useColumnLabel" value="true"/>
	        <setting name="useGeneratedKeys" value="false"/>
	        <setting name="autoMappingBehavior" value="PARTIAL"/>
	        <setting name="autoMappingUnknownColumnBehavior" value="WARNING"/>
	        <setting name="defaultExecutorType" value="SIMPLE"/>
	        <setting name="defaultStatementTimeout" value="25"/>
	        <setting name="defaultFetchSize" value="100"/>
	        <setting name="safeRowBoundsEnabled" value="false"/>
	        <setting name="mapUnderscoreToCamelCase" value="false"/>
	        <setting name="localCacheScope" value="SESSION"/>
	        <setting name="jdbcTypeForNull" value="OTHER"/>
	        <setting name="lazyLoadTriggerMethods" value="equals,clone,hashCode,toString"/>
	    </settings>
	    <typeAliases>
	        <!--用来减少Mapper 中 parameter 和 result 的限定名称，指定这个包以后，会使用 Bean 的首字母小写的非限定类名来作为它的别名 -->
	        <package name="person.walker.Domain"/>
	    </typeAliases>
	    <!--environments 内部可以 配置多个环境， 我们在这里默认使用 development 的 environment。
	           默认的环境ID development-->
	    <environments default="development">
	        <environment id="development">
	            <!--利用JDBC的事务管理-->
	            <transactionManager type="JDBC"/>
	            <!--配置 数据源,使用连接池的属性-->
	            <dataSource type="POOLED">
	                    <!--数据库JDBC的URL地址-->
	                    <property name="url" value="jdbc:mysql://localhost:3306/redisAction?useUnicode=true&amp;characterEncoding=utf8"/>
	                    <!--指定驱动为JDBC-->
	                    <property name="driver" value="com.mysql.jdbc.Driver"/>
	                    <property name="username" value="root"/>
	                    <property name="password" value="3252860"/>
	            </dataSource>
	        </environment>
	    </environments>
	    <mappers>
			<!--这个 ./ 就表示resource 目录的根目录-->
	        <mapper resource="./mapper/user.xml"/>
	    </mappers>
	</configuration>

```
