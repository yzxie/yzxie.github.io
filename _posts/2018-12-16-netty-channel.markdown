---
layout:     post
title:      "Netty源码分析-网络通信NIO和Channel"
subtitle:   "Netty Channel"
date:       2018-12-16 07:12:00
author:     "XYZ"
header-img: "img/article-bg1.jpg"
tags:
    - Netty源码分析
---
### Java socket之BIO和NIO
在网络编程当中，在应用层主要通过Socket Api来完成客户端和服务端之间的网络通信。
* BIO: 在Java中，服务端使用ServerSocket监听客户端连接请求，客户端使用Socket连接服务端，ServerSocket和Socket都是BIO，即阻塞IO。阻塞IO存在的问题是任何时候，在一个线程当中只能存在一个socket进行连接，不能多个socket进行并行连接，如果需要支持多个socket并行连接，则每个socket需要在一个独立的线程中，通常会使用一个线程池来实现。这种方式在高并发下，对系统资源开销很大，不能适合高并发访问。
* NIO: 为了解决BIO存在的问题，即在一个线程当中能够支持多个socket的连接，在Java NIO包中，定义了SocketChannel和Selector，用于实现NIO，即非阻塞IO。
1. Selector管理多个SocketChannel，监听这些SocketChannel的IO事件，然后分发给对应的SocketChannel进行处理，其底层主要依赖于操作系统提供的IO多路复用技术，如Linux的select，poll，epoll系统调用，Windows的kqueue，实现在一个线程里面处理多个socket的连接。
2. 在数据传输方面，SocketChannel中封装了socket，在读取数据时，将channel中的数据读取到ByteBufer缓冲区，在写数据时，将数据写到ByteBuffer缓冲区，然后通过channel发送出去，可以通过方法configureBlocking，设置是否启用非阻塞模式，如果设置了，则SocketChannel可以直接返回，而不需要阻塞，通常SocketChannel设置为非阻塞模式，而ServerSocketChannel为服务端监听客户端连接请求，可以设置为阻塞模式，即在accept中阻塞等待客户端的连接请求。
### Netty体系结构
如果使用Java NIO原生Api进行网络编程，则需要考虑如线程安全，网络异常处理等各种问题，对于编写企业级、高稳定性网络应用难度较高。所以Netty对Java NIO的SocketChannel进行了封装和优化。
* 通过EventLoop事件循环机制保证SocketChannel在多线程环境的线程安全。
* 通过ChannelPipeline事件处理器管道，ChannelHandler事件处理器的设计，简化了网络应用的开发，开发者可以只需要使用ChannelHandler定义业务逻辑的实现，同时指定各个ChannelHandler在ChannelPipeline中处理顺序即可，不需要考虑底层网络通信以及线程安全方面的问题。
* 同时也对Java NIO的Api进行了优化，如ByteBuffer缓冲区的优化，这个在之后的文章继续分享。
* Netty Channel接口则将这些设计融合起来，也作为整个设计对外的一个入口。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181220232456217.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAwMTM1NzM=,size_16,color_FFFFFF,t_70)
### Netty Channel的核心设计
#### Channel接口核心属性和方法
* EventLoop：channel所绑定的用于处理channel IO操作的线程
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181220235201837.png)
* ChannelPipeline：channel读取或写出的message，需要流经的处理管道或者处理链，里面包括多个ChannelHandler，分别对message进行处理。
![在这里插入图片描述](https://img-blog.csdnimg.cn/2018122023553573.png)
* 缓冲区生成器ByteBufAllocator：用于生成进行数据缓存的ByteBuf，可以生成堆内或堆外内存，以及是否池化pooled。
![在这里插入图片描述](https://img-blog.csdnimg.cn/2018122023563639.png)
* socket套接字地址对：本地地址和远程地址
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181221100757926.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAwMTM1NzM=,size_16,color_FFFFFF,t_70)
#### 异步IO机制
在Channel中所有的IO操作，包括read, write, bind, connect，都是异步操作，即各种IO操作执行完之后都是直接返回，然后通过ChannelFuture或ChannelPromise异步获取IO操作的执行结果。一般通过addListener方法添加ChannelFutureListener监听器接口的实现类来处理IO操作的结果。
* 实现ChannelFutureListener的operationComplete方法来处理操作结果。如图：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181221000139950.png)
* ChannelFutureListener接口自身提供的处理器，以常量形式提供，如下：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181221000503973.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAwMTM1NzM=,size_16,color_FFFFFF,t_70)
关闭Channel添加Listener：ChannelFutureListener.CLOSE
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181221105932707.png)
#### NioSocketChannel的数据读写，方法调用从顶层到底层分析：
1. IO操作，包括read, write，bind, connect，channel将message交给pipeline，在顶层基类AbstractChannel类实现：
![在这里插入图片描述](https://img-blog.csdnimg.cn/2018122100245840.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAwMTM1NzM=,size_16,color_FFFFFF,t_70)
Channel接口包含一个ChannelPipeline类型的pipeline属性，流经channel的所有数据都是需要在pipeline中进行传输的。
2. ChannelPipeline维护了ChannelHandler处理链，包括head和tail ChannelHandler，如图：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181221003051484.png)
其中head和tail都是ChannelHandlerContext，head和tail是Netty只身提供的处理链的头部和尾部，这两个是固定的，中间为用户定义和添加的一个或多个ChannelHandler。ChannelHandlerContext是封装了ChannelHandler实例的，如图：
![在这里插入图片描述](https://img-blog.csdnimg.cn/2018122100341894.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAwMTM1NzM=,size_16,color_FFFFFF,t_70)
ChannelHandlerContext主要用于ChannelHandler和ChannelPipeline的交互，以及建立ChannelPipeline中多个ChannelHandler之间的关联，如图接口源码注释：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181221103907313.png)
通过prev，next方法可以获取到ChannelPipeline中前一个或下一个ChannelHandler。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181221004123843.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAwMTM1NzM=,size_16,color_FFFFFF,t_70)
3. ChannelPipeline中的read, write, bind, connect，均交给ChannelHandleContext执行，关于ChannelPipeline的源码具体分析，在之后再分享。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181221004418717.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAwMTM1NzM=,size_16,color_FFFFFF,t_70)
4. ChannelHandlerContext的read操作：执行数据读取，即在channel所在的eventLoop中对应的线程EventExecutor中执行invokeRead()
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181221004748467.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAwMTM1NzM=,size_16,color_FFFFFF,t_70)
invokeRead操作：通过invokeHandler方法，判断当前ChannelHandlerContext是否已经添加过了ChannelHandler，如果有则调用真正的ChannelHandler执行read操作；没有则调用ChannelHandlerContext自身的read方法，继续交给next的ChannelHandlerContext。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181221004950951.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAwMTM1NzM=,size_16,color_FFFFFF,t_70)
5. ChannelHandler的read操作：以ChannelHandlerContext作为参数，在read方面里面可以自定义处理逻辑，然后再调用ChannelHandlerContext的read等传给ChannelPipeline中的下一个ChannelHandler继续处理，直到最后一个ChannelHandler，即上面介绍的head或tail。或者不调用ctx.read了，不再传给下一个channelHandler，如果是write，也可以直接调用Channel的write或writeAndFlush直接写到socket中，不再传给下个ChannelHandler。
6. 最后通过具体的Channel实现类，如NioSocketChannel，在相应的doConnect，doBind等方法中，调用Java的SocketChannel完成最后实际数据的read，write，bind, connect等，如图NioSocketChannel的doWrite：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181221103649641.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAwMTM1NzM=,size_16,color_FFFFFF,t_70)