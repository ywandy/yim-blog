---
title: Netty-4_NettyDemo
tags: 
  - Netty
date: 2023-03-29 09:00:00
categories:
  - Netty
---

上一篇文章我们对于I/O多路复用、Java NIO包 和 Netty 的关系有了全面的认识。

到目前为止，我们已经从I/O模型出发，逐步接触到了Netty框架。这个过程中，基本解答了Netty是什么、为什么使用Netty等前置问题。给我们学习Netty提供了最原始的背景知识。

有了这些做基础，下面我们可以开始慢慢去揭开Netty的神秘面纱了。

 本文预计阅读时间约 5分钟，将重点围绕以下几个问题展开：

- 如何用Netty编写一个Server端服务Demo
- 从Demo看Netty的逻辑架构，初识各个组件

## 1.编写一个Server端Demo

### 1.1 基于主从Reactor模式的Demo实现

如果从来没用过Netty，那么了解一下用Netty编写的Server端Demo是必不可少的。

还记得我们上一篇说的 “主从Reactor模式” 吗？可以构建两个 Reactor，主 Reactor 单独监听server socket，accept新连接，然后将建立的 SocketChannel 注册给指定的从 Reactor，从Reactor再执行事件的读写、分发，把业务处理就扔给worker线程池完成。

![从一个Demo开始，揭开Netty的神秘面纱](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202303300955411.jpeg)

 

我们就按照这个模式，用Netty编写一个服务端程序吧。

直接上代码！

一个简单的自定义ChannelHandler类，用来自定义业务处理逻辑：

![从一个Demo开始，揭开Netty的神秘面纱](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202303300955819.png)

 

一个包含Bootstrap的服务端启动类：

```java
public class EchoServer {
    private int port;

    public EchoServer(int port) {
        this.port = port;
    }

    public static void main(String[] args) throws Exception {
        new EchoServer(8833).start();
    }

    public void start() throws Exception {
        //1.Reactor模型的主、从多线程
        EventLoopGroup mainGroup = new NioEventLoopGroup();
        EventLoopGroup childGroup = new NioEventLoopGroup();

        try {
            //2.构造引导器实例ServerBootstrap
            ServerBootstrap b = new ServerBootstrap();
            b.group(mainGroup, childGroup)
                    .channel(NioServerSocketChannel.class) //2.1 设置NIO的channel
                    .localAddress(new InetSocketAddress(port)) //2.2 配置本地监听端口
                    .childHandler(new ChannelInitializer<SocketChannel>() { //2.3 初始化channel的时候，配置Handler
                        @Override
                        protected void initChannel(final SocketChannel socketChannel) {
                            socketChannel.pipeline()
                                    .addLast("codec", new HttpServerCodec())
                                    .addLast("compressor", new HttpContentCompressor())
                                    .addLast("aggregator", new HttpObjectAggregator(65536))
                                    .addLast("handler", new EchoServerHandler()); //2.4 加入自定义业务逻辑ChannelHandler
                        }
                    });
            ChannelFuture f = b.bind().sync(); //3.启动监听
            System.out.println("Http Server started， Listening on " + port);
            f.channel().closeFuture().sync();
        } finally {
            mainGroup.shutdownGracefully().sync();
            childGroup.shutdownGracefully().sync();
        }
    }
}
```

启动后，通过curl调用，得到响应。

![从一个Demo开始，揭开Netty的神秘面纱](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202303300955273.png)

 

Demo完成了！

对于之前觉得用Java NIO包实现起来很复杂的的 “主从Reactor模式” ，用Netty简简单单就完成了。

![从一个Demo开始，揭开Netty的神秘面纱](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202303300955376.png)

 

只需要创建两个EventLoopGroup，然后绑定到引导器ServerBootstrap上就好了.

mainGroup 是主 Reactor，childGroup 是从 Reactor。它们分别使用不同的 NioEventLoopGroup，主 Reactor 负责处理 Accept，然后把 Channel 注册到从 Reactor 上，从 Reactor 主要负责 Channel 生命周期内的所有 I/O 事件。

### 1.2 Demo分析

从上面的Demo代码可以看出，对于所有用Netty编写的服务端程序，至少需要两个部分：

- 至少一个ChannelHandler
- Bootstrapping

1）ChannelHandler

这个组件用来实现对客户端发送过来的数据进行处理，可能包括编解码、自定义业务逻辑处理等等。

对于ChannelHandler来说，有非常多的实现。在Demo中我们简单使用了几个Netty自带的Handler，包括HttpServerCodec、HttpContentCompressor、HttpObjectAggregator，也使用了一个自定义的EchoServerHandler。

可以看到，对于Handler的使用，是非常重要也是非常方便的一个环节。我们会在以后的文章中详细展开。

2）Bootstrapping

启动代码部分。用来配置服务端的启动参数，包括监听端口、服务端线程池配置、网络连接属性配置、ChannelHandler配置等等。

结合Demo来看，主要分为这几个步骤：

- 创建一个ServerBootstrap实例，用来引导启动。
- 创建一个（当我们使用主从Reactor模式时，需要创建两个）NioEventLoopGroup实例来处理事件， 比如接受一个新的客户端连接、读写数据等。
- 指定一个端口，用来作为服务端的监听端口。
- 使用一系列channelHandler来初始化每个Channel，包括自定义业务逻辑实现的channelHandler。
- 调用ServerBootstrap.bind() 来真正触发启动。

## 2. Netty的逻辑架构

通过上面的Demo演示，我们对 Netty 的使用已经有了一个大概的印象。

下面，我们根据Demo中使用的几个组件，一起梳理一下 Netty 的逻辑架构。

![从一个Demo开始，揭开Netty的神秘面纱](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202303300955640.png)

 

结合我们的Demo和这个逻辑架构图，我们梳理下各个组件的流转过程：

- 服务端利用ServerBootstrap进行启动引导，绑定监听端口
- 启动初始化时有 main EventLoopGroup 和 child EventLoopGroup 两个组件，其中 main EventLoopGroup负责监听网络连接事件。当有新的网络连接时，就将 Channel 注册到 child EventLoopGroup。
- child EventLoopGroup 会被分配一个 EventLoop 负责处理该 Channel 的读写事件。
- 当客户端发起 I/O 读写事件时，服务端 EventLoop 会进行数据的读取，然后通过 ChannelPipeline 依次有序触发各种ChannelHandler进行数据处理。
- 客户端数据会被依次传递到 ChannelPipeline 的 ChannelInboundHandler 中，在一个handler中处理完后就会传入下一个handler。
- 当数据写回客户端时，会将处理结果依次传递到 ChannelPipeline 的 ChannelOutboundHandler 中，在一个handler中处理完后就会传入下一个handler，最后返回客户端。

以上便是 Netty 各个组件的逻辑架构，我们暂时只需要了解个大致框架即可，后面我们会详细介绍各个组件。

有几个比较常见的问题在这里总结下：

1）什么是Channel
Channel 的字面意思是“通道”，它是网络通信的载体，提供了基本的 API 用于网络 I/O 操作，如 register、bind、connect、read、write、flush 等。

Netty 实现的 Channel 是以 JDK NIO Channel 为基础的，提供了更高层次的抽象，屏蔽了底层 Socket。

2）什么是ChannleHandler和ChannelPipeline

ChannelHandler实现对客户端发送过来的数据进行处理，可能包括编解码、自定义业务逻辑处理等等。

ChannelPipeline 负责组装各种 ChannelHandler，当 I/O 读写事件触发时，ChannelPipeline 会依次调用 ChannelHandler 列表对 Channel 的数据进行拦截和处理。

3）什么是EventLoopGroup？

EventLoopGroup 本质是一个线程池， 是 Netty Reactor 线程模型的具体实现方式，主要负责接收 I/O 请求，并分配线程执行处理请求。我们在demo中使用了它的实现类 NioEventLoopGroup，也是 Netty 中最被推荐使用的线程模型。

我们还通过构建main EventLoopGroup 和 child EventLoopGroup 实现了 “主从Reactor模式”。

4）EventLoopGroup、EventLoop、Channel有什么关系？

一个 EventLoopGroup 往往包含一个或者多个 EventLoop。

EventLoop 用于处理 Channel 生命周期内的所有 I/O 事件，如 accept、connect、read、write 等 I/O 事件。

EventLoop 同一时间会与一个线程绑定，每个 EventLoop 负责处理多个 Channel。