---
title: Netty-6_ChannelHandler(1)
tags: 
  - Netty
date: 2023-03-29 09:00:00
categories:
  - Netty
---

上一篇文章我们深入学习了Netty逻辑架构中的核心组件EventLoop和EventLoopGroup，掌握了Netty的线程模型，并且介绍了Netty4线程模型中的无锁串行化设计。

今天，我们继续学习Netty逻辑架构中的另一个核心组件ChannelHandler和ChannelPipeline。

如果说线程模型是Netty的 “核心内功”，那么ChannelHandler就是Netty最著名的 “武功招式”，是我们日常使用Netty时接触最多的组件。

引用《Netty in action》中的一句话

> From the appliaction developer's standpoint, the primary component of Netty is the ChannelHandler.

所以，阿丸尽可能通过 图 和 代码demo，来让大家获得最直观的使用体验。

本文预计阅读时间约 10分钟，将重点围绕以下几个问题展开：

- 什么是ChannelHandler和ChannelPipeline？
- ChannelHandler的事件传播机制
- ChannelHandler的异常处理机制
- ChannelHandler的最佳实践

## 1. 什么是ChannelHandler和ChannelPipeline？

ChannelHandler是一个包含所有应用处理逻辑的容器载体，用来对Netty的输入输出数据进行加工处理。

> 比如数据格式转换、异常处理等

ChannelPipeline 则是 ChannelHandler 的容器载体，负责以链式的形式调度各个注册的ChannelHandler。

我们回顾下之前介绍过的Netty逻辑架构，观察下ChannelPipeline和ChannelHandler的位置。

![Netty基础招式——ChannelHandler的最佳实践](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202303301056419.png)

 

再从局部放大，可以更加明确地看到ChannelPipeline和ChannelHandler的作用。

![Netty基础招式——ChannelHandler的最佳实践](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202303301056478.png)

 

如上图所示，当EventLoop中监听到事件后，会对I/O事件进行处理。而这个处理，就是交给ChannelPipeline进行，更严格地说，是交给ChannelPipeline中的各个ChannelHandler按照一定的顺序进行处理。

根据数据的流向，Netty把ChannelHandler分为2类，InboundHandler和OutboundHandler。

![Netty基础招式——ChannelHandler的最佳实践](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202303301056524.png)

 

如上图所示，Netty接收到数据后，经过若干 InboundHandler 处理后接收成功。如果要输出数据，就需要经过若干个 OutboundHandler 处理完成后发送。

比如，我们经常需要对接收到的数据进行解码，就是在某一个专门decode的InboundHandler中处理的。如果要发送数据，往往需要编码，就是在某一个专门encode的OutBoundHandler中处理的。

值得一提的是，虽然我们在使用Netty时，直接打交道的是ChannelPipeline和ChannelHandler，但是，它们之间有一座“隐形”的桥梁，名字叫做ChannelHandlerContext。

顾名思义，ChannelHanderContext就是ChannelHandler的上下文，每个 ChannelHandler 都对应一个 ChannelHandlerContext。

每一个 ChannelPipeline 都包含多个 ChannelHandlerContext，所有 ChannelHandlerContext 之间组成了双向链表。如下图所示。

![Netty基础招式——ChannelHandler的最佳实践](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202303301056579.png)

 

其中，有两个特殊的ChannelHandlerContext，分别是HeadContext和TailContext，表示双向链表的头尾节点。

![Netty基础招式——ChannelHandler的最佳实践](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202303301056752.png)

 

从类图上可以看到，HeadContext同时实现了ChannelInboundHandler和ChannelOutboundHandler。因此，HeadContext在读取数据时作为头节点，向后传递InBound事件，同时，在写数据时作为尾节点，处理最后的OutBound事件。

TailContext只实现了ChannelInboundHandler。它在InBound事件传递的末尾，负责处理一些资源释放的工作。在OutBound事件传递的第一个节点，不做任何处理，仅仅传递OutBound事件给prev节点。

而我们平时自定义的ChannelHandler，就是插在这两个头尾节点之间的。

至此，我们对ChannelHandler和ChannelPipeline有了基本的认识。具体到实践上，我们该如何正确地使用ChannelHandler呢？

对ChannelHandler的使用，必须先了解ChannelHandler的事件传播机制和异常处理机制。

## 2. ChannelHandler的事件传播机制

前面我们提到了Netty中的两种事件类型，Inbound事件和Outbound事件，分别对应InboundHandler和OutbountHandler进行处理。

当我们使用Netty进行开发的时候，必须了解Inbound事件和Outbound事件在ChannelPipeline中如何进行“事件传播”，注册InboundHandler和OutboundHandler的顺序有什么影响。

话不多说，我们先来一个demo直观地感受一下。

自定义一个ChannelInboundHandler

![Netty基础招式——ChannelHandler的最佳实践](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202303301056924.png)

 

自定义一个ChannelOutboundHandler

![Netty基础招式——ChannelHandler的最佳实践](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202303301056106.png)

 

简单组装一下EchoPipelineServer，特别注意一下 6个handler 的注册顺序。

![Netty基础招式——ChannelHandler的最佳实践](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202303301056332.png)

 

然后我们通过命令行简单访问一下这个Netty Server

```css
curl localhost:8081
```

可以看到控制台的如下输出

![Netty基础招式——ChannelHandler的最佳实践](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202303301056543.png)

 

这样就清楚了事件传播顺序：
\- 对于Inbound事件，InboundHandler的处理顺序是和注册顺序一致
\- 对于Outbound事件，OutboundHandler的处理顺序和注册顺序相反

结合上一节说的HeadContext和TailContext，我们画个图来更直观地看一下这个ChannelPipeline中的handler构建顺序是怎样的。

![Netty基础招式——ChannelHandler的最佳实践](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202303301056599.png)

 

在上面的ChannelInitializer中，我们按需添加了3个InboundHandler和3个OutboundHandler。所以，在头节点HeadContext和TailContext之间，有序构成了双向链表。

而InboundHandler3中，通过调用 ctx.channel.writeAndFlush( msg ) 方法，将消息从TailContext开始，依据OutboundHandler的路径向HeadContext方向传播出去。具体可以看下DefaultChannelPipeline类中的实现

![Netty基础招式——ChannelHandler的最佳实践](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202303301056499.png)

 

> 虽然这里是双向链表，但是无论是Inbound事件还是Outbound事件，在按序访问链表节点时，会根据事件类型进行过滤。

## 3. ChannelHandler的异常传播机制

我们已经了解了ChannelPipeline的链式传递规则，如果双向链表中任意一个handler抛出了异常，那么应该怎么处理呢？

### 3.1 InboundHandler的异常处理

我们修改下示例中的TestInboudHandler进行模拟。

- channelRead方法中抛出异常
- 重写exceptionCaught方法，打印当前节点捕获异常情况

![Netty基础招式——ChannelHandler的最佳实践](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202303301056230.png)

 

得到输出如下

![Netty基础招式——ChannelHandler的最佳实践](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202303301056532.png)

 

可以看到，虽然在InboundHander1中抛出了异常，但是仍然会被3个InboundHandler都捕获一次，并按序向tail节点方向传递，然后抛出异常。

我们也看到了，Netty给出了会警告，在最后的节点没有进行异常处理。

```armasm
An exceptionCaught() event was fired, and it reached at the tail of the pipeline. 
It usually means the last handler in the pipeline did not handle the exception.
```

### 3.2 OutboundHandler的异常处理

OutboundHandler也是这么操作吗？
我们来做个实验。

- 在write操作中抛出异常
- 重写下exceptionCaught方法（这个方法在OutboundHandler中被标记为废弃）

重写组装下channelPipeline，第二个OutboundHandler中抛出异常

![Netty基础招式——ChannelHandler的最佳实践](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202303301056792.png)

 

结果得到的输出如下

![Netty基础招式——ChannelHandler的最佳实践](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202303301056049.png)

 

咦？异常被吃掉了！！
不仅没有走进exceptionCaught方法，也没有其他异常抛出。
只是对后续handler的write方法不再执行，而flush方法还是都执行了一遍。

我们从源码找找原因吧。跟一下断点，马上就找到了原因：

![Netty基础招式——ChannelHandler的最佳实践](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202303301056328.png)

 

在
AbstractChannelHandlerContext中，对OutboundHandler的write方法做了异常捕获，然后对ChannelPromise进行了通知。
后续源码就不展开了，有兴趣的同学自己打断点跟一下，比较清楚。

那么问题来了，怎么在OutboundHandler中捕获异常呢？很明显就是直接添加ChannelPromise的回调。
上代码：

![Netty基础招式——ChannelHandler的最佳实践](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202303301056667.png)

 

在前面提到的ExceptionHandler中，复写write方法，然后注册一个ChannelPromise的Listener就行了。
当然，这个ExceptionHandler同样要注册到ChannelPipeline。

> 千万注意！！这里ExceptionHandler同样是添加到ChannelPipeline的tail方向的最后，而不是添加在head方向。
> 无论是inboundHandler或者是outboundHandler的异常，都是按序向tail方向传递的。

异常就这样抓到了。

![Netty基础招式——ChannelHandler的最佳实践](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202303301056943.png)

 

## 4. ChannelHandler的最佳实践

其实前面已经对ChannelHandler的常用机制做了介绍，这里简单再介绍下两个最佳实践。

### 4.1 不在ChannelHandler中做耗时处理

这一点其实在前一篇《 深入Netty逻辑架构，从Reactor线程模型开始》已经提到过，这里作为自定义ChannelHandler的最佳实践再强调一下，不在ChannelHandler中做耗时处理。

这里包括两点。

一是不在I/O线程中直接处理耗时操作。

二是也不把耗时操作放进EventLoop的任务队列中。

由于Netty4的无锁串行化设计，一旦任何耗时操作阻塞了某个EventLoop，那么这个EventLoop上的各个channel都会被阻塞。更详细内容可以参考上一篇《 深入Netty逻辑架构，从Reactor线程模型开始》。

所以，我们对于耗时操作，我们要放在自己的业务线程池中进行处理，如果需要发送response，需要提交任务到EventLoop的任务队列中执行。

给个简单的demo。

![Netty基础招式——ChannelHandler的最佳实践](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202303301056115.png)

 

## 4.2 统一的异常处理

在本文的第三节中，讲解了ChannelHandler的异常传播机制。

对于InboundHandler来说，如果你有跟handler特定相关的异常，可以直接在handler里进行exceptionCaught。如果是一些通用的异常，可以自定义ExceptionHandler注册到ChannelPipeline的末尾进行统一拦截。

对于OutboudHandler来说，就是通过自定义ExceptionHandler，重写对应方法，并注册ChannelPromise的Listener。同样的，ExceptionHandler注册到ChannelPipeline的末尾进行统一拦截。

所以，总结下如何添加一个“统一”的异常拦截器呢？

- 自定义ExceptionHandler继承ChannelDuplexHandler，并注册到 tail节点前（ChannelPipeline的最后一个节点）
- 对于Inbound事件，我们需要在exceptionCaught()进行处理
- 对于Outbound事件，我们需要不同的ChannelFutureListener

异常拦截器的注册位置应该在tail方向的最后一个Handler。

![Netty基础招式——ChannelHandler的最佳实践](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202303301056456.png)

 

> 注意，统一异常处理除了更优雅处理通用异常外，也是排查故障的好帮手。比如有时候对于编解码异常，可以在统一处理异常处捕获，快速定位问题。

## 5.小结

来简单回顾下吧。

本文介绍了什么是ChannelHandler和ChannelPipeline。能厘清InboundChannelHandler、OutboundChannelHandler、ChannelHandlerContext是什么吗？

然后对ChannelHandler的事件传播机制、异常处理机制做了详细介绍。

最后说明了日常开发中ChannelHandler的最佳实践。

希望对大家有所帮助。