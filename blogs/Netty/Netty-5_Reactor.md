---
title: Netty-5_Reactor线程模型
tags: 
  - Netty
date: 2023-03-29 09:00:00
categories:
  - Netty
---

上一篇文章我们从一个Netty的使用Demo，了解了用Netty构建一个Server服务端应用的基本方式。
并且从这个Demo出发，简述了Netty的逻辑架构，并对Channel、ChannelHandler、ChannelPipeline、EventLoop、EventLoopGroup等概念有了初步的认识。

回顾一下逻辑架构图。

 

![深入Netty逻辑架构，从Reactor线程模型开始](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202303301016885.png)

 

 

今天主要是深入学习下逻辑架构中的EventLoop 和 EventLoopGroup，掌握Netty的线程模型，这是Netty最精髓的知识点之一。 

本文预计阅读时间约 15分钟，将重点围绕以下几个问题展开：

- 什么是Reactor线程模型？

- EventLoopGroup、EventLoop 怎么实现Reactor线程模型？

- 深入Netty的线程模型优化

- - Netty3和Netty4的线程模型变化
  - 什么是Netty4线程模型的无锁串行化

- 从线程模型看最佳实践

## 1.什么是Reactor线程模型？

先来回顾下我们在Netty系列的第2篇介绍的I/O线程模型，包括BIO、NIO、I/O多路复用、信号驱动IO、AIO。IO多路复用在Java中有专门的NIO包封装了相关的方法。

前面的文章也说过，使用Netty而不是直接使用Java NIO包，就是因为Netty帮我们封装了许多对NIO包的使用细节，做了许多优化。

其中非常著名的，就是Netty的「Reactor线程模型」。

> 前置知识如果还不太清楚，可以回头看看前面几篇文章：
>
> 《没搞清楚网络I/O模型？那怎么入门Netty》
> 《从网络I/O模型到Netty，先深入了解下I/O多路复用》
> 《从I/O多路复用到Netty，还要跨过Java NIO包》

Reactor模式 是一种「事件驱动」模式。

「Reactor线程模型」就是通过 单个线程 使用Java NIO包中的Selector的select()方法，进行监听。当获取到事件（如accept、read等）后，就会分配（dispatch)事件进行相应的事件处理（handle）。

如果要给 Reactor线程模型 下一个更明确的定义，应该是：

```undefined
Reactor线程模式 = Reactor（I/O多路复用）+ 线程池
```

其中Reactor负责监听和分配事件，线程池负责处理事件。

然后，根据Reactor的数量和线程池的数量，又可以将Reactor分为三种模型

- 单Reactor单线程模型 (固定大小为1的线程池)
- 单Reactor多线程模型
- 多Reactor多线程模型 (一般是主从2个Reactor)

### 1.1 单Reactor单线程模型

![深入Netty逻辑架构，从Reactor线程模型开始](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202303301016960.png)

 

Reactor内部通过 selector 监听连接事件，收到事件后通过dispatch进行分发。

- 如果是连接建立的事件，通过accept接受连接，并创建一个Handler来处理连接后续的各种事件。
- 如果是读写事件，直接调用连接对应的Handler来处理，Handler完成 read => (decode => compute => encode) => send 的全部流程

这个过程中，无论是事件监听、事件分发、还是事件处理，都始终只有 一个线程 执行所有的事情。

> 缺点：
> 在请求过多时，会无法支撑。因为只有一个线程，无法发挥多核CPU性能。
> 而且一旦某个Handler发生阻塞，服务端就完全无法处理其他连接事件。

### 1.2 单Reactor多线程模型

为了提高性能，我们可以把复杂的事件处理handler交给线程池，那就可以演进为 「单Reactor多线程模型」 。

 

![深入Netty逻辑架构，从Reactor线程模型开始](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202303301016065.png)

 

 

这种模型和第一种模型的主要区别是把业务处理从之前的单一线程脱离出来，换成线程池处理。

1）Reactor线程

通过select监听客户请求，如果是连接建立的事件，通过accept接受连接，并创建一个Handler来处理连接后续的读写事件。这里的Handler只负责响应事件、read和write事件，会将具体的业务处理交由Worker线程池处理。

只处理连接事件、读写事件。

2）Worker线程池

处理所有业务事件，包括(decode => compute => encode) 过程。

充分利用多核机器的资源，提高性能。

> 缺点：
> 在极个别特殊场景中，一个Reactor线程负责监听和处理所有的客户端连接可能会存在性能问题。例如并发百万客户端连接（双十一、春运抢票）

### 1.3 多Reactor多线程模型

为了充分利用多核能力，可以构建两个 Reactor，这就演进为 「主从Reactor线程模型」 。

![深入Netty逻辑架构，从Reactor线程模型开始](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202303301016146.png)

 


1）主Reactor

主 Reactor 单独监听server socket，accept新连接，然后将建立的 SocketChannel 注册给指定的 从Reactor，

2）从Reactor

从Reactor 将连接加入到连接队列进行监听，并创建handler进行事件处理。执行事件的读写、分发，把业务处理就扔给worker线程池完成。

3）Worker线程池
处理所有业务事件，充分利用多核机器的资源，提高性能。

轻松处理百万并发。

> 缺点：
> 实现比较复杂。

不过有了Netty，一切都变得简单了。

Netty帮我们封装好了一切，可以快速使用主从Reactor线程模型（Netty4的实现上增加了无锁串行化设计），具体代码这里就不贴了，可以看看上一篇的Demo。

## 2.EventLoop、EventLoopGroup 怎么实现Reactor线程模型？

上面我们已经了解了Reactor线程模型，了解了它的核心就是：

```undefined
Reactor线程模式 = Reactor（I/O多路复用）+ 线程池
```

它的运行模式包括四个步骤：

- 连接注册：建立连接后，将channel注册到selector上
- 事件轮询：selcetor上轮询（select()函数）获取已经注册的channel的所有I/O事件（多路复用）
- 事件分发：把准备就绪的I/O事件分配到对应线程进行处理
- 事件处理：每个worker线程执行事件任务

那这样的模型在Netty中具体怎么实现呢？

这就需要我们了解下EventLoop和EventLoopGroup了。

### 2.1 EventLoop是什么

EventLoop 不是Netty独有的，它本身是一个通用的 事件等待和处理的程序模型。主要用来解决多线程资源消耗高的问题。例如 Node.js 就采用了 EventLoop 的运行机制。

那么，在Netty中，EventLoop是什么呢？

- 一个Reactor模型的事件处理器。
- 单独一个线程。
- 一个EventLoop内部会维护一个selector和一个「taskQueue任务队列」，分别负责处理 「I/O事件」 和 「任务」。

「taskQueue任务队列」是多生产者单消费者队列，在多线程并发添加任务时，可以保证线程安全。

「I/O事件」即selectionKey中的事件，如accept、connect、read、write等；

「任务」包括 普通任务、定时任务等。

- 普通任务：通过 NioEventLoop 的 execute() 方法向任务队列 taskQueue 中添加任务。例如 Netty 在写数据时会封装 WriteAndFlushTask 提交给 taskQueue。
- 定时任务：通过调用 NioEventLoop 的 schedule() 方法向 定时任务队列 scheduledTaskQueue 添加一个定时任务，用于周期性执行该任务（如心跳消息发送等）。定时任务队列的任务 到了执行时间后，会合并到 普通任务 队列中进行真正执行。

一图胜千言:

![深入Netty逻辑架构，从Reactor线程模型开始](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202303301016268.png)

 

EventLoop单线程运行，循环往复执行三个动作：

- selector事件轮询
- I/O事件处理
- 任务处理

### 2.2 EventLoopGroup是什么

EventLoopGroup比较简单，可以简单理解为一个“EventLoop线程池”。

![深入Netty逻辑架构，从Reactor线程模型开始](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202303301016319.png)

 

> Tips：
>
> 监听一个端口，只会绑定到 BossEventLoopGroup 中的一个 Eventloop，所以， BossEventLoopGroup 配置多个线程也无用，除非你同时监听多个端口。

### 2.3 具体实现

Netty可以通过简单配置，支持单Reactor单线程模型 、单Reactor多线程模型 、多Reactor多线程模型。

我们以 「多Reactor多线程模型」 为例，来看看Netty是如何通过EventLoop来实现的。

还是一图胜千言：

![深入Netty逻辑架构，从Reactor线程模型开始](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202303301016960.png)

 

我们结合Reactor线程模型的四个步骤来梳理一下：

1）连接注册

master EventLoopGroup中有一个EventLoop，绑定某个特定端口进行监听。

一旦有新的连接进来触发accept类型事件，就会在当前EventLoop的I/O事件处理阶段，将这个连接分配给slave EventLoopGroup中的某一个EventLoop，进行后续 事件的监听。

2）事件轮询

slave EventLoopGroup中的EventLoop，会通过selcetor对绑定到自身的channel进行轮询，获取已经注册的channel的所有I/O事件（多路复用）。

当然，EventLoopGroup中会有 多个EventLoop 运行，各自循环处理。具体EventLoop数量是由 用户指定的线程数 或者 默认为核数的2倍。

3）事件分发

当slave EventLoopGroup中的EventLoop获取到I/O事件后，会在EventLoop的 I/O事件处理（processSelectedKeys） 阶段分发给对应ChannelPipeline进行处理。

> 注意，仍然在当前线程进行串行处理

4）事件处理

在ChannelPipeline中对I/O事件进行处理。

I/O事件处理完后，EventLoop在 任务处理（runAllTasks） 阶段，对队列中的任务进行消费处理。

至此，我们就能完全梳理清楚EventLoopGroup/EventLoop 和 Reactor线程模型的关系了。

咦，好像有什么地方不对劲？

没错，细心的朋友可能会发现，slave EventLoopGroup中并不是

```undefined
一个selector + 线程池
```

而是有多个EventLoop组成的

```undefined
多selector + 多个单线程 
```

这是为什么呢？

那就得继续深入了解下Netty4的线程模型优化了。

## 3.深入Netty的线程模型优化

上文说过，对每个EventLoop来说，都是单线程运行，并循环往复执行三个动作：

- selector事件轮询
- I/O事件处理
- 任务处理

在slave EventLoopGroup中，并不是 “一个selector + 线程池”模式，而是有多个EventLoop组成的 “多selector + 多个单线程“ 模型，这是为什么呢？

这主要是因为我们分析的是Netty4的线程模型，跟Netty3的传统Reactor模型相比有了不同之处。

### 3.1 Netty3和Netty4的线程模型变化

在Netty3的线程模型中，分为 读事件处理模型 和 写事件处理模型。

![深入Netty逻辑架构，从Reactor线程模型开始](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202303301016272.png)

 

- read事件的ChannelHandler都是由Netty的 I/O 线程（对应Netty 4 中的 EventLoop）中负责执行。
- I/O线程调度执行ChannelPipeline中Handler链的对应方法，直到业务实现的End Handler。
- End Handler将消息封装成Runnable，放入到业务线程池中执行，I/O线程返回，继续读/写等I/O操作。

 

![深入Netty逻辑架构，从Reactor线程模型开始](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202303301016592.png)

 

- write事件是由调用线程处理，可能是 I/O 线程，也可能是业务线程。
- 如果是业务线程，那么业务线程会执行ChannelPipeline中的Channel Handler。
- 执行到系统最后一个ChannelHandler，将编码后的消息Push到发送队列中，业务线程返回。
- Netty的I/O线程从发送消息队列中取出消息，调用SocketChannel的write方法进行消息发送。

由上文可以看到，在Netty3的线程模型中，是采用“selector + 业务线程池”的模型。

> 注意，在这种模型下，读写模型不一致。尤其是读事件、写事件的「执行线程」是不一样的。

但是在Netty4的线程模型中，采用了“多selector + 多个单线程”模型。

![深入Netty逻辑架构，从Reactor线程模型开始](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202303301016842.png)

 


读事件：

- I/O线程NioEventLoop从SocketChannel中读取数据报，将ByteBuf投递到ChannelPipeline，触发ChannelRead事件；
- I/O线程NioEventLoop调用ChannelHandler链，直到将消息投递到业务线程，然后I/O线程返回，继续后续的操作。

写事件：

- 业务线程调用ChannelHandlerContext.write(Object msg)方法进行消息发送。
- ChannelHandlerInvoker将发送消息封装成 任务，放入到EventLoop的Mpsc任务队列中，业务线程返回。后续由EventLoop在循环中统一调度和执行。
- I/O线程EventLoop在进行 任务处理 时，从Mpsc任务队列中获取任务，调用ChannelPipeline进行处理，处理Outbound事件，直到将消息放入发送队列，然后唤醒Selector，执行写操作。

Netty4中，无论读写，都是通过I/O线程(也就是EventLoop)来统一处理。

为什么Netty4的线程模型做了这样的变化？答案就是 无锁串行化设计。

### 3.2 什么是Netty4线程模型的无锁串行化

我们先看看Netty3的线程模型存在什么问题：

- 读/写线程模型 不一致，带来额外的开发心智负担。
- 写操作由业务线程发起时，通常业务会使用 线程池多线程并发执行 某个业务流程，所以某一个时刻会有多个业务线程同时操作ChannelHandler，我们需要对ChannelHandler进行并发保护，大大降低了开发效率。
- 频繁的线程上下文切换，会带来额外的性能损耗。

而Netty4线程模型的 「无锁串行化」设计，就很好地解决了这些问题。

一图胜千言：

![深入Netty逻辑架构，从Reactor线程模型开始](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202303301016112.png)

 


从事件轮询、消息的读取、编码以及后续Handler的执行，始终都由I/O线程NioEventLoop内部进行串行操作，这就意味着整个流程不会进行线程上下文的切换，避免多线程竞争导致的性能下降，数据也不会面临被并发修改的风险。

表面上看，串行化设计似乎CPU利用率不高，并发程度不够。但是，通过调整slave EventLoopGroup的线程参数，可以同时启动多个NioEventLoop，串行化的线程并行运行，这种局部无锁化的串行线程设计相比「一个队列-多个工作线程模型」性能更优。

总结下Netty4无锁串行化设计的优点：

- 一个EventLoop会处理一个channel全生命周期的所有事件。从消息的读取、编码以及后续Handler的执行，始终都由I/O线程NioEventLoop负责。
- 每个EventLoop会有自己独立的任务队列。
- 整个流程不会进行线程上下文的切换，数据也不会面临被并发修改的风险。
- 对于用户而言，统一的读写线程模型，也降低了使用的心智负担。

## 4.从线程模型看最佳实践

NioEventLoop 无锁串行化的设计这么好，它就完美无缺了吗？

不是的！

在特定的场景下，Netty3的线程模型可能性能更高。比如编码和其它写操作非常耗时，由多个业务线程并发执行，性能肯定高于单个EventLoop线程串行执行。

因此，虽然单线程执行避免了线程切换，但是它的缺陷就是不能执行时间过长的 I/O 操作，一旦某个 I/O 事件发生阻塞，那么后续的所有 I/O 事件都无法执行，甚至造成事件积压。

所以，Netty4的线程模型的最佳实践需要注意以下两点：

- 无论读/写，不在自定义ChannelHandler中做耗时操作。
- 不把耗时操作放进 任务队列。

------

本文深入学习了Netty逻辑架构中的EventLoop，掌握Netty最精髓的知识点 线程模型。

从Reactor线程模型开始说起，到Netty如何用EventLoop实现Reactor线程模型。

然后对Netty4的线程模型优化做了详细介绍，尤其是「无锁串行化设计」。

最后从EventLoop线程模型出发，说明了日常开发中使用Netty4开发的最佳实践。

希望大家能对EventLoop有全面的认识。

另外，限于篇幅，EventLoop中有两个非常重要的数据结构没有展开介绍，你们知道是什么吗？

后面会单独写两篇进行分析，敬请期待。