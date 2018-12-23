---
layout:     post
title:      "Netty源码分析-ChannelHandler的包装器-ChannelHandlerConext"
subtitle:   "Netty ChannelPipeline"
date:       2018-12-19 07:12:00
author:     "XYZ"
header-img: "img/article-bg1.jpg"
tags:
    - Netty源码分析
---
### ChannelHandler与ChannelHandlerContext的关系
#### ChannelHandler
定义对Channel中IO数据的处理逻辑，主要是面向业务逻辑的处理，只需按业务需要的数据处理顺序，通过调用ChannelPipeline的addLast等方法，将多个ChannelHandler添加到ChannelPipeline中即可，无需关心多个ChannelHandler在ChannelPipeline中如何联系起来的，这个工作由ChannelHandlerContext完成。
#### ChannelHandlerContext
实现Netty内部各组件，如ChannelHandler、ChannelPipeline，合作完成数据处理的基础，核心功能是实现ChannelHandler和ChannelPipeline以及其他ChannelHandler之间通信、交互。在ChannelPipeline中，每个ChannelHandler对象实例，对应一个ChannelHandlerContext，由该ChannelHandlerContext代表ChannelHandler或者说ChannelHandler委托ChannelHandlerContext，在ChannelPipeline中对Channel的数据进行处理（实际底层仍旧是使用ChannelHandler进行数据处理），以及与其他ChannelHandler的交互通信。具体表现为：
（1）ChannelHandlers之间的交互：调用内部关联的ChannelHandler实例处理数据，或者将数据交给ChannelPipeline中的下一个ChannelHandler处理；
（2）在运行时，可以按需要动态对ChannelPipeline进行修改，如添加、删除ChannelHandler。
故在ChannelHandlerContext需要包含ChannelHandler和ChannelPipeline的引用：
![在这里插入图片描述](https://img-blog.csdnimg.cn/201812221734407.png)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181222173506705.png)
同时维护当前ChannelHandler在ChannelPipeline中相邻的，即pre（上一个）和next（下一个）的ChannelHandler的引用，而pre和next的赋值维护是由ChannelPipeline完成的（在addLast等方法维护）：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181222173813239.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAwMTM1NzM=,size_16,color_FFFFFF,t_70)
#### ChannelHandler的共享
由于ChannelHandler通常在每个方法中使用ChannelHandlerContext和需要处理的数据msg作为参数，即使用局部方法保证线程安全，如图：
![在这里插入图片描述](https://img-blog.csdnimg.cn/2018122220444159.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAwMTM1NzM=,size_16,color_FFFFFF,t_70)
同时添加到ChannelPipeline中就会创建一个独立ChannelHandlerContext实例与该ChannelHandler关联，故同一个ChannelHandler实例可以添加到多个ChannelPipeline或者在一个ChannelPipeline中添加多次，而涉及到线程安全的有状态数据则使用ChannelHandlerContext的attr()方法结合AttributeKey来实现ChannelHandler在多个ChannelPipeline中互不影响。具体看下面的详细分析。
### 与ChannelPipeline中其他ChannelHandler的交互
* ChannelHandlerContext继承了ChannelInboundInvoker和ChannelOutboundInvoker，将数据交给下一个ChannelHandler时，对于读入的数据，是通过实现ChannelInboundInvoker的fireChannelXXX系列方法来触发的；对于写出的数据，是通过实现ChannelOutboundInvoker的bind，write等方法来处理的。以下分析以AbstractChannelHandlerContext类为基础。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181222180656835.png)
* 读入数据：以read为例，其他还有像channel active，registered等，通过调用ChannelHandlerContext的fireChannelRead来调用下一个的ChannelInboundHandlerContext，即next，具体实现在invokeChannelRead(Object msg)，如果next添加过ChannelInboundHandler（方法invokeHandler()判断），则强制转换为ChannelInboundHandler，并执行channelRead，channelRead的逻辑是由用户自定义的，否则调用fireChannelRead继续往下传给下一个ChannelHandler处理。其中在channelRead中，用户可以处理完消息释放掉（ReferenceCountUtil.release(msg)或ChannelInboundHandler继承SimpleChannelInboundHandler）；或者调用ctx.fireChannelRead往下传输，ctx为channelRead方法，类型为ChannelInboundHandlerContext的入参，即当前ChannelInboundHandler所关联的ChannelInboundHandlerContext。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181222180414413.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAwMTM1NzM=,size_16,color_FFFFFF,t_70)
* 写出数据：以write为例，其他还有像bind, connect等
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181222181734306.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAwMTM1NzM=,size_16,color_FFFFFF,t_70)
上图第三个红框的实现如下图：获取下一个outbound，然后调用invokeWriteAndFlush或invokeWrite，这两个方法在上图实现的，如invokeWrite：如果channelHandlerContext添加过channelHandler，则调用invokeWrite0方法，将channelHandler强制转换为ChannelOutboundHandler，然后执行write，这个write的逻辑是用户自定义了的（在write中，用户可以定义直接写到底层socket，如调用ctx.channel().write(msg)，或者调用，如ctx.write(msg)，传给下一个ChannelOutboundHandler继续处理，这也是**调用Channel的write和ChannelHandlerContext的write的区别**）；
否则传给下一个ChannelOutboundHandler处理。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181222181438880.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAwMTM1NzM=,size_16,color_FFFFFF,t_70)
### ChannelPipeline中共享同一个ChannelHandler实例
* 共享包含两层含义：
1. 被添加到同一个ChannelPipeline中多次；
2. 添加到多个ChannelPipeline
* 伪代码演示
```
FactorialHandler fh = new FactorialHandler();
// 添加到一个ChannelPipeline多次
ChannelPipeline p1 = Channels.pipeline();
p1.addLast("f1", fh);
p1.addLast("f2", fh);

// 添加到多个ChannelPipeline，即p1, p2
ChannelPipeline p2 = Channels.pipeline();
p2.addLast("f3", fh);
p2.addLast("f4", fh);
```
* 状态数据的管理
以同一个ChannelHandler添加到多个ChannelPipeline为例分析：
   * 由于多个ChannelPipeline或者说多个Channel，共享同一个ChannelHandler实例，由该ChannelHandler处理这些不同Channel中流通的数据，故如果在处理过程中需要记录与每个Channel特定的数据，多个Channel互相独立，即维护有状态的数据，则需要使用AttributeKey指定状态数据的key和value的类型，使用ChannelHandlerContext的attr方法，将该key对应的数据存到ChannelHandlerContext，因为ChannelHandlerContext是每次（添加到同一个ChannelPipeline多次或者添加到多个ChannelPipeline）添加到ChannelPipeline中都会创建一个独立的实例的。
   * 示例：该ChannelHandler对应上面FactorialHandler
```
 public class FactorialHandler extends ChannelInboundHandlerAdapter {
   // 有状态的数据counter
    private final AttributeKey<Integer> counter = AttributeKey.valueOf("counter");
 
    // This handler will receive a sequence of increasing integers starting
    // from 1.
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) {
     // 获取当前ChannelHandlerContext特定的counter
      Integer a = ctx.attr(counter).get();
 
      if (a == null) {
        a = 1;
      }
 
      attr.set(a * (Integer) msg);
    }
  }
```