---
layout:     post
title:      "Dubbo源码分析：远程传输remoting"
subtitle:   "Dubbo transporter"
date:       2018-12-05 16:12:00
author:     "XYZ"
header-img: "img/article-bg1.jpg"
tags:
    - Dubbo源码分析
---
### 概述
由rpc协议实现模块分析可知，远程传输模块主要为rpc协议中的dubbo协议提供：服务提供者和服务消费者之间的数据传输功能。远程通讯对外提供了一个Exchange的概念，即消息交换。
* 服务消费者：包含一个ExchangeClient，通过connect方法连接服务提供者，并指定对应的ExchangeHandler用于远程调用的处理，将请求的方法和参数通过URL的方式，传递给服务提供者；
* 服务提供者：包含一个ExchangeServer，通过bind方法绑定到指定URL和端口，用于监听服务消费者的连接和远程调用请求，并指定ExchangeHandler用于处理请求，根据请求的方法和参数调用本地方法，获取结果响应给服务消费者。
如下Exchange也是SPI实现，根据adaptive动态选择对应的Exchange实现：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181207162805691.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAwMTM1NzM=,size_16,color_FFFFFF,t_70)
### 整体架构
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181207162825610.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAwMTM1NzM=,size_16,color_FFFFFF,t_70)
上图主要为客户端和服务端的一个整体设计，即每个客户端Client通过Channel与服务端Server通信，通过ChannelHandler处理Channel相关事件；服务端Server维护与客户端连接的Channel集合。
* Client与Server的底层通信是通过Transporter实现的，Transporter完成bind绑定创建Server对象，完成connect连接服务端创建Client对象。如下：Transporters静态工具类
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181207162841393.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAwMTM1NzM=,size_16,color_FFFFFF,t_70)
* 整体调用关系
rpc模块 -> Exchangers.bind/connect -> Transporters.bind/connect -> Server/Client
除Transporter实现可根据URL选择对应的SPI实现外，其他基本为固定模式；Server和Client根据Transporter的不同实现，而使用不同的Server，Client和Channel，具体在传输模块分析。虽然Exchange为SPI，支持Adaptive，Exchange的实现只用HeaderExchange。
* 总体设计思路
1. 最外层抽象设计为：Endpoint，Client，Server，Channel，即通信的端点，根据角色区分客户端和服务端，客户端，服务端直接通过一个管道Channel进行数据流通；
2. Client，Server等都是一些基础设计，定义了基本connect, bind, send, received等实现通信的基本方法，为了给外部提供一个完整的组件，这些完整的组件包括其他更加具体的功能，如长连接心跳机制等，此时这个设计就是exchange机制，即这是一个进行数据交换的客户端和服务端。exchange包括Exchanger，ExchangeClient，ExchangeServer，其中Exchanger代表一个数据交换器，在服务端通过Exchanger绑定指定的URL创建ExchangeServer，在客户端连接服务端创建ExchangerClient。而为了方便提供给外部使用，使用Exchangers静态工具类来创建Exchanger实现其功能。
3. Exchanger数据交换器在底层在依赖于一个数据传输层进行实际的数据传输，这个就是Transporter，同时包括与Transpoter相关的Server和Client。数据传输Transporter可以支持多种方式。同时为了方便给Exchanger调用，也是使用Transporters静态工具类来获取Transporter实例。具体下面分析。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181207162903184.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAwMTM1NzM=,size_16,color_FFFFFF,t_70)
### 传输
* 传输层的接口设计
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181207162924968.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAwMTM1NzM=,size_16,color_FFFFFF,t_70)
SPI接口，根据URL的transporter参数动态指定使用哪个拓展实现；默认为netty。
bind：服务端绑定指定端口，接收客户端的请求；
connect：客户端连接服务端
具体传输实现则实现这个接口，并返回自身的Client和Server实现。
* 服务端和客户端设计
采用模板设计模式，提供了抽象类AbstractServer，AbstractClient，分别指定了服务端和客户端实现的一个基本框架，具体实现类实现具体逻辑。底层通过Channel进行实际通讯。
   * 服务端实现类
   1. 创建服务端对象实例，具体实现类实现抽象方法doOpen，完成实际服务端实例创建
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181207162947443.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAwMTM1NzM=,size_16,color_FFFFFF,t_70)
   2. 接收连接：判断是否超过了最大可接收连接，没有则调用super.connected(ch)，通过channelHandler来完成channel的创建。断开逻辑类似。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181207162959231.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAwMTM1NzM=,size_16,color_FFFFFF,t_70)
   ![在这里插入图片描述](https://img-blog.csdnimg.cn/20181207163048231.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAwMTM1NzM=,size_16,color_FFFFFF,t_70)
* 客户端实现类
1. connect：创建与服务端的连接，通过connectLock重入锁实现线程并发控制，抽象方法doConnect由实现类实现特定的连接服务端逻辑。
2. 发送请求：通过channel发送请求给服务端。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181207163104291.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAwMTM1NzM=,size_16,color_FFFFFF,t_70)

### netty传输
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181207163120116.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAwMTM1NzM=,size_16,color_FFFFFF,t_70)
bind创建NettyServer，connect创建NettyClient。
NettyServer实现doOpen：创建netty的ServerBootstrap对象
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181207163135126.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAwMTM1NzM=,size_16,color_FFFFFF,t_70)
NettyClient实现doOpen创建netty的Bootstrap对象，实现doConnect连接服务端。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181207163146376.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAwMTM1NzM=,size_16,color_FFFFFF,t_70)
NettyHandler捕获NettyChannel的读写事件，并进行处理。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181207163157745.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAwMTM1NzM=,size_16,color_FFFFFF,t_70)
NettyChannel实现底层数据的传输：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181207163211586.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAwMTM1NzM=,size_16,color_FFFFFF,t_70)

### zookeeper传输
服务端和客户端都是作为zookeeper的client，通过在zookeeper创建分支或者写入数据，监听分支或分支内容变化，获取数据来实现通讯。

### p2p传输
类似于一个多播组通讯。