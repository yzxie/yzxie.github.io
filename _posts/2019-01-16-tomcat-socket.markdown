---
layout:     post
title:      "Tomcat源码分析（五）：Socket和线程模型体系结构设计"
subtitle:   "Tomcat Socket and Worker threads"
date:       2019-01-16 08:12:00
author:     "XYZ"
header-img: "img/article-bg1.jpg"
tags:
    - Tomcat源码分析
---
### 概述
* Tomcat在设计当中，自顶向下主要包括：Catalina容器，Coyte连接器和底层Socket通信端点EndPoint三部分组成。底层Socket通信端点EndPoint主要完成socket通信的相关细节和整个Tomcat框架线程模型的实现。
* 服务启动：Tomcat启动时，从Catalina容器开始启动，往下依次创建和启动Coyote连接器，创建服务端监听请求socket和请求处理工作线程池。
* 请求处理：在客户端发起连接请求时，则是服务端监听请求socket先监听到客户端连接请求，然后处理socket三次握手等细节，将对应的channel注册NIO的selector，由selector负责监听该客户端后续数据请求到来。当客户端发起数据请求时，则从工作线程池获取一个线程，由该线程负责该请求整体的处理过程，包括通过coyte连接器传送到Catalina容器，交给对应的应用处理。

### Socket体系和线程模型
* 这里主要基于NIO的socket对Tomcat的Socket和线程模型体系结构进行分析。
* 在Tomcat的体系结构设计当中，上层的Catalina容器和Coyte连接器主要是对Servlet规范的实现，其中并不涉及底层通信相关的数据处理和线程分配逻辑，而这部分主要是在底层socket通信Endpoint来完成。

#### Endpoint
* Tomcat主要通过Endpoint来抽象底层整体的socket通信和线程模型设计，在抽象类AbstractEndpoint中负责定义
   1. socket通信tcp相关和线程相关的属性的默认值；
   2. socket请求监听器Acceptor；
   3. socket数据处理器Handler；
   4. 工作线程池Executor；
   5. Endpoint整体的初始化init，监听端口绑定，启动start，关闭stop的骨架实现。

#### 启动流程
* 绑定服务端请求监听端口
* 创建工作线程池executor
* 启动Poller线程
* 启动Acceptor线程

#### Acceptor线程
* 线程数量默认为1
* 接收客户端连接请求，然后将该请求对应的channel绑定到Poller的selector中。

#### Poller线程
* 线程数量为2和处理器数量之间的最小值
* 每个poller内部包含一个selector，由该selector监听绑定到其上的channel的selectionKey来获取读写事件。
* 在NIO2中已经不存在Poller了，Acceptor接收到客户端连接请求后，创建一个与该连接请求对应的SocketProcessor，从工作线程池获取一个线程，将该SocketProcessor在该线程执行。

#### Executor工作线程池
* 线程数量默认最大为200，初始化为10.
* Poller监听到客户端socket的数据读写请求时，创建一个SocketProcessor处理器，然后从工作线程池Executor中获取一个线程，让该SocketProcessor在这个线程中执行。在SocketProcessor中，通过Handler来将请求往上传给Coyte连接器和Catalina容器。

#### NioSocketWrapper
* 对Socket进行包装，添加poller等的引用，以及定义字节数据读写的方法。

#### SocketProcessor
* Socket数据处理器，实现Runnable接口，在run方法中对socket请求数据进行处理，以及调用Handler将数据往上传给Coyte连接器和Catalina容器。
* 从工作线程池Executor中获取一个工作线程来执行以上过程。

#### Handler
* 链接Endpoint和Coyte连接器和Catalina容器，将底层数据往上传给Catalina容器处理。

### TCP和连接相关默认值
#### 连接数相关
* 最大连接数：10000，在NIO2中为-1，表示不限制。

#### http和tcp相关
* KeepAlive最大请求数：100
* TCP backlog：100
* TCP nodelay：true
* Http请求headers最大数量：100