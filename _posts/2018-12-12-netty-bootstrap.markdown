---
layout:     post
title:      "Netty源码分析-BootStrap服务启动类"
subtitle:   "Netty Bootstrap"
date:       2018-12-02 07:12:00
author:     "XYZ"
header-img: "img/article-bg1.jpg"
tags:
    - Netty源码分析
---
## 启动基类：AbstractBootStrap
该类主要定义了客户端和服务端启动netty均需要的字段和方法，核心字段包括：
* EventLoopGroup：线程池，如果是服务端则在拓展类ServerBootstrap中可以选择再定义一个childEventLoopGroup，用于处理已建立连接的客户端的请求。该线程池在服务端主要为acceptor提供执行线程，执行客户端的连接请求，而在客户端则为建立连接提供线程。
* ChannelFactory：channel工厂，泛型类，根据泛型类型，生成实际的channel，如NioServerSocketChannel、EpollServerSocketChannel，对于服务端来说则是生产监听客户端连接的acceptor channel，即ServerSocketChannel（即对应socket编程中的ServerSocket）；对应客户端则生产发起与服务端连接的channel。
* ChannelHandler：channel的事件处理器，即接收客户端请求，然后在该handler中自定义处理逻辑。
* ChannelOptions：保存以上channel的选项，如设置buffer的allocator，tcp的keepAlive， nodelay等。
* AttributeKey：保存在channel中的属性，在整个连接周期中可以访问，如长连接中保存一些状态数据，如设备Id之类的。
* localAdress：绑定的地址，服务端为acceptor监听地址，客户端为建立连接的地址。
### 核心方法：bind和initAndRegister
* bind：创建channel并从eventloopgroup线程池中获取线程，该线程负责处理该channel整个生命周期的io事件；第二步为在第一步成功之后，根据localAddress绑定到实际的ip和port；
* initAndRegiter：bind方法在第一步中需要调用initAndRegiter，initAndRegister调用channelFactory创建channel，并调用init（这是一个protected的abstract方法，由具体实现类负责定义）初始化该channel；然后将该channel注册到线程池eventLoopGroup中，实际为从线程池获取线程，然后建立该channel和线程的绑定关系。以上注册操作为异步操作，返回ChannelFuture。该方法使用了模板设计模式，可变部分有init定义。
* initAndRegiter完成之后，后续的bind或connect均在该注册到的线程中进行处理。源码如下：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181202194759715.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAwMTM1NzM=,size_16,color_FFFFFF,t_70)
### ServerBootStrap：服务端启动类
* ServerBootStrap为AbstractBootStrap的实现类，服务端启动类，接收客户端连接，处理已成功建立连接的客户端后续的请求。这两个功能类似于socket编程中的ServerSocket和Socket，即接收连接的channel为parent channel，而由他接收和建立连接生成的channel为child channel。为了拓展性，即提供区分对待这两种channel的拓展性，在ServerBootStrap中，定义了childHandler, childGroup, childOptions, childAttrs用于处理child channel。
* 核心逻辑
init方法的实现和内部类ServerBootstrapAcceptor，该内部类是一个ChannelInBoundHandler，用于处理channel监听到的事件，即处理客户端连接请求。
init方法：源码核心逻辑如下：
![在这里插入图片描述](https://img-blog.csdnimg.cn/2018120219502556.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAwMTM1NzM=,size_16,color_FFFFFF,t_70)
即初始化为监听客户端请求的channel，其中添加了两个channelHandler，首先为其添加由用户初始化指定的channelHandler，然后在添加ServerBootStrapAcceptor，同时构造参数为child系列group，childHandler，options，attrs，因为child系列的，是为新创建的child channel服务。
* ServerBootstrapAcceptor
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181202195141697.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAwMTM1NzM=,size_16,color_FFFFFF,t_70)
ServerBootstrapAcceptor为ChannelnBoundHandlerAdapter的实现类，在监听channel的pipeline中，为第二个或第一个channelInBoundHandler，在监听channel有数据可读时，即有新的客户端连接到来时，调用channelRead方法处理。源码如下：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181202195221367.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAwMTM1NzM=,size_16,color_FFFFFF,t_70)
因为这个是用于accept客户端连接的handler，故将msg转为channel，为该channel设置child系列options，attrs等，然后将该child channel注册到childGroup，childGroup和监听channel可为同一个线程池，也可以是不同的两个线程池，由用户代码指定（即调用group(group）或group(parentGroup, childGroup)）。childGroup为该child channel分配一个线程，该child channel的整个生命周期的IO事件均在这个线程中处理，而不会切换到其他线程，所以没有线程安全问题，不需要使用到线程同步。
### BootStrap：客户端启动类
* BootStrap：客户端启动类，定义客户端连接服务器逻辑，也是AbstractBootStrap类的实现类，由于连接服务端和接收服务端发送的数据为同一个channel，所以不需要额外定义child系列字段，但是客户端需要进行域名解析，故定义了AddressResolveGroup，用于进行域名解析。处于线程安全的考虑，每个eventLoop对应一个addressResolve。
* 核心方法为：connect和doResolveAndConnect0
doResolveAndConnect0：分为地址解析和connect服务端两步，均在该channel绑定的同一线程执行。
在地址解析为异步操作，返回Future。阻塞该future直到解析完成后，从该future获取remotAddress，则调用doConnect进行实际服务端的连接操作。核心源码如下：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181202195350487.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAwMTM1NzM=,size_16,color_FFFFFF,t_70)
