# Mybatis拦截器与自定义分页插件的实现

+ 首先Mybatis 默认是内存分页，就是将所有数据从磁盘读入到内存中后，再去分页。这样会大幅度降低应用性能，至于为什么，自己看代码。
+ Mybatis 的物理分页可以Mybatis的拦截器来做
  + 拦截StatementHandler的prepare 方法，然后在拦截器中把Sql语句更换成对应的分页查找查询语句，然后再调用StatementHandler对象的prepare方法，即调用`invocation.proceed()`方法
    +
+ Mybatis拦截器只能拦截四种类型的接口：Executor、StatementHandler、ParameterHandler和ResultSetHandler。

+ 一个简单的拦截器例子
 + @Intercept 注解中的@Signature中标示的属性，标示当前拦截器要拦截的**那个类的那个方法，拦截方法的传入的参数**

```java
	//拦截StatementHandler.class的prepare 方法！，这个方法的参数是一个Connecttion对象和Interger对象
	@Intercepts( {
        @Signature(method = "prepare", type = StatementHandler.class, args = { Connection.class ,Integer.class}) })
	public class TestInterceptor implements Interceptor {
		Object intercept(Invocation invocation) throws Throwable{....};

  		Object plugin(Object target){....};

  		void setProperties(Properties properties){....};
	}
---------------------注意在Mybatis 的XML中注册拦截器！--------------------------------
		
```

**现在已经掌握一个拦截器的写法，现在的问题就在于，从什么地方着手去拦截？**

+ 首先要明白，Mybatis 是对JDBC的一个高层次的封装。而JDBC 在完成数据操作的时候必须要有一个Statement对象。而Statement对应的SQL语句是在是在Statement之前产生的。所以我们的思路就是在生成Statement之前对sql进行下手。更改sql语句成我们需要的！
+ 对于Mybatis，其Statement生成是在`RouteStatementHandler`中。所以我们要做的就是拦截这个handler的`prepare`方法！然后修改Sql语句！！！！

```java
	//拦截到Sql 并输出到控制台
   @Override
    public Object intercept(Invocation invocation) throws Throwable {
         // 其实就是代理模式！
        RoutingStatementHandler handler = (RoutingStatementHandler) invocation.getTarget();
        StatementHandler delegate = (StatementHandler)ReflectUtil.getFieldValue(handler, "delegate");
        System.out.println(delegate.getBoundSql().getSql());
        //Sy).getClass().getName());stem.out.println(invocation.getTarget(

        return invocation.proceed();
    }
```

**我们知道利用Mybatis查询一个集合时传入Rowbounds对象即可指定其Offset和Limit，只不过其没有利用原生sql去查询罢了，我们现在做的，就是通过拦截器 拿到这个参数，然后织入到SQL语句中，这样我们就可以完成一个物理分页！**


```Java

	package person.walker.interceptor;
	
	import org.apache.ibatis.executor.Executor;
	import org.apache.ibatis.executor.statement.RoutingStatementHandler;
	import org.apache.ibatis.executor.statement.StatementHandler;
	import org.apache.ibatis.mapping.BoundSql;
	import org.apache.ibatis.mapping.MappedStatement;
	import org.apache.ibatis.plugin.*;
	import org.apache.ibatis.session.ResultHandler;
	import org.apache.ibatis.session.RowBounds;
	import sun.reflect.Reflection;
	
	import java.lang.reflect.Field;
	import java.sql.Connection;
	import java.util.Properties;
	
	/**
	 * Created by Administrator on 2017/3/1 0001.
	 */
	
	@Intercepts({
	        @Signature(method = "prepare", type = StatementHandler.class, args = {Connection.class, Integer.class})})
	public class TestInterceptor implements Interceptor {
	    // select语句正则表达式匹配：
	    private final static String REGEX = "^\\s*[Ss][Ee][Ll][Ee][Cc][Tt].*$";
	
	    @Override
	    public Object intercept(Invocation invocation) throws Throwable {
	        RoutingStatementHandler handler = (RoutingStatementHandler) invocation.getTarget();
	        // BoundSql类中有一个sql属性，即为待执行的sql语句
	        BoundSql boundSql = handler.getBoundSql();
	        String sql = boundSql.getSql();
	        if (sql.matches(REGEX)) {
	            // delegate是RoutingStatementHandler通过mapper映射文件中设置的statementType来指定具体的StatementHandler
	            Object delegate = getFieldValue(handler, "delegate");
	            // rowBounds,即为Mybais 原生的Sql 分页参数,由于Rowbounds 在BaseStateHandler中所以我们需要去找父类
	            RowBounds rowBounds = (RowBounds) getFieldValue(delegate, "rowBounds");
	            // 如果rowBound不为空，且rowBounds的起始位置不为0，则代表我们需要进行分页处理
	            if (rowBounds != null) {
	                // assemSql(...)完成对sql语句的装配及rowBounds的重置操作
	                setFieldValue(boundSql, "sql", assemSql(sql, rowBounds));
	            }
	        }
	        return invocation.proceed();
	    }
	
	    @Override
	    public Object plugin(Object target) {
	        return Plugin.wrap(target, this);
	    }
	
	    @Override
	    public void setProperties(Properties properties) {
	        String prop1 = properties.getProperty("prop1");
	        String prop2 = properties.getProperty("prop2");
	        System.out.println(prop1 + "------" + prop2);
	    }
	
	    private Object getFieldValue(Object object, String fieldName) {
	        Field field = null;
	        for (Class<?> clazz=object.getClass(); clazz != Object.class; clazz=clazz.getSuperclass()) {
	            try {
	                field = clazz.getDeclaredField(fieldName);
	                if (field != null){
	                    field.setAccessible(true);
	                    break;
	                }
	
	            } catch (NoSuchFieldException e) {
	                //这里不用做处理，子类没有该字段可能对应的父类有，都没有就返回null。
	            }
	        }
	        try {
	            return field.get(object);
	        } catch (IllegalAccessException e) {
	            e.printStackTrace();
	        }
	        return null;
	    }
	
	    private void setFieldValue(Object object, String fieldName, Object value) {
	        Field field = null;
	        for (Class<?> clazz=object.getClass(); clazz != Object.class; clazz=clazz.getSuperclass()) {
	            try {
	                field = clazz.getDeclaredField(fieldName);
	                if (field != null){
	                    field.setAccessible(true);
	                    try {
	                        field.set(object,value);
	                    } catch (IllegalAccessException e) {
	                        e.printStackTrace();
	                    }
	                    break;
	                }
	
	            } catch (NoSuchFieldException e) {
	                //这里不用做处理，子类没有该字段可能对应的父类有，都没有就返回null。
	            }
	        }
	    }
	
	    public String assemSql(String oldSql, RowBounds rowBounds) throws Exception {
	        String sql = oldSql + " limit " + rowBounds.getOffset() + "," + rowBounds.getLimit();
	        // 这两步是必须的，因为在前面置换好sql语句以后，实际的结果集就是我们想要的所以offset和limit必须重置为初始值
	        setFieldValue(rowBounds, "offset", RowBounds.NO_ROW_OFFSET);
	        setFieldValue(rowBounds, "limit", RowBounds.NO_ROW_LIMIT);
	        return sql;
	    }
	
	}
	
```