---
layout:     post
title:      "Dubbo源码分析：RPC协议实现-接口定义"
subtitle:   "Dubbo RPC Interface"
date:       2018-11-30 15:12:00
author:     "XYZ"
header-img: "img/article-bg1.jpg"
tags:
    - Dubbo源码分析
---
## 定义
提供者Provider：服务提供者如何将自己能提供的服务暴露出去，使得服务消费者可以远程调用。
消费者Consumer：服务消费者对自己需要的服务，如何感知该服务在哪里，如何通过远程调用的方式，调用服务提供者提供的服务。
核心接口如下：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181201233410370.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAwMTM1NzM=,size_16,color_FFFFFF,t_70)
## 核心接口设计
1. Protocol：RPC协议接口
* export方法： 提供者需要实现export方法，定义如何将自己提供的服务保留出去，同时如何使得消费者可以调用。
* refer方法：消费者实现refer方法，提供自己需要调用的远程服务的class类型，provider暴露服务的url地址，获得对远程服务进行实际服务调用的代理，即该代理对服务消费者而言，就是服务提供者，代理在服务提供者和服务消费者之间作为中间人，搭建了一个服务调用的桥梁。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181201172813269.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAwMTM1NzM=,size_16,color_FFFFFF,t_70)

2. Provider相关：服务提供者提供服务
* Exporter：服务暴露接口，该接口表示提供者将服务暴露或者说注册到了一个注册中心。该接口实例获取该次注册，服务提供者的调用器，同时可以实现服务的取消或者回收。
* ExporterListener：服务暴露监听器，在对服务进行暴露后，或者取消已经暴露的服务，进行回调处理。
* Invoker：服务调用器，服务提供者定义该接口的实现，即接收消费者的远程服务调用的请求，invoke调用本地方法实现，返回结果给消费者。
* Invocation：服务提供者实现该接口获取消费者调用的相关数据，如方法、参数等，实际发起对本地服务调用的接口。

3. Consumer相关：服务消费者消费服务
* Invoker：服务调用器，服务消费者实现该接口，定义需要消费、调用的接口，即提供者提供的服务；invoke发起对远程服务提供者，所提供的服务的请求、调用。
* Invocation：底层实际发起对远程服务调用的接口，消费者实现该接口，提供进行远程服务调用的相关数据，如要调用的方法，进行调用的相关参数和参数值等。
* InvokerListener：服务调用监听器，消费者实现该接口，在进行了对远程服务的调用进行回调；在服务提供者取消或回收了服务后，对消费者本地的消费代理进行销毁。