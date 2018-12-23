---
layout:     post
title:      "Netty源码分析-数据拦截和处理管道ChannelPipeline的设计"
subtitle:   "Netty ChannelPipeline"
date:       2018-12-18 07:12:00
author:     "XYZ"
header-img: "img/article-bg1.jpg"
tags:
    - Netty源码分析
---
 ChannelPipeline是Java[拦截器设计模式](http://www.oracle.com/technetwork/java/interceptingfilter-142169.html)的一种高级实现方式，在pipeline中通过定义一系列ChannelHandler来处理或拦截Channel中数据的输入输出操作，使得用户可以通过ChannelHandler，定义如何对Channel收到或写出的数据进行处理，以及定义这些ChannelHandler之间的交互合作、对数据的处理顺序。
### ChannelPipeline接口设计
* 设计要点：ChannelPipeline作为数据处理管道，里面包含一系列的数据处理器ChannelHandlers，Channel中读入和写出的数据都在这个数据管道流通，并且读写会细化到多种IO事件，如OP_CONNET，OP_BIND，OP_WRITE，OP_READ等，则对每种IO事件的处理方法的定义，有两种方法：
1. 全部方法定义在ChannelPipeline，但是这里会导致：
（1）拓展性差：ChannelPipeline接口的实现必须同时对数据输入、输出的处理，无法定义一个只对数据输入的实现类；
（2）接口臃肿：ChannelPipeline需要声明处理数据输入、输出相关IO的方法，接口太大，臃肿。
故Netty采用了第二中设计方法，如下：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181222164102913.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAwMTM1NzM=,size_16,color_FFFFFF,t_70)
2. 接口细化：定义ChannelInboundInvoker和ChannelOutboundInvoker两个接口，分别定义触发：对Channel中读入数据的IO事件的处理方法的调用，写出数据的IO事件的处理方法的调用；然后ChannelPipeline继承了ChannelInboundInvoker和ChannelOutboundInvoker。Invoker中的方法，作为ChannelPipeline和ChannelHandlerContext之间的一个桥梁，即ChannelPipeline实现Invoker接口的方法，在方法内调用第一个ChannelHandlerContext开始数据的处理，开启数据处理链的调用。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181222164957497.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAwMTM1NzM=,size_16,color_FFFFFF,t_70)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181222165031871.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAwMTM1NzM=,size_16,color_FFFFFF,t_70)

### ChannelPipeline的创建时机
* 每个Channel都绑定了一个独立的ChannelPipeline，在创建Channel时也会创建一个ChannelPipeline实例。
### ChannelPipeline中的数据流动顺序
* ChannelPipeline中包含一系列的ChannelHandlers用于进行数据处理，而在Netty实现中，使用ChannelHandlerContext对ChannelHandler进行封装，然后在ChannelPipeline中是包括一个有顺序的ChannelHandlerContext处理链，在ChannelPipeline实现中自定义了head和tail两个ChannelHandlerContext分别作为ChannelHandlerContext处理链的链头和链尾，由用户按需要在head和tail中间添加自定义ChannelHandler。其中输入IO从head开始，输出IO从tail开始。
* 当服务端Channel获取到客户端Channel传递过来的请求数据，或者客户端Channel获取到服务端Channel传递过来的响应数据时，channel都会将数据放进ChannelPipeline这个数据处理管道当中。
#### 数据输入IO
1. 即由SocketChannel#read(ByteBuffer)的调用开始，读取到对方传递过来的数据，在read中调用ChannelPipeline开始数据的处理。
2. ChannelPipeline调用ChannelHandlerContext的fireChannelRead(Object)方法（或者其他fireChannelXXX，如下文中的列表）将数据传递到下一个数据输入处理器ChannelInboundHandler，具体为ChannelInboundHandler的channelRead方法（或者其他的channelXXX）。在ChannelInboundHandler的channelRead方法中进行数据处理。
3. ChannelInboundHandler的channelRead处理完数据之后：
   1. 往下传输数据：调用ChannelHandlerContext的fireChannelRead(Object)，如ctx.fireChannelRead(msg)，将数据交给ChannelInboundHandler链的下一个ChannelInboundHandler，往下传输数据；
   2. 释放数据：调用ReferenceCountUtil.release(msg)主动释放数据或者ChannelHandler实现类继承SimpleChannelInboundHandler自动释放数据，不再往下传输，具体在ChannelHandler的文章继续分析。
#### 数据输出IO
即由ChannelHandlerContext的write(Object)的调用，写数据传递给对方时。
* 经过ChannelOutboundHandler链处理再写出：ChannelPipeline将从tail开始，数据传递到数据输出处理器ChannelOutboundHandler链进行处理，最终调用SocketChannel#write(ByteBuffer)或writeAndFlush(ByteBuffer)将数据传输出去；
* 直接写出：或者直接调用SocketChannel#write(ByteBuffer)或writeAndFlush(ByteBuffer)将数据写到底层socket，而不经过ChannelOutboundHandler链。
##### 数据输入/输出，过程流程图如下：左边为数据输入，右边为数据输出
```
                                                 I/O Request
                                             via {@link Channel} or
                                         {@link ChannelHandlerContext}
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
```
例子：如下如果是输入IO，则依次调用1，2，5，输出IO，则依次调用5，4，3
```
 ChannelPipeline p = ...;
 p.addLast("1", new InboundHandlerA());
 p.addLast("2", new InboundHandlerB());
 p.addLast("3", new OutboundHandlerA());
 p.addLast("4", new OutboundHandlerB());
 p.addLast("5", new InboundOutboundHandlerX());
 ```
### ChannelPipeline中ChannelHandler之间的数据流通
 * 由于ChannelHandlerContext封装了ChannelHandler，故ChannelHandlerContext负责将数据传给下一个ChannelHandler，也就是下一个ChannelHandler的ChannelHandlerContext。
* 触发输入数据的流通的IO类型
```
ChannelHandlerContext#fireChannelRegistered()
ChannelHandlerContext#fireChannelActive()
ChannelHandlerContext#fireChannelRead(Object)
ChannelHandlerContext#fireChannelReadComplete()
ChannelHandlerContext#fireExceptionCaught(Throwable)
ChannelHandlerContext#fireUserEventTriggered(Object)
ChannelHandlerContext#fireChannelWritabilityChanged()
ChannelHandlerContext#fireChannelInactive()
ChannelHandlerContext#fireChannelUnregistered()
```
例子：
```
public class MyInboundHandler extends ChannelInboundHandlerAdapter {
     @Override
     public void channelActive({@link ChannelHandlerContext} ctx) {
         System.out.println("Connected!");
         // 传给下一个ChannelInboundHandler
         ctx.fireChannelActive();
    }
}
```

* 进行输出数据的流通的IO类型
```
ChannelHandlerContext#bind(SocketAddress, ChannelPromise)
ChannelHandlerContext#connect(SocketAddress, SocketAddress, ChannelPromise)
ChannelHandlerContext#write(Object, ChannelPromise)
ChannelHandlerContext#flush()
ChannelHandlerContext#read()
ChannelHandlerContext#disconnect(ChannelPromise)
ChannelHandlerContext#close(ChannelPromise)
ChannelHandlerContext#deregister(ChannelPromise)
```
例子：
```
public class MyOutboundHandler extends ChannelOutboundHandlerAdapter {
     @Override
     public void close(ChannelHandlerContext ctx, ChannelPromise promise) {
         System.out.println("Closing ..");
         // 传给下一个ChannelOutboundHandler
         ctx.close(promise);
     }
}
```
### ChannelHandler的耗时处理
正常情况下，Channel的一次数据IO操作，都是在其所绑定的eventLoop所在的IO线程处理的，如果某个ChannelHandler的处理时间很长，则可以为在添加这个ChannelHandler到pipeline时，指定一个线程池，让这个ChannelHandler在一个额外的线程，而不是eventLoop的线程，这样就不会阻塞eventLoop线程，不会影响到该eventLoop管理的其他Channel的数据IO操作。

```
// 可以在ChannelInitializer的实现类中，定义一个static的线程池，由所有Channel共享
static final EventExecutorGroup group = new DefaultEventExecutorGroup(16);

// 在initChannel方法为每个新建的Channel的pipeline创建ChannelHandler实例
ChannelPipeline pipeline = ch.pipeline();
pipeline.addLast("decoder", new MyProtocolDecoder());
pipeline.addLast("encoder", new MyProtocolEncoder());
pipeline.addLast(group, "handler", new MyBusinessLogicHandler());
```
### ChannelPipeline是线程安全的
* 原因：每个channel都绑定到一个eventLoop线程，即不会有其他线程会对channel进行操作，故是线程安全的，没有线程竞争问题，而channelPipeline又是channel内部的一个属性，故对channelPipeline的操作也是在这个线程中的，是线程安全的。
* ChannelHandler可以在任何时候添加或移除，如数据加密的handler对应敏感的信息可以加密，加密完成之后删除掉该handler。
