# 集成websocket

在Spring工程中集成websocket ，网上基本上都是用的spring-boot 。这种方式很简单，因为spring-boot 已经把所有的配置工作基本都做完了。但是我们项目没有使用到Spring boot ，很多配置 还是需要自己来做，集成到我们现存的项目中的时候还是踩了不少的坑。下面记录一下，防止同学再踩坑，也防止自己遗忘



在现有的工程里面集成Websocket，因为工程是基于Spring MVC的，最好的办法是使用Spring 4.0.0版本以后提供的Webscocket API。 

在集成过程中主要参考了 [spring 官方手册](https://docs.spring.io/spring/docs/4.0.0.RELEASE/spring-framework-reference/html/websocket.html#websocket-server-deployment), 过程中遇到非常多的问题。在这里做一下记录

## Spring websocket api做的简单封装

由于Spring API的Websocket的握手要通过拦截器来处理，直接使用起来回需要到处找代码。另外websocket的handle类还需要在配置文件（类）中进行配置。



为了解决以上问题，我做了一些封装。示例代码 参考 2.7版本下面 gwcrm 的`com.opengroup.hongshi.gwcrm.web.crm.web.websocket` 下面的代码

```
@SocketController("/workOrderIndex.do") 
public class WorkbenchIndexHandler extends BaseHandler {
    @Autowired
    private KvClient kvClient;
    @Override
    public void handleMessage(final WebSocketSession session, WebSocketMessage<?> message)
            throws Exception {
      	//处理client端发送的信息
    }



    @Override
    boolean beforeHandshake(HttpServletRequest request,
                            HttpServletResponse response,
                            Map<String, Object> attributes) {
     	//TODO : 握手时会调用，进行登录验证等等操作
    }
}

```



## 握手过程遇到的坑

1. **在工程中添加下面两个依赖**

   ```xml
   			<dependency>
   				<groupId>org.springframework</groupId>
   				<artifactId>spring-websocket</artifactId>
   				<version>4.0.0.RELEASE</version>		
   			</dependency>
   			<dependency>
   				<groupId>org.springframework</groupId>
   				<artifactId>spring-messaging</artifactId>
   				<version>4.0.0.RELEASE</version>
   			</dependency>
   ```

   ​

2. **保证Spring 的版本大于4.0.0-REALASE** ：spring的websocket 是从Spring 4.0.0-RELEASE 版本以后开始支持的。所以首先要保证版本大于等于这个版本

3. 保证Tomcat的版本大于 7.0.52。**注意我们的 tomcat-tomcat7-plugin插件的运行的版本 是tomcat 7.0.42 这个tomcat是缺少websocket 容器的，需要做下下面的配置 来更改插件运行的tomcat的版本号 [看这个文档改版本号](http://tomcat.apache.org/maven-plugin-2.2/tomcat7-maven-plugin/adjust-embedded-tomcat-version.html)**  。。另外，在上面的基础上，你还需要加上下面的插件依赖

   ```xml
   					<dependency>
   						<groupId>org.apache.tomcat.embed</groupId>
   						<artifactId>tomcat-embed-websocket</artifactId>
   						<version>${tomcat.version}</version>
   					</dependency>
   ```

4. 保证web.xml的servlet版本号大于 3.0..并且**需要在 web.xml 中添加下面的配置来支持 JSP-356**

   ```
   	<absolute-ordering>
   		<name>spring_web</name>
   	</absolute-ordering>
   ```

   ​

5. websock 握手是通过dispatcher servlet进行握手的，。握手的路径一定要满足dispatcher servlet能映射这个握手请求 例如 gwcrm映射 `*.do`的映射，所以在gwcrm下面的 握手请求相对路径都需要满足 `/gwcrm/*.do`

6. 在pom中添加下面的两个依赖

   ```
   		<dependency>
   			<groupId>javax.websocket</groupId>
   			<artifactId>javax.websocket-api</artifactId>
   			<scope>provided</scope>
   			<version>1.1</version>
   		</dependency>
   		<!-- https://mvnrepository.com/artifact/org.apache.tomcat/tomcat7-websocket -->
   		<dependency>
   			<groupId>org.apache.tomcat</groupId>
   			<artifactId>tomcat7-websocket</artifactId>
   			<version>7.0.61</version>
   			<scope>provided</scope>
   		</dependency>
   ```

   ​

7. https://stackoverflow.com/questions/37853810/difference-between-topic-queue-for-simplemessagebroker-in-spring-websocket

