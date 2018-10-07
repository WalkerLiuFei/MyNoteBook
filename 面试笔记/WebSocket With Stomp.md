---
title: 搭建一个配合Stomp消息的WebSocket服务
---


# Server端
> <a href="http://blog.109yxlm.cc/2016/11/10/WebSocket%E6%9C%8D%E5%8A%A1/">最简单的WebSocket服务</a>

## 配置

```Java
@Configuration
@EnableWebSocketMessageBroker//这个注解不仅配置了WebSocket，还配置了基于代理的STOMP消息。
public class WebSocketMessageConfig extends AbstractWebSocketMessageBrokerConfigurer {
    /**
     * Spring的Web消息功能基于消息代理（message broker）
     */
    @Override
    public void configureMessageBroker(MessageBrokerRegistry registry) {
        //配置消息代理,客户端在监听相应信息时加加上这个前缀
        registry.enableSimpleBroker("/client");
        //所有带有"/app*"前缀的请求都会被映射到带有@MessageMapping注解的方法中去
        registry.setApplicationDestinationPrefixes("/app");
    }

    public void registerStompEndpoints(StompEndpointRegistry registry) {
        /**
         * 这些路径即为Stomp 监听WebSocket的路径。
         * 客户端在配置websocket url时应为 : "ws://localhost:8081/hello/websocket"
         */
        registry.addEndpoint("/hello","/client").withSockJS();
    }
}
```

+ @EnableWebSocketMessageBroker 这个注解不仅配置了WebSocket，还配置了基于代理的STOMP消息
+ 重写的registerEndPoint里面添加的Endpoint,将“/hello”注册为**STOMP端点**。这个路径与之前发送和接收消息的**目的地路径**有所不同。这是一个端点，客户端在订阅或发布消息到目的地路径前，要通过websocket连接该端点。
+ 通过重载configureMessageBroker()方法配置了一个简单的消息代理。所以消息代理将会处理前缀为“/client”的消息,除此之外，发往应用程序的消息将会带有“/app”前缀
+ 所有目的地以“/app”打头的消息都将会路由到带有@MessageMapping注解的方法中，而不会发布到代理队列或主题中。

## 处理来自客户端的STOMP消息

```Java
@Controller
public class MessageController {
    @MessageMapping("/hello")
    @SendTo("/client/welcome")
    public WelcomeMessage greeting(HelloMessage message) throws Exception {
        Thread.sleep(1000); // simulated delay
        return new WelcomeMessage("welcome " + message.getName() + "!");
    }

    @MessageMapping("/schedule")
    @SendTo("/client/schedule")  
    public ScheduleTaskMessage schedule(ScheduleTaskMessage message) throws Exception {
        System.out.println("schedule task message " + message.getContent());
        Thread.sleep(1000); // simulated delay
        message.setContent("I already received the message："+message.getContent());
        return message;
    }
}
```

在本例中，这个目的地也就是“/app/hello”和“/app/schedul“ 的消息，“/app”前缀是隐含的，因为我们将其配置为应用的目的地前缀，并发送消息到指定的路径。注意，发送的路径要以在 SimpleBroker开头。不带Stomp接受不到。在这里是以“client开头的”


# Client

> Client端是自实现的Stomp协议，有些负责这里只介绍最基本的使用，具体请查看<a href="https://github.com/WalkerLiuFei/WebSocketAction">源码</a>

```Java
StompClient mStompClient = Stomp.over(WebSocket.class, "ws://localhost:8081/hello/websocket");
mStompClient.connect(); //连接到指定路径的WebSocket链接

//订阅Stomp消息
mStompClient.topic("/client/welcome").subscribe(topicMessage -> {
    System.out.println("the schedule message :" + topicMessage.getPayload()); 
});
//向客户端发送一个Stomp消息
mStompClient.send("/app/hello", objectMapper.writeValueAsString(new HelloMessage("walker"))).subscribe();

```

         



