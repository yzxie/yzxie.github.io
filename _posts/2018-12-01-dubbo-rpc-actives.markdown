---
layout:     post
title:      "Dubbo源码分析：RPC协议实现-客户端并发调用控制"
subtitle:   ""
date:       2018-12-01 15:12:00
author:     "XYZ"
header-img: "img/article-bg1.jpg"
tags:
    - Dubbo源码分析
---
## 概述
Dubbo支持在服务或者方法粒度，通过actives参数，控制客户端对提供者服务的所有方法或者某个方法进行并发访问控制，即在同一时刻，客户端只允许active个请求并发调用服务的某个方法，超过的请求需要等待，如果在timeout时间内还是无法执行调用，则异常退出。
## 用法
#### 服务级别
限制 com.foo.BarService 的每个方法，每客户端并发执行（或占用连接的请求数）不能超过 10 个：
```
<dubbo:service interface="com.foo.BarService" actives="10" />
或
<dubbo:reference interface="com.foo.BarService" actives="10" />
```
#### 方法级别
限制 com.foo.BarService 的 sayHello 方法，每客户端并发执行（或占用连接的请求数）不能超过 10 个：
```
<dubbo:service interface="com.foo.BarService">
    <dubbo:method name="sayHello" actives="10" />
</dubbo:service>
或
<dubbo:reference interface="com.foo.BarService">
    <dubbo:method name="sayHello" actives="10" />
</dubbo:service>
```
如果dubbo:service和 dubbo:reference都配了actives，dubbo:reference优先。
## 源码实现
1. 实现类：在rpc包下的ActiveLimitFilter，即通过过滤器的方式对请求进行过滤，当未达到actives个并发请求时，则将rpc请求直接传给下个过滤器，否则等待直到可以执行或者超时异常。
2. 核心实现：通过synchronized来进行线程同步，实现等待；方法当前并发连接数，通过线程安全静态统计类RpcStatus来实现的。
* ActiveLimitFilter类
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181202114149918.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAwMTM1NzM=,size_16,color_FFFFFF,t_70)
从上到下分析：
（1）根据请求的url和method，获取该方法的状态统计类实例count，该实例由所有请求线程共享；
（2）对count进行synchronized同步，当当前并发请求大于配置的max时，则通过wait执行等待，同时设置超时时间。
（3）wait：要么是这个请求之前的其他请求调用成功，调用notify唤醒到该请求处理线程，要么wait超时。
（4）elapsed：不管是被其他线程notify唤醒还是wait超时唤醒，都需要计算一下从开始到现在过去了多长时间，值为elapsed，如果超过或者刚好是达到timeout，则不管是超时唤醒，还是刚好超时时刻被其他线程notify缓存，则都抛出超时异常。这样设计主要是只有不是超时唤醒，是在超时之前，被其他线程notify唤醒才进行执行。
（5）执行方法调用：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181202114212366.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAwMTM1NzM=,size_16,color_FFFFFF,t_70)
   * RpcStatus.beginCcount：累加当前并发调用数
   * 执行方法调用
   * 不过是调用成功还是异常，则均需要递减当前并发调用数
   * finally：调用notify通知其他等待线程可以进行执行

* RpcStatus：Rpc调用统计类，线程安全
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181202114245297.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAwMTM1NzM=,size_16,color_FFFFFF,t_70)
（1）线程安全：主要是使用java.concorrent包的线程安全类保证线程安全，除METHOD_STATISTICS，其他属性为对应一个服务的一个方法的调用相关统计，如当前并发活跃请求数active，总请求数total等。
（2）METHOD_STATISTICS：方法统计缓存。key为uri，即服务providerURL，value为ConcurrentMap<String, RpcStatus>，其中这个key为方法名methodName，value为该方法的统计RpcStatus实例。
（3）getStatus：获取方法的统计实例。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181202114308348.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAwMTM1NzM=,size_16,color_FFFFFF,t_70)
（4）AtomicInteger：原子类保证线程安全
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181202114335550.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAwMTM1NzM=,size_16,color_FFFFFF,t_70)

