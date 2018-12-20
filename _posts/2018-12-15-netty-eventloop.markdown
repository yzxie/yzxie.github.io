---
layout:     post
title:      "Netty源码分析-线程模型与EventLoop事件循环机制"
subtitle:   "Netty EventLoop"
date:       2018-12-15 07:12:00
author:     "XYZ"
header-img: "img/article-bg1.jpg"
tags:
    - Netty源码分析
---
### 线程模型
对于处理channel的IO读写事件，Netty提供了EventLoopGroup和EventLoop两个接口。
* EventLoop是一个IO线程，但是与普通IO线程不一样的是，注册到或者说绑定到这个线程的channel，在该channel的整个生命周期内，即客户端和服务端的交互，都是在这个eventLoop线程处理的，不会切换到其他线程，这样就没有线程安全问题，这也是命名为eventLoop（IO事件循环）的原因。一个EventLoop线程可以处理多个channel的IO事件，但是一个channel只能绑定到一个EventLoop线程。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181220121213190.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAwMTM1NzM=,size_16,color_FFFFFF,t_70)
* EventLoopGroup：eventLoop线程池实现，继承于ScheduledExecutorService，维护一个eventLoop线程池，在register方法中，通过调用next方法从线程池中获取一个eventLoop线程，然后将channel绑定到该eventLoop线程。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181220121420719.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAwMTM1NzM=,size_16,color_FFFFFF,t_70)

### NioEventLoopGroup：事件处理线程池
NioEventLoopGroup底层是基于NIO，即非阻塞IO的。
* 线程池：NioEventLoopGroup继承于MultithreadEventLoopGroup，MultithreadEventLoopGroup提供了线程池实现，默认线程池大小为：机器处理器个数的2倍，如下：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181220134959913.png)
* SelectorProvider：selectorProvider用来创建Java NIO selector实例，类似于一个selector的工厂类。
   * selectorProvider自身实例是通过SelectorProvider的静态provider方法创建的，且该静态provider方法线程安全的，使用了synchronized锁。
   * selectorProvider实例只创建一次，后续调用provider方法，则直接返回，所以该selectorProvider实例是被所有EventLoopGroup所共享的。selectorProvider的具体类型，则可以根据应用自身指定，如下方法：loadProviderFromProperty，loadProviderAsService，如果都没有指定，则根据操作系统来获取，如操作系统是Linux，2.6及以上版本内核使用EPollSelectorProvider，以下则是使用PollSelectorProvider。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181220143010959.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAwMTM1NzM=,size_16,color_FFFFFF,t_70)
   * selectorProvider的所有实例方法都是线程安全，如下创建selector用到的openSelector。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181220145204278.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAwMTM1NzM=,size_16,color_FFFFFF,t_70)
   * selector的创建：selectorProvider生产selector，具体为通过openSelector方法创建的，这个是抽象实例方法，不是静态方法，由具体selectorProvider实现类，如EPollSelectorProvider，负责实现创建selector的逻辑，如下：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181220143651314.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAwMTM1NzM=,size_16,color_FFFFFF,t_70)
在每个NioEventLoop中都包含一个NIO selector，eventLoop通过该selector获取绑定到该eventLoop的channels的IO事件。

### NioEventLoop：事件处理线程
NioEventLoop可以被多个channels绑定，channel将自己注册到NioEventLoop实例的selector，通过selector监听和获取该channel的IO事件。（每个channel只能绑定到一个eventLoop）每个channel整个生命周期内的IO事件都由该EventLoop线程处理。
* NIO selector: Java NIO Selector是一个IO多路复用器，是实现非阻塞IO的核心。NioEventLoop自身包含一个Java NIO的selector实例，由该selector负责监听所有绑定到该eventLoop线程的channel的所有IO事件。如下NioEventLoop的selector相关的字段：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181220150503831.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAwMTM1NzM=,size_16,color_FFFFFF,t_70)
* 单线程模型
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181220150337381.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAwMTM1NzM=,size_16,color_FFFFFF,t_70)
### 事件循环机制：channel整个生命周期IO事件由同一个eventLoop线程处理
简单来说，就是创建channel时，将channel注册到NioEventLoop的selector， NioEventLoop监听自身的selector，获取有IO事件到来的channel并处理。由于NioEventLoop自身就是一个线程实现类，故channel整个生命周期的IO事件都是在这个线程处理的，而不会由其他线程处理。
具体过程如下分析：
#### channel绑定NioEventLoop线程，其实是绑定到NioEventLoop线程的selector
1. EventLoopGroup从eventLoop线程池获取一个eventLoop线程：
由上面的分析以及Bootstrap源码分析[Netty源码分析-BootStrap服务启动类](https://blog.csdn.net/u010013573/article/details/84726991)可知：channel是通过注册到EventLoopGroup，然后由EventLoopGroup从自身所管理的eventLoop线程池中获取一个eventLoop线程，然后将channel绑定到这个eventLoop线程的。具体如何将channel绑定到eventLoop线程是由eventLoop自身控制的。如下图：next()方法为返回一个eventLoop。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181220141142580.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAwMTM1NzM=,size_16,color_FFFFFF,t_70)
2. eventLoop线程通过register方法，完成channel到eventLoop的绑定：将eventLoop作为参数，调用channel自身的register完成到eventLoop线程的绑定：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181220152905834.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAwMTM1NzM=,size_16,color_FFFFFF,t_70)
3. channel的register：在AbstractChannel中实现，channel自身类实现绑定到eventLoop的逻辑。
   1. 对channel的eventLoop进行赋值，然后看当前执行线程是否就是eventLoop线程，是则直接执行register0；不是则使用eventLoop.execute -> register0
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181220154013411.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAwMTM1NzM=,size_16,color_FFFFFF,t_70)
   2. register0：将该channel注册到eventLoop的selector中，具体为在doRegister方法完成channel到selector的注册，doRegister为抽象方法，由具体实现类，即selector的类型，实现。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181220154312448.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAwMTM1NzM=,size_16,color_FFFFFF,t_70)
   3. doRegister的实现类
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181220154602313.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAwMTM1NzM=,size_16,color_FFFFFF,t_70)
   4. AbstractNioChannel的doRegister实现
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181220155329546.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAwMTM1NzM=,size_16,color_FFFFFF,t_70)
   5. javaChannel().register方法：selector注册channel并产生select key，添加到select key集合，最终完成channel到eventLoop的selector的注册。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181220155500312.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAwMTM1NzM=,size_16,color_FFFFFF,t_70)
#### NioEventLoop对channel的IO请求处理
1. NioEventLoop在run方法for循环轮询，监听有IO事件的Select keys集合
![在这里插入图片描述](https://img-blog.csdnimg.cn/2018122015122589.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAwMTM1NzM=,size_16,color_FFFFFF,t_70)
2. processSelectedKeys方法处理对应channel的IO事件，方法调用关系如下：
processSelectedKeys -> processSelectedKeysPlain(selector.selectedKeys()) ->
processSelectedKey(SelectionKey k, AbstractNioChannel ch)，该过程均在NioEventLoop线程内完成。
3. processSelectedKey(SelectionKey k, AbstractNioChannel ch)：获取到对应的channel和事件类型SelectedKey，在eventLoop线程中使用channel来完成数据的读写：底层调用如下，获取到对应的channel和事件类型SelectedKey，在eventLoop线程中使用channel来完成数据的读写：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181220151851213.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAwMTM1NzM=,size_16,color_FFFFFF,t_70)

