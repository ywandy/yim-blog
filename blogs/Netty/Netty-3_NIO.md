---
title: Netty-3_NIO
tags: 
  - Netty
date: 2023-03-29 09:00:00
categories:
  - Netty
---

上一篇文章我们深入了解了I/O多路复用的三种实现形式，select/poll/epoll。

那Netty是使用哪种实现的I/O多路复用呢？这个问题，得从Java NIO包说起。

Netty实际上也是一个封装好的框架，它的网络I/O本质上还是使用了Java的NIO包(New IO，不是网络I/O模型的NIO，Nonblocking IO)包。所以，从网络I/O模型到Netty，我们还需要了解下Java NIO包。

本文预计阅读时间 5 分钟，将重点回答以下几个问题：

- 如何用Java NIO包实现一个服务端
- Java NIO包如何实现I/O多路复用模型
- 有了Java NIO包，为什么还要封装一个Netty?

## 1.先来看一个Java NIO服务端的例子

上一篇文章我们已经了解了I/O多路复用的实现形式。
就是多个的进程的IO可以注册到一个复用器（selector）上，然后用一个进程调用select，select会监听所有注册进来的IO。

NIO包做了对应的实现。如下图所示。

![从I/O多路复用到Netty，还要跨过Java NIO包](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202303291455915.png)

 

有一个统一的selector负责监听所有的Channel。这些channel中只要有一个有IO动作，就可以通过Selector.select（）方法检测到，并且使用selectedKeys得到这些有IO的channel，然后对它们调用相应的IO操作。

我们来个简单的demo做一下演示。如何使用NIO中三个核心组件（Buffer缓冲区、Channel通道、Selector选择器）来编写一个服务端程序。

```cpp
public class NioDemo {
    public static void main(String[] args) {
        try {
            //1.创建channel
            ServerSocketChannel socketChannel1 = ServerSocketChannel.open();
            //设置为非阻塞模式，默认是阻塞的
            socketChannel1.configureBlocking(false);
            socketChannel1.socket().bind(new InetSocketAddress("127.0.0.1", 8811));

            ServerSocketChannel socketChannel2 = ServerSocketChannel.open();
            socketChannel2.configureBlocking(false);
            socketChannel2.socket().bind(new InetSocketAddress("127.0.0.1", 8822));

            //2.创建selector，并将channel1和channel2进行注册。
            Selector selector = Selector.open();
            socketChannel1.register(selector, SelectionKey.OP_ACCEPT);
            socketChannel2.register(selector, SelectionKey.OP_ACCEPT);

            while (true) {
                //3.一直阻塞直到有至少有一个通道准备就绪
                int readChannelCount = selector.select();
                Set<SelectionKey> selectionKeys = selector.selectedKeys();
                Iterator<SelectionKey> iterator = selectionKeys.iterator();
                //4.轮训已经就绪的通道
                while (iterator.hasNext()) {
                    SelectionKey key = iterator.next();
                    iterator.remove();
                    //5.判断准备就绪的事件类型，并作相应处理
                    if (key.isAcceptable()) {
                        // 创建新的连接，并且把连接注册到selector上，并且声明这个channel只对读操作感兴趣。
                        ServerSocketChannel serverSocketChannel = (ServerSocketChannel)key.channel();
                        SocketChannel socketChannel = serverSocketChannel.accept();
                        socketChannel.configureBlocking(false);
                        socketChannel.register(selector, SelectionKey.OP_READ);
                    }
                    if (key.isReadable()) {
                        SocketChannel socketChannel = (SocketChannel) key.channel();
                        ByteBuffer readBuff = ByteBuffer.allocate(1024);
                        socketChannel.read(readBuff);
                        readBuff.flip();
                        System.out.println("received : " + new String(readBuff.array()));
                        socketChannel.close();
                    }
                }
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

通过这个代码示例，我们能清楚地了解如何用Java NIO包实现一个服务端：

- 1）创建channel1和channel2，分别监听特定端口。
- 2）创建selector，并将channel1和channel2进行注册。
- 3）selector.select()一直阻塞,直到有至少有一个通道准备就绪。
- 4）轮训已经就绪的通道
- 5）并根据事件类型做出相应的响应动作。

程序启动后，会一直阻塞在selector.select()。
通过浏览器调用localhost:8811 或者 localhost:8822就能触发我们的服务端代码了。

## 2.Java NIO包如何实现I/O多路复用模型

上文演示的Java NIO服务端已经比较清楚地展示了使用NIO编写服务端程序的过程。

那这个过程中如何实现了I/O多路复用的呢？

我们得深入看下selector的实现。

```kotlin
//2.创建selector，并将channel1和channel2进行注册。 
Selector selector = Selector.open();
```

从open这里开始吧。

![从I/O多路复用到Netty，还要跨过Java NIO包](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202303291455083.png)

 

这里用了一个SelectorProvider来创建selector。

进入SelectorProvider.provider(),看到具体的provider是由
sun.nio.ch.DefaultSelectorProvider创建的，对应的方法是：

![从I/O多路复用到Netty，还要跨过Java NIO包](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202303291455235.png)

 

咦？原来不同的操作系统会提供不同的provider对象。这里包括了PollSelectorProvider、EPollSelectorProvide等。

名字是不是有点眼熟？

没错，跟我们上一篇文章分析过的I/O多路复用的不同实现方式poll/epoll有关。

我们选择默认的
sun.nio.ch.PollSelectorProvider往下看看。

![从I/O多路复用到Netty，还要跨过Java NIO包](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202303291455364.png)

 

OK，找到了实现类PollSelectorImpl。

然后，通过以下调用:

![从I/O多路复用到Netty，还要跨过Java NIO包](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202303291455566.png)

 

找到最终的native方法poll0。

![从I/O多路复用到Netty，还要跨过Java NIO包](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202303291455624.png)

 

是不是仍然很眼熟？

没错！跟我们上一篇文章分析过的poll函数是一致的。

```cpp
int poll (struct pollfd *fds, unsigned int nfds, int timeout);
```

绕了这么久，到最后，还是找到了我们聊过I/O多路复用的 poll 实现。

至此，我们终于把Java NIO和 I/O多路复用模型串联起来了。

Java NIO包使用selector，实现了I/O多路复用模型。

同时，在不同的操作系统中，会有不同的poll/epoll选择。

## 3.为什么还需要Netty呢？

那既然已经有了NIO包了，我们可以自己手动编写服务框架了，为什么还需要封装一个Netty框架呢？有什么好处呢？

好处当然是有很多了！我们从一开始实现的demo说起。

### 3.1 设计模式的优化

我们的demo确实已经能够工作了，但是还是有比较明显的问题。第4步（轮询已经就绪的通道）和第5步（对事件作相应处理）是在同一个线程中的，当事件处理比较耗时甚至阻塞时，整个流程就会阻塞了。

我们使用的实际上就是 “单Reactor单线程” 设计模式。

![从I/O多路复用到Netty，还要跨过Java NIO包](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202303291455886.png)

 

这种模型在Reactor中负责监听端口、接收请求，如果是连接事件交给acceptor处理，如果是读写事件和业务处理就交给handler处理，但始终只有一个线程执行所有的事情。

为了提高性能，我们理所当然相当可以把事件处理交给线程池，那就可以演进为 “单Reactor多线程” 设计模式。

![从I/O多路复用到Netty，还要跨过Java NIO包](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202303291455145.png)

 

这种模型和第一种模型的主要区别是把业务处理从之前的单一线程脱离出来，换成线程池处理。Reactor线程只处理连接事件、读写事件，所有业务处理都交给线程池，充分利用多核机器的资源，提高性能。

但是这仍然不够！

我们可以发现，一个Reactor线程承担了所有的网络事件，例如监听和响应，高并发场景下单线程存在性能问题。

为了充分利用多核能力，可以构建两个 Reactor，主 Reactor 单独监听server socket，accept新连接，然后将建立的 SocketChannel 注册给指定的从 Reactor，从Reactor再执行事件的读写、分发，把业务处理就扔给worker线程池完成。这就演进为 ”主从Reactor模式“ 设计模式。

![从I/O多路复用到Netty，还要跨过Java NIO包](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202303291455870.png)

 

所以，如果有人直接帮我们 封装好这样的设计模式 ，是不是太好了？

> 没错，Netty就是这样的“活雷锋”!
> Netty就使用了主从Reactor模式封装了Java NIO包的使用，大大提高了性能。

#### 3.2 其他优点 (以后的核心知识点）

除了封装了高性能的设计模式外，Netty还有许多其他优点：

- 稳定性。 Netty 更加可靠稳定，修复和完善了 JDK NIO 较多已知问题，包括 select 空转导致 CPU 消耗 100%、keep-alive 检测等问题。
- 性能优化。对象池复用技术。 Netty 通过复用对象，避免频繁创建和销毁带来的开销。零拷贝技术。 除了操作系统级别的零拷贝技术外，Netty 提供了面向用户态的零拷贝技术，在 I/O 读写时直接使用 DirectBuffer，避免了数据在堆内存和堆外内存之间的拷贝。
- 便捷性。 Netty 提供了很多常用的工具，例如行解码器、长度域解码器等。如果我们使用JDK NIO包，那么这些常用工具都需要自己进行实现。

正是因为 Netty 做到了高性能、高稳定性、高易用性，完美弥补了 Java NIO 的不足，所以在我们在网络编程时，首选Netty，而不是自己直接使用Java NIO。

------

回顾一下前几章内容，到目前为止，我们从网络I/O模型出发，一步步了解到了Netty的网络I/O模型。

对于I/O多路复用、Java NIO包 和 Netty 的关系也有了全面的认识。

有了这些知识基础，我们初步了解了Netty是什么，为什么使用Netty。

后面的文章，我们将逐步展开Netty框架的核心知识点，敬请期待。