---
title: WebSocket协议和Stomp协议
---

# Stomp协议基础

+ Stomp基于消息，但是允许二进制传递。默认的消息格式是UTF-8
+ 类似Http协议，Stomp是同样是基于帧的协议
+ Stomp帧的样子..帧通常是以COMMAND字节开始，以EOL结束
+ COMMANDS和headers都是大小写敏感的，并且在UTF-8编码的headers中除了CONNECT和CONNETED帧之外，任何的回车，换行，colon found(?)都会被转义
<pre>
COMMAND
header1:value1
header2:value2
Body^@
</pre>
+ 对于body，只有send，message和ERROR帧有body，其他所有帧都没有body。并且这三个帧的也必须携带 content-length的header。

+ Stomp client通过Connect frame与server建立流或者TCP链接

<pre>
CONNECT
accept-version:1.2
host:stomp.github.org
^@
</pre>

+ 然后server返回给connected frame给client

<pre>
CONNECTED
version:1.2
^@
</pre>

或者相应ERROR Frame给client说明为什么被拒绝


+ 对于Stomp 1.1以后的版本，Connected frame必须包含accept-version header. 它的值为client所支持的Stomp版本号，如果client和server不支持共同的版本协议。server返回类似下面的ERROR frame并断开连接

```
ERROR
version:1.2,2.1
content-type:text/plain

Supported protocol versions are 1.2 2.1^@
```

## 心跳
> 心跳被用于测试底层TCP连接的可用性，确保远程服务处于活动状态,一般是作为header出现在connect frame中。

```
CONNECT
heart-beat:<cx>,<cy>
   
```

header-beat 头的两个值都是为数字，

+ 第一个数字表示发送方能做什么，为0表示不能发送心跳，其他值表保证两次心跳的最小毫秒数
+ 第二个数字代表发送方能获得什么: 0表示它不想接收心跳 ,否则它表示两次心跳期望的毫秒数 

如果接受者在规定的时间内没有收到新数据，表明连接已经断开

## Client Frame

+ **SEND**: 必须要包含Body和destination的header
+ **SUBSCRIBE** ：同样需要destination的header,表示订阅的目的地 
+ UNSUBSCRIBE：携带id header
+ BEGIN
+ COMMIT
+ ABORT
+ ACK：用于通知server，你发来的消息我已经消费。
+ NACK
+ DISCONNECT


## Message Frame

+ Message：下面是一个最基本的messageframe，另外如果frame Body携带信息的话，header应该带有content type和content length

```

MESSAGE
subscription:0
message-id:007    
destination:/queue/a  
content-type:text/plain 

hello queue a^@

```
