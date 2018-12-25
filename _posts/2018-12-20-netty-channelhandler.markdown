---
layout:     post
title:      "Netty源码分析-数据处理器ChannelHandler的设计"
subtitle:   "Netty ChannelHandler"
date:       2018-12-20 07:12:00
author:     "XYZ"
header-img: "img/article-bg1.jpg"
tags:
    - Netty源码分析
---
### 概述
ChannelHandler用于处理Channel产生的IO事件，如服务端和客户端之间的数据读写read、write，客户端发起连接请求connect，服务端绑定监听端口bind等，用户可以自定义ChannelHandler接口的实现来处理IO事件。以及可以将该IO事件往下传给ChannelPipeline中的下一个ChannelHandler。
### ChannelHandler的接口设计
* 在整个IO事件处理体系的接口设计层面，由于Channel产生的IO事件是在ChannelPipeline中进行传播的，ChannelHandlerContext接口负责将ChannelHandler添加到ChannelPipeline，只有这样ChannelHandler才能拦截到Channel产生IO的事件并进行处理；同时ChannelHandlerContext也负责ChannelHandler向ChannelPipeline中相邻的其他ChannelHandler的传播IO事件。ChannelHandler接口自身并不提供如何对IO事件进行处理的方法。
* 故只需要定义添加和删除ChannelHandler到ChannelHandlerContext的方法，而不需要定义对IO事件进行具体处理的方法，这些处理方法由子类定义和实现。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181223100902739.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAwMTM1NzM=,size_16,color_FFFFFF,t_70)
### ChannelHandler的子类型
ChannelHandler的子类型主要是根据IO事件是数据读入还是写出来区分：
1. ChannelInboundHandler： 处理数据读入IO事件，适配器类为ChannelInboundHandlerAdapter；
2. ChannelOutboundHandler： 处理数据写出IO事件，适配器类为ChannelOutboundHandlerAdapter；
3. ChannelDuplexHandler：对数据读入和写出IO事件均进行处理。
子类型的具体设计，在之后的文章分析。
### ChannelHandlerContext：ChannelHandler的包装器
 ChannelHandler只对定义对Channel产生的IO事件的数据进行处理逻辑，而通过ChannelHandlerContext的包装，增强的功能包括：
 1. 将ChannelHandler接入ChannelPipeline中，使得ChannelHandler可以拦截捕获Channel的IO事件；
 2. 将ChannelPipeline中的多个ChannelHandler组成一条处理链，实现Channel的IO事件在这些ChannelHandler之间传播；
 3. ChannelHandler可以通过ChannelHandlerContext对ChannelPipeline动态修改，如添加或删除ChannelHandler
 4. ChannelHandlerContext通过与AttributeKey的结合使用，使得一个ChannelHandler实例被多个Channel或者是多个ChannelPipeline所共享。通过ChannelHandlerContext来存储，针对每个Channel，在这个共享的ChannelHandler中的特定状态数据，详见下面的分析。
* 关于ChannelHandlerContext的详细分析可以看：[Netty源码分析-ChannelHandler的包装器-ChannelHandlerConext](https://blog.csdn.net/u010013573/article/details/85212804)
 ### ChannelHandler的状态数据管理
 ChannelPipeline中每个ChannelHandler经常需要有自己的一些状态数据，先定义一个状态数据类：
```
 public interface Message {
     // your methods here
 }
```
ChannelHandler状态管理的实现方式包括：
 1. 最简单的是使用成员变量，为了保证数据安全性，需要保证ChannelPipeline中每个ChannelHandler都需要是不同的实例，即都需要通过new创建一个实例。如下：由于每个连接Channel都有一个特定的数据loggedIn，用于当前连接的Channel是否登录了，为了避免数据竞争，即不同Channel的loggedIn混用导致未登录的用户也可以请求数据，需要为每个Channel创建一个独立的ChannelHandler实例。
```
 public class DataServerHandler extends SimpleChannelInboundHandler<Message> {
     // 成员变量记录当前连接的Channel是否登录了
      private boolean loggedIn;
 
      @Override
      public void channelRead0(ChannelHandlerContext ctx, Message message) {
          if (message instanceof LoginMessage) {
              authenticate((LoginMessage) message);
              loggedIn = true;
          // 只有连接成功的Channel才能发起获取数据的请求   
          } else (message instanceof GetDataMessage) {
              if (loggedIn) {
                  ctx.writeAndFlush(fetchSecret((GetDataMessage) message));
              } else {
                  fail();
              }
          }
      }
  }
```
2. 通过ChannelInitializer的initChannel方法，在创建新的Channel时，为该Channel的pipeline添加，即new一个独立的DataServerHandler实例。
```
 public class DataServerInitializer extends ChannelInitializer<Channel> {
      @Override
      public void initChannel(Channel channel) {
         // 每个Channel都new一个DataServerHandler
          channel.pipeline().addLast("handler", new DataServerHandler());
      }
 }
```
3. 使用AttributeKey和ChannelHandlerContext实现：通常是推荐使用成员变量来存储ChannelHandler的状态数据，但是这种方式会创建大量的ChannelHandler实例。所以如果需要多个Channel共享ChannelHandler实例，则可以使用ChannelHandlerContext提供的AttributeKey，结合ChannelHandlerContext的attr方法，从而将状态数据交给ChannelHandlerContext来存储和取出，Netty框架中每个ChannelHandlerContext实例都是不同的实例。
```
// 共享注解
 @Sharable
 public class DataServerHandler extends SimpleChannelInboundHandler<Message> {
     // 状态数据的key和value的类型
     private final AttributeKey<Boolean> auth = AttributeKey.valueOf("auth");
     
     @Override
     public void channelRead(ChannelHandlerContext ctx, Message message) {
         // 通过ChannelHandlerContext的attr方法取出
         Attribute<Boolean> attr = ctx.attr(auth);
         if (message instanceof LoginMessage) {
             authenticate((LoginMessage) o);
             // 通过Attribute的set方法设值
             attr.set(true);
         } else (message instanceof GetDataMessage) {
             if (Boolean.TRUE.equals(attr.get())) {
                 ctx.writeAndFlush(fetchSecret((GetDataMessage) o));
             } else {
                 fail();
             }
         }
     }
 }
```
在ChannelInitializer调用：
```
 // 状态数据交个ChannelHandlerContext管理了，所以可以将同一个ChannelHandler添加到多个ChannelPipeline中了，即多个Channel共享同一个ChannelHandler实例
 public class DataServerInitializer extends ChannelInitializer<Channel> {
 
      // 静态变量，则由DataServerInitializer初始化的Channel都共享同一个DataServerHandler实例
      private static final DataServerHandler SHARED = new DataServerHandler();
 
      @Override
      public void initChannel(Channel channel) {
          // 不再使用new创建
          channel.pipeline().addLast("handler", SHARED);
      }
 }
```
### @Sharable注解在Netty中的作用
@Sharable是一种声明作用，说明可以看：[Java @Sharable](http://www.javaconcurrencyinpractice.com/annotations/doc/)
在Netty中如果自己使用@Sharable修饰一个ChannelHandler实现，则需保证能被多个Channel共享；在Netty框架自身提供的有@Sharable注解修饰的ChannelHandler则可以只创建new一个实例，在多个Channel中重用这个实例；否则记得每个Channel都要new一个新的ChannelHandler实例再添加到ChannelPipeline中。
