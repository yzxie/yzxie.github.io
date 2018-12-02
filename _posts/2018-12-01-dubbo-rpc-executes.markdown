---
layout:     post
title:      "Dubbo源码分析：RPC协议实现-服务端并发控制与Semaphore信号量"
subtitle:   ""
date:       2018-12-01 17:12:00
author:     "XYZ"
header-img: "img/article-bg1.jpg"
tags:
    - Dubbo源码分析
---
## 概述
Dubbo支持在服务端通过在service或者method，通过executes参数设置每个方法，允许并发调用的最大线程数，即在任何时刻，只允许executes个线程同时调用该方法，超过的则抛异常返回，从而对提供者服务进行并发控制，保护资源。
## 用法
### 服务级别
限制 com.foo.BarService 的每个方法，服务器端并发执行（或占用线程池线程数）不能超过 10 个：
```
<dubbo:service interface="com.foo.BarService" executes="10" />
```
### 方法级别
限制 com.foo.BarService 的 sayHello 方法，服务器端并发执行（或占用线程池线程数）不能超过 10 个：
```
<dubbo:service interface="com.foo.BarService">
    <dubbo:method name="sayHello" executes="10" />
</dubbo:service>
```
## 源码实现
* Java Semaphore：信号量
   * 定义：Java信号量类似于一个计数器，用于限制在任何时刻，只允许给定个线程对某个共享资源的访问。
   * 用法：
1. 获取信号量：每个线程通过执行tryAcquire（非阻塞）或者acquire（阻塞，可中断）获取一个信号量或者说是通行证，同时将信号总量减一，当数量变为0时，则后面来的线程获取则返回false或者阻塞；
2. 释放信号量：线程对并发资源访问完毕之后，通过调用relase方法将信号总量加1，允许一个线程访问该共享资源；
* ExecuteLimitFilter：线程并发控制过滤器
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181202143206597.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAwMTM1NzM=,size_16,color_FFFFFF,t_70)
1. 服务端每个线程处理RPC请求，进行provider方法调用时，该线程执行ExecuteLimitFilter的invoke方法，定义Semaphore类型的executeLimit的局部变量，该局部变量的主要作用是用于引用与该provider方法绑定的RpcStatus，该RpcStatus由所有调用该provider方法的线程共享；
2. 调用count.getSemaphore(max)获取该provider方法（即共享资源）的信号量：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181202143233116.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAwMTM1NzM=,size_16,color_FFFFFF,t_70)
* 通过synchronized和double check来保证信号量只初始化一次或者当前调用的url指定的executes数量变化时，更新。注意executesLimit为RpcStatus的实例的field，而单个provider的方法所绑定的rpcStatus是又所有调用这个方法的线程共享的，故为了保证该executesLimit对其他线程的可见性，从而在其中一个线程初始化之后，其他线程在进入synchronized块，再次判断executeLimit是否为null时，感知其他线程已经初始化过了，需要将executesLimit设置为volatile：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181202143257971.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAwMTM1NzM=,size_16,color_FFFFFF,t_70)
* 调用Semaphore的tryAcquire获取一个信号量，如果获取到则进行往下执行进行provider的方法调用，否则抛异常返回。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181202151032182.png)
其中tryAcquire为非阻塞的，直接返回true或者false，当前线程进行往下执行，需要在程序中控制是否获取成功的处理，并且是unfair的。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181202143331862.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAwMTM1NzM=,size_16,color_FFFFFF,t_70)