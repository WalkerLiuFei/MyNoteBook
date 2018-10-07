---
title: netty 基础学习（一）
---

# 基础使用

+ 建立一个Netty Server的步奏

 + 需要一个EventLoopGroup 作为ParentEventLoop 来接受外部的request，还有一个EvnentLoop作为Childs 来处理进入的链接
 + 利用一个BootStrap类group 方法包裹住这两个EventLoop来启动这个服务
 + 然后利用BootStrap的channel方法指定channel
 + 然后指定handler。

```Java
        EventLoopGroup bossGroup = new NioEventLoopGroup();  //用来接受进来的连接
        EventLoopGroup workerGroup = new NioEventLoopGroup(); //用来处理已经被接受的连接
        try {
            ServerBootstrap b = new ServerBootstrap();
            b.group(bossGroup,workerGroup)     //group 之后 boss接受到的连接都会映射到 work上面
                    .channel(NioServerSocketChannel.class)
                    .childHandler(new ChannelInitializer<SocketChannel>() {
                        protected void initChannel(SocketChannel ch) throws Exception {
                            ch.pipeline().addLast(new DiscardServerHandler())
                                         .addLast(new TimeServerHandler()) //在这里添加一个新的channel handler.
                                         .addLast(new TimeEncoder())
                                         .addLast(new TimeServerHandler2());  //应该在将Encoder 放在 server handler 之前！
                        }
                    })
                    .option(ChannelOption.SO_BACKLOG,128)   //为NioServerSocketChannel 设置的选项
                    .childOption(ChannelOption.SO_KEEPALIVE,true); //为parent Server group 设置的选项

            //绑定端口，开始接收进来的连接
            ChannelFuture f = b.bind(port).sync();
            //等待服务器 Socket 关闭
            f.channel().closeFuture().sync();
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            //...优雅的关闭。。。这名字，我也是醉了
            workerGroup.shutdownGracefully();
            bossGroup.shutdownGracefully();
        }
```

+ 每个Netty 的Server都需要一个 HandlerAdpater，来处理接收到的消息  ChandleHandlerAdapter
+
	



# 概念

## Netty中的缓冲区
 
Netty使用<a href = "http://netty.io/5.0/api/io/netty/buffer/package-summary.html#package_description">自建的buffer Api,而不是NIO 的byte buffer</a> 来标示一个连续的字节序列。

 + 如果需要吗，允许使用自定义的缓冲实现。
 + 复合缓冲类型中内置的透明的零拷贝实现
 + 开箱即用的动态缓冲类型，具有像StringBuffer一样的动态缓冲能力
 + 不再需要调用的flip() 方法。
 + 正常情况下具有比ByteBuffer更快的相应速度

### 可扩展性

ByteBuf 具有丰富的操作集，可以实现快速的协议优化。你可以扩展包装现有的缓冲类型来提供方便的访问。自定义缓冲仍然实现自ByteBuf接口，而不是引入一个不兼容的类型。

### 透明的零拷贝

```Java
// 复合类型与组件类型是兼容的。
ByteBuf message = Unpooled.wrappedBuffer(header, body);
// 因此，你甚至可以通过混合复合类型与普通缓冲区来创建一个复合类型。
ByteBuf messageWithFooter = Unpooled.wrappedBuffer(message, footer);
// 由于复合类型仍是 ByteBuf，访问其内容很容易，
//并且访问方法的行为就像是访问一个单独的缓冲区，
//即使你想访问的区域是跨多个组件。
//这里的无符号整数读取位于 body 和 footer
messageWithFooter.getUnsignedInt(
messageWithFooter.readableBytes() - footer.readableBytes() - 1);
```

### 自动增量扩展

### 更好的性能

相对于ByteBuffer ByteBuf的实现 是一个非常薄的字节数组包装器（例如一个字节），它没有复杂的边界检查和索引检查补偿，因此对于JVM的优化缓冲区的访问更加简单。ByteBuf更多是在对缓冲区的切分和重组，它的性能和ByteBuffer几乎相当

## Netty中的IO实现

### 更好的API兼容

不想原生Java 的**IO/NIO/AIO** 的的API，Netty的所有类型的IO 实现都通过同一个接口<a href = "http://netty.io/4.0/api/io/netty/channel/package-summary.html#package_description">Channel</a>,如果你想切换不的IO实现，只需要很简单的步奏，例如选择一个<a href = "http://netty.io/4.0/api/io/netty/bootstrap/ChannelFactory.html">ChannelFactory 的实现</a>

+ 基于 NIO 的 TCP/IP 传输 (见 io.netty.channel.nio),
+ 基于 OIO 的 TCP/IP 传输 (见 io.netty.channel.oio),
+ 基于 OIO 的 UDP/IP 传输, 和本地传输 (见 io.netty.channel.local)

并且你还可以继承Channle 添加自定义的实现!

## Netty中的事件模型
	
Netty 中事件模型的核心组件是<a href ="http://netty.io/4.0/api/io/netty/channel/ChannelPipeline.html">ChannelPipeline</a>

<pre>
                                                 I/O Request
                                            via <a href="http://netty.io/4.0/api/io/netty/channel/Channel.html" title="interface in io.netty.channel"><code>Channel</code></a> or
                                        <a href="../../../io/netty/channel/ChannelHandlerContext.html" title="interface in io.netty.channel"><code>ChannelHandlerContext</code></a>
                                                      |
  +---------------------------------------------------+---------------+
  |                           ChannelPipeline         |               |
  |                                                  \|/              |
  |    +---------------------+            +-----------+----------+    |
  |    | Inbound Handler  N  |            | Outbound Handler  1  |    |
  |    +----------+----------+            +-----------+----------+    |
  |              /|\                                  |               |
  |               |                                  \|/              |
  |    +----------+----------+            +-----------+----------+    |
  |    | Inbound Handler N-1 |            | Outbound Handler  2  |    |
  |    +----------+----------+            +-----------+----------+    |
  |              /|\                                  .               |
  |               .                                   .               |
  | ChannelHandlerContext.fireIN_EVT() ChannelHandlerContext.OUT_EVT()|
  |        [ method call]                       [method call]         |
  |               .                                   .               |
  |               .                                  \|/              |
  |    +----------+----------+            +-----------+----------+    |
  |    | Inbound Handler  2  |            | Outbound Handler M-1 |    |
  |    +----------+----------+            +-----------+----------+    |
  |              /|\                                  |               |
  |               |                                  \|/              |
  |    +----------+----------+            +-----------+----------+    |
  |    | Inbound Handler  1  |            | Outbound Handler  M  |    |
  |    +----------+----------+            +-----------+----------+    |
  |              /|\                                  |               |
  +---------------+-----------------------------------+---------------+
                  |                                  \|/
  +---------------+-----------------------------------+---------------+
  |               |                                   |               |
  |       [ Socket.read() ]                    [ Socket.write() ]     |
  |                                                                   |
  |  Netty Internal I/O Threads (Transport Implementation)            |
  +-------------------------------------------------------------------+
 </pre>


在netty 5.0 中`ChannelInboundHandler` 和`ChannelOutboundHandler` 合并为`ChannelHandler`，统一的Adapter也是一样合并了 

## SSL/TLS 支持

+ 通过类<a href = "http://netty.io/4.0/api/io/netty/handler/ssl/SslHandler.html">SslHandler </a> 
+ check the example <a href = "http://netty.io/4.1/xref/io/netty/example/securechat/package-summary.html"> Secure Chat </a>
+ netty <a href="http://stackoverflow.com/questions/26803353/ssl-support-in-netty-io-through-custom-certificate">使用自签名作为SSL认证</a>
	+ 第一步： 配置好OpenSSL环境
	+ 第二步： 利用命令`openssl req -x509 -keyout abc.key -out xyz.csr -newkey                  rsa:2048 `	 产生证书文件和私匙文件  
    + 第三步： 利用命令`openssl pkcs8 -topk8 -inform PEM -outform PEM -in abc.key -out abc.pem` 将key转换为 pkcs8 格式的格式
    + 第四步： 创建一个SSLContext
    
	<pre> SslContext sslCtx = SslContext.newServerContext(new File(
            "./key/xyz.csr"), new File("./key/abc.key"),"password");</pre>

## WebSocket 实现

## Summary

这个架构由三部分组成——缓冲（ buffer） ，通道（ channel） ，事件模型（ event
model） ——所有的高级特性都构建在这三个核心组件之上。