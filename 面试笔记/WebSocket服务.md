---
title: 搭建一个最简单的WebSocket服务
---

>  


# Server端

>Server 利用Spring Boot Websocket 进行构建。可以参考链接Spring Project的官方文档：<a href="http://docs.spring.io/spring/docs/current/spring-framework-reference/htmlsingle/#websocket">WebSocket Support</a>

利用Spring Boot搭建一个最简单的WebSocket服务，首先我们需要在Gradle中添加SpringBootWebSocket的依赖,Spring-boot-start依赖自动为我们添加了开箱即用的其他依赖库。


```
dependencies {
    compile group: 'org.springframework.boot', name: 'spring-boot-starter-websocket', version: '1.4.1.RELEASE'
}
```

```Java

@Configuration
@EnableWebSocket
public class WebSocketConfig implements WebSocketConfigurer {
    //当webSocket

    public void registerWebSocketHandlers(WebSocketHandlerRegistry registry) {
        registry.addHandler(myHandler(),"/hello").withSockJS();
    }

    @Bean
    public WebSocketHandler myHandler() {
        return new MyHandler();
    }
}

```

# Client端

client端利用 java-client这个第三方的库来实现

```
    compile group: 'org.java-websocket', name: 'Java-WebSocket', version: '1.3.0'
```

然后继承实现一个WebSocketCLient类
```Java

public class ExampleWebSocket extends WebSocketClient {
    public ExampleWebSocket(URI serverURI) {
        super(serverURI,new Draft_17());
    }

    public void onOpen(ServerHandshake handshakedata) {
        Iterator<String> stringIterator = handshakedata.iterateHttpFields();
        while (stringIterator.hasNext()){
            String name = stringIterator.next();
            System.out.println(name+" "+handshakedata.getFieldValue(name
            ));
        }
    }

    public void onMessage(String message) {
        System.out.println(message);
    }

    public void onClose(int code, String reason, boolean remote) {
        System.out.println("closed:"+reason);
    }

    public void onError(Exception ex) {
        System.out.println(ex.getMessage());
    }
}

```

最后通过 url进行连接。 注意，这url的路径 /hello就是你在Server端配置的监听路径。另外后继的/webSocket指明这是个WebSocket链接
```
exampleWebSocket = new ExampleWebSocket(new URI("ws://127.0.0.1:8081/hello/websocket"));
exampleWebSocket.connect();
```


## 源码

<a href=""git@github.com:WalkerLiuFei/WebSocketAction.git>源码</a>