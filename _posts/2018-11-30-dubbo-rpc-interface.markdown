---
layout:     post
title:      "Dubbo源码分析：RPC协议实现-RPC过程与核心接口设计"
subtitle:   "Dubbo RPC Interface"
date:       2018-11-30 15:12:00
author:     "XYZ"
header-img: "img/article-bg1.jpg"
tags:
    - Dubbo源码分析
---
## RPC的基本过程
* 提供者Provider：提供服务的接口定义和接口的具体实现，然后通过URL的方式告诉消费者，某个URL对应某个service实现，一般是将服务的信息注册到一个注册中心，如zookeeper或者Redis等；
* 消费者Consumer：获取提供者的接口定义，一般通过引入jar包的方式，从而获得提供者的接口和方法定义，如方法参数个数，参数类型；在消费者本地封装数据，如方法参数值，对提供者提供的服务进行远程调用，而这个远程调用是通过消费者本地的一个代理对象进行的，即将需要调用的方法，进行方法调用的数据，交给该代理对象，由该代理对象发起对提供者的请求，获取提供者的响应，然后将请求结果返回给消费者。 
* 整个过程如下：其中1. register提供者注册，4.invoker是由消费者调用均是通过代理完成的。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181202154913453.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAwMTM1NzM=,size_16,color_FFFFFF,t_70)
远程调用过程：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181202155029476.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAwMTM1NzM=,size_16,color_FFFFFF,t_70)
* 核心接口
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181201233410370.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAwMTM1NzM=,size_16,color_FFFFFF,t_70)
## 核心接口设计
### Protocol：RPC协议接口
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181201172813269.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAwMTM1NzM=,size_16,color_FFFFFF,t_70)
* export方法
   * 模板方法，提供者需要实现export方法，具体为AbstractProxyProtocol，定义如何将自己提供的服务暴露出去，同时设置消费者访问调用请求的requestHandler，处理消费者的远程调用请求。
   * 具体是由AbstractProxyProtocol的实现类定义和实现doExport方法完成实际注册操作，如HttpProtocol协议实现如下：
（1）httpBinder.bind(url, new InternalHandler())：监听消费者请求，InternalHandler负责处理请求和进行相应；
（2）createExporter(impl, type)：进行服务注册。
![在这里插入图片描述](https://img-blog.csdnimg.cn/2018120216440596.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAwMTM1NzM=,size_16,color_FFFFFF,t_70)

* refer方法
   * 模板方法，消费者实现refer方法，具体为AbstractProxyProtocol，提供自己需要调用的远程服务的class类型，provider暴露服务的url地址，获得对远程服务进行实际服务调用的代理，即该代理对服务消费者而言，就是服务提供者，代理在服务提供者和服务消费者之间作为中间人，搭建了一个服务调用的桥梁。
   * 具体的服务调用由AbstractProxyProtocol的实现类定义和实现doRefer方法实现，如HttpProtocol协议实现如下：
构造httpProxyFactoryBean实例，设置好调用的远程服务接口名，服务暴露的URL，调用相关数据，如参数值，通过httpProxyBean完成服务调用。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181202165204481.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAwMTM1NzM=,size_16,color_FFFFFF,t_70)


### Provider相关：服务提供者提供服务
* Exporter：服务暴露接口，该接口表示提供者将服务暴露或者说注册到了一个注册中心，同时可以取消、回收服务的注册。
* ExporterListener：服务暴露监听器，在对服务进行暴露后，或者取消已经暴露的服务，进行回调处理。
### Consumer相关：服务消费者消费服务
* Invoker：服务调用接口，服务消费者实现该接口，定义需要消费、调用的接口；invoke发起对远程服务提供者，所提供的服务的请求、调用。
* Invocation：封装consumer进行远程服务调用的相关数据，如要调用的方法，进行调用的相关参数、参数类型和参数值等。
* InvokerListener：服务调用监听器，消费者实现该接口，在进行了对远程服务的调用进行回调；在服务提供者取消或回收了服务后，对消费者本地的消费代理进行销毁。
### ProxyFactory：代理工厂接口
* Dubbo支持两种类型的代理，分别是javassist和jdk，可以通过URL的proxy参数来指定，默认为javassist代理，如下：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181202160341994.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAwMTM1NzM=,size_16,color_FFFFFF,t_70)
* ProxyFactory定义：所有方法均有@Adaptive注解，可以通过URL的proxy参数动态决定使用哪种proxyFactory实现，默认为javassist。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181202162131232.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAwMTM1NzM=,size_16,color_FFFFFF,t_70)
* 每个具体的RPC协议实现均包含一个proxyFactory代理工厂属性，通过proxyFactory获取proxy实例，从而对invoker进行封装，通过代理进行服务调用。源码如下：
1. AbstractProxyProtocal：包含setProxyFactory方法，ExtensionLoader会根据@Adaptive规则，自动注入。
2. 在提供者export暴露服务时，调用proxy.getProxy获取proxy实例作为doExport参数；消费者refer调用远程服务时，底层调用doRefer(type, url)方法，该方法返回proxy实例，由该proxy实例进行服务调用。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181202163105892.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAwMTM1NzM=,size_16,color_FFFFFF,t_70)
3. 具体RPC实现类继承AbstractProxyProtocol
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181202163850748.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAwMTM1NzM=,size_16,color_FFFFFF,t_70)