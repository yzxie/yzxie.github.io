---
layout:     post
title:      "Dubbo源码分析：序列化方式与调用"
subtitle:   "Dubbo serialization"
date:       2018-12-06 22:12:00
author:     "XYZ"
header-img: "img/article-bg1.jpg"
tags:
    - Dubbo源码分析
---
## 概述
* 序列化模块主要为dubbo协议提供服务提供者和服务消费者之间的数据序列化功能。dubbo是一种适合于高并发、小数据量的互联网应用场景的框架，而序列化对于远程调用的响应速度，吞吐量，网络带宽消耗也其中至关重要的作用，是提高分布式系统性能的最关键因素之一。
* 在源码实现中，在远程传输模块remoting，调用Transporter传输数据之前，通过Codec架构对数据进行序列化，在Codec中调用dubbo-serialization模块的序列化类进行序列化。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181211214804815.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAwMTM1NzM=,size_16,color_FFFFFF,t_70)

## 使用方式

```
 <dubbo:protocol name="dubbo" serialization="kryo"/>
```

## 序列化的类型
dubbo框架原生支持四种序列化类型，分别为：dubbo序列化，hessian2序列化，json序列化，jdk序列化；性能依次下降。默认为hessian2序列化。dubbo序列化为dubbo框架自身实现的一种Java序列化方案，但是不够成熟，不建议在生产环境使用。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181211220739524.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAwMTM1NzM=,size_16,color_FFFFFF,t_70)
在成为Apache孵化项目之后，对序列化方式进行了优化，支持的类型，分别为：fastjson，fst，hessian2，jdk和kryo。默认序列化方案还是hessian2。
其中fst为完全兼容JDK序列化协议的序列化框架，序列化速度是JDK的4到10倍，大小是JDK的1/3左右。kryo序列化速度也比JDK的要快，并且大小是JDK的1/10左右。fst和kryo性能普通好于其他序列化方案，生产环境比较推荐使用。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181211220809675.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAwMTM1NzM=,size_16,color_FFFFFF,t_70)