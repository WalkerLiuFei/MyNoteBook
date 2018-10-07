---
title: Netty 基础学习（二）
---


+ EventLoop 是来处理channel 中的IO操作，Netty中所有的IO操作默认是异步的，你需要利用ChannleFuture 注册相应ChannleFutureListeners来对异步IO的状态进行监听。
+ Netty其实是利用多线程来处理IO事件的，通过源码你会发现，EventLoop其实继承的是ExcutorService。对于NioEventLoopGroup来说，其默认的线程数是2 * CPU核心数。
+ **Netty 中为什么不需要关心同步问题？**
+ ServerBootGroup 需要两个EventLoopGroup,用来绑定本地端口。BootStrap需要一个EventLoopGroup .....对于ServerBootGroup来讲，他需要一个EventGroup来处理连接，然后传递给后面子EventGroup。
+ Channel 是线程安全的，怎么保证线程安全的？
# ChannleHandler

+ 每一个ChannelHandler 在从一个ChannlePipeline中移除之前不能添加到另一个ChannlePipline中去,并且一般来讲，每一个ChannelHandler是不允许阻塞IO 的，如果有这方面的需求，你可能需要在添加ChannlePipline的同时添加一个EventLoopGroup
+ ChannleHandler 中以`fireXXX`开头的方法都是唤醒Handler所在Pipleling下面的的对应方法，对于Outbound 和inbound的顺序注意区分
+ 每次添加channelHanndler 到pipline中的时候会有一个ChannleHandlerContext被创建并被分配
+ 如果你想将一个ChannelHandler的实例添加到不同pipline中，可以通过添加注解`@Shareable`的方法实现， 不过要注意线程安全的问题
+ 在channelRead 中你应该及时释放资源**ReferenceCountUtil.release(msg)**不然会造成内存泄露，另外你也可以利用继承SimpleInboundHanlder来替你完成这些工作 


# Netty中的ByteBuf

+ ByteBuf实现了ReferenceCount接口。顾名思义其是个计数类，参考JVM垃圾回收机制
+ netty ByteBuf 相对于Java Nio ByteBuffer的优势：
	+ 自定义缓冲类型
	+ 通过一个内置的复合类型实现零拷贝
	+ 扩展性好，比如StringBuffer
	+ 不需要调用flip()来切换读写状态
	+ 读取和写入索引分开
	+ 方法链
	+ 引用计数功能
	+ Pool（池）的引入
+ ByteBuf的类型
 + HeapBuffer : 顾名思义，底层存储使用数组的数据结构将数据存储在JVM的堆空间上面----> ByteBuf.array() 来获取byte[]数据
 + DirectBuffer：同Nio 中的Direct Buffer，Netty通过内存池的概念来解决其分配内存和释放内存的复杂的问题，通过间接方式来对directBuffer来读写

	
	 ```Java
    ByteBuf directBuf = Unpooled.directBuffer(16);
        directBuf.writeBytes("I'm datas".getBytes());
        if (!directBuf.hasArray()){ //如果Bytebuf 的存储状态不是 堆
            int len = directBuf.readableBytes(); // 首先得到其长度
            byte[] arr = new byte[len];
            directBuf.getBytes(0,arr);
            System.out.println(new String(arr));
        }
	 ```

 + Composite Buffer： 
	 + Composite Buffer 的hasArray总是返回false,虽然它有可能会在堆内存上有存储 
 + 对于已读字节，以readerIndex 为准，使用 discardReadBytes，可以清楚【0...readerIndex】区间的字节
 + 利用ByteBuf的clear 函数来 复位readerIndex和writerIndex为0,clear不会清空ByteBuf的内容，它影响的只是Index。。。
 + Netty Buffer包中的工具类
	 + BufferAllactor 用来分配ButyBuf实例
	 + Unpooled 是一个用来创建缓冲区的工具
	 + ByteBufUtil,其中的 HexDump 用来返回ByteBuf中 16进制信息。可以用来调试


# Netty中的编解码

+ 编码器负责入站，而解码器负责出站。**他们是怎么实现区分入站和出站的区别判断的？**
+ MessageToByteEncoder和ByteToMessageDecoder是使用最多的编解码类。尝试读这两个类的源码，另外还有MessageToMessageEncode和MessageToMessageDecoder两个类

# Netty 中的Http

+ 在netty中处理Http请求，需要两个编解码类 分别为 HttpReqestDecoder，HttpRequestDecoder,HttpResponseDecoder,HttpResponseEncoder...其中一般在HttpRequestDecoder和HttpResponseEncoder 用的较多
+ Netty Server 处理Request请求时比较底层，没有做过多的封装，需要自行判断握手，信息传递，长连接判断，和trailer header的判断
最后在Netty中映射的类分别为 `HttpRequst -----> HttpCotent...HttpContent -----> LastHttpContent`

# 测试Neety 代码