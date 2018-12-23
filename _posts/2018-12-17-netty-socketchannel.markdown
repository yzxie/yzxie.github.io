---
layout:     post
title:      "Netty源码分析-服务端创建SocketChannel的底层实现原理"
subtitle:   "Netty SocketChannel"
date:       2018-12-17 07:12:00
author:     "XYZ"
header-img: "img/article-bg1.jpg"
tags:
    - Netty源码分析
---
### 概述
* SocketChannel在服务端用于处理客户端的IO事件，即ServerSocketChannel接收到客户端的连接请求后，创建SocketChannel用于后续该客户端和服务端之间的IO请求处理。
* 服务端是通过ServerSocketChannel来监听客户端的连接请求并创建SocketChannel的，故ServerSocketChannel的pipeline中流通的数据msg是SocketChannel。pipeline中的最后一个ChannelHandler为ServerBootstrapAcceptor：ServerBootstrapAcceptor为ChannelInBoundHandler
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181222002833957.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAwMTM1NzM=,size_16,color_FFFFFF,t_70)
在ServerBootstrapAcceptor的channelRead方法中，完成对SocketChannel的初始化，包括为SocketChannel的ChannelPipeline添加ChannelHandler，添加ChannelOptions，Attributes和从childGroup线程池中获取一个eventLoop线程，并将该channel绑定到该eventLoop，由该eventLoop线程处理该SocketChannel整个生命周期的IO。如图：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181222000050653.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAwMTM1NzM=,size_16,color_FFFFFF,t_70)
* 但是从源码可知，child是通过Object了下的msg通过强制转换而来的，也就是说在调用ServerBootstrapAcceptor的channelRead时，msg是已经创建好了的，为SocketChannel类型的变量，但是底层是在哪里创建好的呢？而ServerSocketChannel也是注册在bossEventLoopGroup的一个eventLoop的selector中的，通过该selector监听客户端的连接请求事件，即通过SelectKey获取OP_READ或OP_ACCEPT事件，如下：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181222000607584.png)
从selector的监听到IO事件到传到ChannelHandler处理的基本流程是：
selector -> channel -> channelPipeline -> channelHandlers，而ServerBootstrapAcceptor是最后一个ChannelHandler。其中服务端和客户端之间通信所用的SocketChannel为在channel的read方法中创建，具体看以下分析。
### 服务端创建SocketChannel的底层实现
1. NioEventLoop的selector监听到连接事件：其中ch为ServerSocketChannel
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181221174202870.png)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181221174135609.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAwMTM1NzM=,size_16,color_FFFFFF,t_70)
2. ServerSocketChannel的unsafe实现
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181221174328244.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAwMTM1NzM=,size_16,color_FFFFFF,t_70)
3. newUnsafe方法，根据具体的ServerSocketChannel的返回对应的unsafe实例，注意unsafe是ServerSocketChannel的一个门面类，里面封装了一个channel实例，只是说明unsafe的方法只能在内部使用而已，如果在外部使用是不安全。以下以NioServerSocketChannel为例：由1知，调用的是unsafe.read()，具体read方法内会调用doReadMessages方法，doReadMessages方法的实现如下：
（1）通过SocketUtils.accept(javaChannel())：从Java NIO的ServerSocketChannel中获取Java NIO的SocketChannel；
（2）通过new NioSocketChannel，从Java NIO的SocketChannel创建Netty的SocketChannel实例，其中NioSocketChannel为SocketChannel的一个实现类，故在这里创建了Netty的SocketChannel对象，然后在pipeline中的流通的数据Object msg，就是SocketChannel类型。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181222004614135.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAwMTM1NzM=,size_16,color_FFFFFF,t_70)
（3）通过pipeline.fireChannelRead(readBuf.get(i))将新创建的NioSocketChannel交给NioServerSocketChannel的pipeline：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181221174453571.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAwMTM1NzM=,size_16,color_FFFFFF,t_70)
其中ServerBootstrapAcceptor为pipeline中的最后一个ChannelHandler，如图在创建ServerSocketChannel时添加的：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181221175216842.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAwMTM1NzM=,size_16,color_FFFFFF,t_70)
5. 故可以在ServerBootstrapAcceptor的channelRead中将msg强制转换为Channel类型。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181221175343652.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAwMTM1NzM=,size_16,color_FFFFFF,t_70)