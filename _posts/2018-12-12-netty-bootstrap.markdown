---
layout:     post
title:      "Netty源码分析-BootStrap服务启动类"
subtitle:   "Netty Bootstrap"
date:       2018-12-12 07:12:00
author:     "XYZ"
header-img: "img/article-bg1.jpg"
tags:
    - Netty源码分析
---
### AbstractBootStrap：启动基类
该类主要定义了客户端和服务端启动netty均需要的字段和方法，核心字段包括：
* EventLoopGroup：线程池，如果是服务端，则在拓展类ServerBootstrap中可以选择再定义一个childEventLoopGroup，用于处理已建立连接的客户端，之后的IO请求。EventLoopGroup线程池在服务端主要为acceptor提供执行线程，acceptor处理客户端的连接请求，在客户端则为connect服务端提供执行线程，通常EventLoopGroup的数量为1即可：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181220003725185.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAwMTM1NzM=,size_16,color_FFFFFF,t_70)
* ChannelFactory：channel工厂，泛型类，根据泛型类型，生成实际的channel，如NioServerSocketChannel、EpollServerSocketChannel。根据channel方法指定的channel类型，确定生产哪种channel。对服务端来说，生产监听客户端连接的acceptor channel，即ServerSocketChannel（即对应socket编程中的ServerSocket）：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181220003607128.png)
客户端则生产发起与服务端连接的channel：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181220003645245.png)
* ChannelHandler：channel的事件处理器，处理channel的读写请求，其中Inbound类型则是处理读，OutBound处理写，Duplex处理读写，在该handler中自定义处理逻辑。在服务端则是处理客户端的连接请求，同时在拓展类ServerBootstrap中还定义一个childChannelHandler，这个是与已建立连接的客户端对应的channel的处理器，可以通过实现ChannelInitializer的initChannel方法来添加更多的handler来处理：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181220004826944.png)
在客户端则是处理客户端channel的读写请求。
* ChannelOptions：保存以上channel的选项，如设置buffer的allocator，tcp的keepAlive， nodelay，通过类而不是字符串来实现类型安全。
* AttributeKey：保存在channel中的属性，在整个连接周期中可以访问，通常为业务数据，如长连接中保存一些状态数据，如设备Id之类的。
* localAdress：绑定的地址，服务端为acceptor监听地址，客户端为建立连接的地址。
#### 核心方法：bind和initAndRegister
* 方法实现的调用关系：bind -> doBind -> initAndRegister，doBind0 -> init，group.register
* 其中initAndRegister调用init，group.register完成channel的初始化和绑定到一个eventLoop线程，对服务端而言是从bossEventLoopGroup中获取一个线程；doBind0完成channel的socket地址的绑定。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181220012627575.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAwMTM1NzM=,size_16,color_FFFFFF,t_70)
* bind
   * 客户端：UDP连接时才会调用bind方法绑定本地端口；如果是TCP连接，则使用拓展类Bootstrap提供的connect方法连接服务端；
   * 服务端：创建ServerSocketChannel，为ServerSocketChannel绑定监听端口，监听客户端的连接请求；
* initAndRegister
完成channel的创建，通过init方法完成初始化，通过group().register(channel)，从eventLoopGroup获取eventLoop线程，由该eventLoop处理该channel整个生命周期的IO请求
  ![在这里插入图片描述](https://img-blog.csdnimg.cn/2018122001234185.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAwMTM1NzM=,size_16,color_FFFFFF,t_70)
   * init：channel初始化
初始化channel，服务端和客户端不相同，客户端比较简单，主要是将bootstrap.handler传进的handler放到客户端channel的pipeline中：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181220011000596.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAwMTM1NzM=,size_16,color_FFFFFF,t_70)
服务端的init方法则比较复杂，具体有ServerBootstrap实现，具体看下面ServerBootstrap部分的分析。
   * group().register(channel)：线程绑定
eventLoop绑定，从eventLoopGroup中获取一个特定线程eventLoop，该线程负责处理该channel整个生命周期的IO事件。
### ServerBootStrap：服务端启动类
* ServerBootStrap为AbstractBootStrap的实现类，服务端启动类，ServerSocketChannel接收客户端连接，SocketChannel处理已成功建立连接的客户端后续的请求。这两个功能类似于socket编程中的ServerSocket和Socket，即接收连接的channel为parent channel，而由他接收和建立连接生成的channel为child channel。为了拓展性，即提供区分对待这两种channel的拓展性，在ServerBootStrap中，定义了childHandler, childGroup, childOptions, childAttrs用于处理child channel。
* init方法实现：初始化ServerSocketChannel
1. 为服务端的ServerSocketChannel添加handlers，其中最后一个handler是ServerBootstrapAcceptor；
2. acceptor handler创建：构造childGroup，childHandler作为参数，创建ServerBootstrapAcceptor，ServerBootstrapAcceptor用来accept客户端的连接请求，然后创建对应的SocketChannel，该SocketChannel用于处理该客户端连接后续的读写请求。
  ![在这里插入图片描述](https://img-blog.csdnimg.cn/20181220011600568.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAwMTM1NzM=,size_16,color_FFFFFF,t_70)
  
#### ServerBootstrapAcceptor：服务端请求接收器
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181202195141697.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAwMTM1NzM=,size_16,color_FFFFFF,t_70)
* ServerBootstrapAcceptor为ChannelnBoundHandlerAdapter的实现类，在服务端的监听channel，即ServerSocketChannel，的pipeline中，为第二个或第一个channelInBoundHandler，在有新的客户端connect请求到来时，调用channelRead方法处理，创建和初始化类型SocketChannel的child channel，然后从childEventLoopGroup中获取一个eventLoop线程，该线程为IO线程，由该线程负责处理该child channel后续的IO请求。源码如下：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181202195221367.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAwMTM1NzM=,size_16,color_FFFFFF,t_70)
* 源码解读：因为这个是ServerSocketChannel，accept客户端连接的handler，故将msg转为channel，为该channel设置child系列options，attrs等，然后将该child channel注册到childGroup，如上图源码255行：
*childGroup.register(child)*
childGroup和监听channel可为同一个线程池，也可以是不同的两个线程池，由用户代码指定，即如果调用group(group）为同一个，调用group(parentGroup, childGroup)分别指定两个不同的线程池。childGroup为该child channel分配一个线程，该child channel的整个生命周期的IO事件均在这个线程中处理，而不会切换到其他线程，所以没有线程安全问题，不需要使用到线程同步。
### BootStrap：客户端启动类
* BootStrap：客户端启动类，定义客户端连接服务器逻辑，也是AbstractBootStrap类的实现类，由于连接服务端和接收服务端发送的数据为同一个channel，所以不需要额外定义child系列字段。
* 域名解析：客户端需要进行域名解析，故定义了AddressResolveGroup，用于进行域名解析。处于线程安全的考虑，每个eventLoop对应一个addressResolve。
* 核心方法为：connect和doResolveAndConnect0
doResolveAndConnect0：分为地址解析和connect服务端两步，均在该channel绑定的同一线程执行。
1. 地址解析：在地址解析为异步操作，返回Future。阻塞该future直到解析完成后，从该future获取remotAddress，然后使用remoteAddress作为参数调用doConnect进行实际服务端的连接操作。核心源码如下：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181202195350487.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAwMTM1NzM=,size_16,color_FFFFFF,t_70)
2. doConnect: 完成实际到服务端的socket连接，在eventLoop线程中执行，也是异步操作，通过connectPromise实现异步回调。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181220102626903.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAwMTM1NzM=,size_16,color_FFFFFF,t_70)