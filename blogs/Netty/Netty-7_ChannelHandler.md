---
title: Netty-7_ChannelHandler(2)
tags: 
  - Netty
date: 2023-03-29 09:00:00
categories:
  - Netty
---

上一篇文章我们深入学习了Netty逻辑架构中的核心组件ChannelHandler和ChannelPipeline，并介绍了它在日常开发使用中的最佳实践。文中也提到了，ChannelHandler主要用于数据输入、输出过程中的加工处理，比如编解码、异常处理等。

今天，我们就选取日常开发中最常用的一种ChannelHandler用途来学习——编解码器。

如果说ChannelHandler的学习是Netty的基础招式，那么编解码就是“基础招式”中衍生出的“常用招式“，我们往往会以一个ChannelHandler来实现编解码逻辑。无论是网络编程实战，还是面试八股文，都离不开编解码的知识。 

本文预计阅读时间约 15分钟，
将重点围绕以下几个问题展开：

- 学习编解码器，从粘包/拆包开始
- 如何实现自定义编解码器
- Netty有哪些开箱即用的编解码器

## 1.学习编解码器，从粘包/拆包开始

### 1.1为什么会有是粘包/拆包

粘包/拆包问题，相信大家都有所耳闻，这个问题的出现主要包括三个原因：

1）MTU 和 MSS 限制

MTU（Maxitum Transmission Unit） 是OSI五层网络模型中 数据链路层 对一次可以发送的最大数据的限制，一般来说大小为 1500 byte。

MSS（Maximum Segement Size） 是指 TCP报文中data部分的最大长度，它是传输层一次发送最大数据的大小限制。

MSS和MTU的关系如下所示：

> MSS长度=MTU长度 - IP Header - TCP Header

因此，当 MSS长度 + IP Header + TCP Header > MTU长度 时，就需要拆分多个报文进行发送，会导致“拆包”现象。

2）TCP滑动窗口
TCP的流量控制方法就是“滑动窗口”。当A向B发送数据时，B作为接收端会告知发送端A自己可以接受的窗口数值，以此来控制A的发送流量大小，从而达到流量控制的目的。

假设接收方B告知发送方A的窗口大小为256，意味着发送方最多还可以发送256个字节，而由于发送方的数据大小是518字节，因此只能发送前256字节，等到接收方ack后，才能发送剩余字节。会导致“拆包”现象。

3）Nagle算法

TCP/IP协议中，无论发送多少大小的数据，都要在数据(DATA)前面加上协议头(TCP Header + IP Header)。如果每次需要发送的数据只有 1 字节，加上 20 个字节 IP Header 和 20 个字节 TCP Header，每次发送的数据包大小为 41 字节，但真正有效的信息只有1个字节，这就造成了非常大的浪费。

因此，TCP/IP中使用Nagle 算法来提高效率。

Nagle 算法核心思想在于“化零为整“。它是在数据未得到确认之前先写入缓冲区，等待数据确认或者缓冲区积攒到一定大小再把数据包发送出去。

多个小数据包合并后一起发送出去，就造成了粘包。

> Q: 如果禁用了Nagle算法，还需要对粘包情况进行处理吗？
> A: 需要。除了Nagle算法外，接收端不及时也可能会造成粘包现象。当上一个数据包还在缓冲区未被接收端处理时，下一个数据包已经到达了，然后接收端根据缓冲区大小取到的数据有可能会取到多个数据包。

### 1.2 怎么处理粘包/拆包

对于TCP，其实我们都知道它的一个特点就是“面向字节流”的传输协议，本身并没有数据包的界限。所以不管什么原因造成了“粘包/拆包”，TCP协议本身的数据传输是可靠且正确的。

我们首先要明确一点：“粘包/拆包”导致的问题，本质上是应用层的数据解析问题。

因此，解决拆包/粘包问题的核心方法：定义应用层的通信协议。

> 核心在于定义正确的数据边界。

常见协议的解决方案包括三种：

1）固定长度

每个数据报文都约定一个固定的长度。

当接收方累计读取到固定长度的报文后，就认为已经获得一个完整的消息。

比如我们要发送一个ABCDEFGHIJKLM的消息，约定固定消息长度为4，那么接收方就可以按照4的长度来解析。如下所示。

|      |      |      |      |
| ---- | ---- | ---- | ---- |
| ABCD | EFGH | IJKL | MN00 |

当发送方的数据小于固定长度时，比如最后一个数据包，只有MN两个字符，这时候就需要空位补齐。

> 这种方案非常简单，但是缺点也非常明显，非常不灵活。
> 如果固定长度定义太长，就会浪费数据传输空间。如果定义太短，就会影响正确的数据传输。
> 这种方法一般不采用。

2）特定分隔符

除了固定长度外，我们比较容易想到的区分“数据边界”的方法，就是用“特定分隔符”。当接收方读到特定的分隔符，就认为拿到了一个完整的消息。

比如我们使用换行符 \n 来区分。

```undefined
AB\nCDEFG\nHIJK\nLMN\n
```

这种方法就比较灵活了，适应不同长度的消息。但是，必须要注意，“特殊分隔符”不能和消息内容重复，否则就会解析失败了。

因此，我们在实践过程中，可以考虑把消息进行编码（如base64），然后用编码字符集之外的符号作为“特定分隔符”。

> 这种方案一般用在协议比较简单的场景中。

3）消息长度+内容
一般项目开发中，最通用的方式还是采用 消息长度+内容 的方式进行处理。
比如定义一个这样的消息格式：

| 消息长度（比如4字节长度存储） | 消息内容 |
| ----------------------------- | -------- |
| 3                             | ABC      |

以这样一个格式存储，消息接收方在解析时，先读取4字节长度的信息作为”消息长度“，这里是3，表示消息长度为3字节。然后就读取3字节的消息内容作为 完整 的消息。

举个例子：

```undefined
2AB5CDEFG4HIJK3LMN
```

消息长度+内容 的方式非常灵活，可以应用于各种场景中。

> 注意，在消息头中，除了定义消息长度外，还可以自定义其他扩展字段，比如消息版本、算法类型等。

## 2.如何在Netty中实现自定义编解码器

上面我们了解了出现“粘包/拆包”的原因以及常用的解决方法。下面看看如何在Netty中实现自定义编解码器。

Netty作为一个优秀的网络通信框架，已经提供了非常丰富的处理编解码的抽象类，我们只需要自定义编解码算法扩展即可。

### 2.1 自定义编码器

我们先来看看自定义编码器。因为编码器比较简单，不需要关注「粘包/拆包问题」。

常用的编码抽象类包括MessageToByteEncoder 和 MessageToMessageEncoder，继承自
ChannelOutboundHandlerAdapter，操作的是Outbound相关数据。

![Netty常用招式——ChannelHandler与编解码](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202303301117716.png)

 

1）MessageToByteEncoder<I>
这个编码器用于消息对象编码成字节流。它提供了encode的抽象方法，我们只需要实现encode方法，就能进行自定义编码了。

编码器实现非常简单，不需要关注拆包/粘包问题。

我们举一个栗子，将String类型消息转换为字节流：

![Netty常用招式——ChannelHandler与编解码](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202303301117826.png)

 

2）MessageToMessageEncoder
这个编码器用于将一种消息对象编码成另一种消息对象。这里的第二个Message可以理解为任意一个对象。如果是使用ByteBuf对象的话，就和上面的MessageToByteEncoder是一样的了。

我们找一个Netty自带的栗子看看，StringEncoder：

![Netty常用招式——ChannelHandler与编解码](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202303301117855.png)

 

### 2.2 自定义解码器

解码器比编码器要复杂一些，因为需要考虑“拆包/粘包”问题。

由于接收方有可能没有接收到完整的消息，所以解码框架需要对入站的数据做缓冲操作，直至获取到完整的消息。

常用的解码器抽象类包括 ByteToMessageDecoder 和 MessageToMessageDecoder，继承自
ChannelInboundHandlerAdapter，操作的是Inbbound相关数据。

![Netty常用招式——ChannelHandler与编解码](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202303301117921.png)

 

一般通用的做法是使用 ByteToMessageDecoder 解析 TCP 协议，解决拆包/粘包问题。解析得到有效的 ByteBuf 数据，然后传递给后续的 MessageToMessageDecoder 做数据对象的转换。

![Netty常用招式——ChannelHandler与编解码](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202303301117949.png)

 

1）ByteToMessageDecoder
ByteToMessageDecoder解码器用于字节流解码成消息对象。

拿上面的“固定长度法”解决“粘包/拆包”举一个栗子，Netty自带的FixedLengthFrameDecoder。

![Netty常用招式——ChannelHandler与编解码](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202303301117607.png)

 

通过固定长度frameLength，来对消息进行解析。

> 生产实践中，可能会使用更加复杂的协议来实现自定义编解码，比如protobuf。

2）MessageToMessageDecoder
MessageToMessageDecoder解码器用于将一种消息对象解码成另一种消息对象。如果你需要对解析后的字节数据做对象模型的转换，这时候便需要用到这个解码器。

## 3.Netty有哪些开箱即用的解码器

作为一个优秀的网络编程框架，Netty除了支持扩展自定义编解码器外，还提供了非常丰富的开箱即用的编解码器。尤其是针对我们上文1.2节中提过的三种解决「粘包/拆包问题」的方式，都有开箱即用的实现。

### 3.1 固定长度解码器 FixedLengthFrameDecoder

这个解码器上文已经提到过，对应1.2节中的「固定长度解码」，这里再稍微展开一下。

![Netty常用招式——ChannelHandler与编解码](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202303301117866.png)

  

通过构造函数配置固定长度 frameLength，然后在decode时，按照frameLength 进行解码。

- 当读取到长度大小为 frameLength 的消息，那么解码器认为已经获取到了一个完整的消息。
- 当消息长度小于 frameLength，FixedLengthFrameDecoder 解码器会一直等后续数据包的到达，直至获得完整的消息。

### 3.2 特殊分隔符解码器 DelimiterBasedFrameDecoder

这个解码器对应1.2节中的「特殊分隔符解码」，也是一个继承自ByteToMessageDecoder的解码器。

这个解码器会使用 1个 或 多个 符号delimiter 对传入的消息（ByteBuf)进行解码。

我们看一下构造器，了解一下几个重要参数。

![Netty常用招式——ChannelHandler与编解码](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202303301117992.png)

 

 

- maxFranmeLength

maxFranmeLength 是待处理消息的最大长度限制。如果超过 maxFranmeLength 还没有检测到指定分隔符，将会抛出 TooLongFrameException。

- stripDelimiter

stripDelimiter是一个boolean类型， 用于判断解码后得到的消息是否移除分隔符。如果 stripDelimiter=false，那么解码后的消息内容就会保留分隔符信息。

- failFast

failFast是一个boolean类型。如果为true，那么消息在超出 maxFranmeLength 后，会立即抛出 TooLongFrameException。如果为false，那么会等到解码出一个完整的消息后才会抛出TooLongFrameException。

- delimiters

delimiters 的类型是 ByteBuf 数组，可以在构造器中同时传入多个分隔符，但是在解析时，最终会选择长度最短的分隔符进行消息拆分。

例如收到的数据为：

```undefined
ABCD\nEFG\r\n 
```

如果指定的分隔符为 \n 和 \r\n，那么会解码出两个消息。

```undefined
ABCD   EFG
```

如果指定的特定分隔符只有 \r\n，那么只会解码出一个消息：

```undefined
ABCD\nEFG 
```

## 3.3 长度域解码器 LengthFieldBasedFrameDecoder

这个解码器是生产实践中运用比较广泛的一种（比如RocketMQ），相对复杂，但是特别灵活，基本能覆盖各种基于长度进行拆包的方案，比如1.2节中提到的「消息长度+内容」的方案。

使用这个解码器的时候，重点需要了解4个参数，掌握了参数的设置，就能快速实现不同的基于长度的拆包解码方案。

| 参数名              | 类型 | 含义                                                         |
| ------------------- | ---- | ------------------------------------------------------------ |
| lengthFieldOffset   | int  | 长度字段的偏移量。表示「长度域」的起始位置                   |
| lengthFieldLength   | int  | 长度字段所占用的字节数                                       |
| lengthAdjustment    | int  | 消息长度的修正值。表示一些复杂协议中，会在「长度域」添加一些其他内容，如版本号、消息类型等，这就需要修正值进行修正处理 |
| initialBytesToStrip | int  | 解码后需要跳过的初始字节数。表示消息内容数据的起始位置       |

1）解码方案一：基于消息长度 + 消息内容，解码结果不截断消息头

报文只包含消息长度 Length 和消息内容 Content 字段，其中 Length 为 16 进制表示，共占用 2 字节，Length 的值 0x000C 代表 Content 占用 12 字节。

| 参数名              | 取值                          |
| ------------------- | ----------------------------- |
| lengthFieldOffset   | 0                             |
| lengthFieldLength   | 2                             |
| lengthAdjustment    | 0                             |
| initialBytesToStrip | 0（表示解码结果不截断消息头） |

解码示例：

![Netty常用招式——ChannelHandler与编解码](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202303301117691.png)

 

2）解码方案二：基于消息长度 + 消息内容，解码结果截断

与方案一不同之处在于，解码结果会截断消息头（跳过2字节）

| 参数名              | 取值                                                        |
| ------------------- | ----------------------------------------------------------- |
| lengthFieldOffset   | 0                                                           |
| lengthFieldLength   | 2                                                           |
| lengthAdjustment    | 0                                                           |
| initialBytesToStrip | 2（表示跳过 Length 字段的字节长度，解码后 只包含 消息内容） |

解码示例：

![Netty常用招式——ChannelHandler与编解码](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202303301117130.png)

 

 

3）解码方案三：基于消息头 + 消息长度 + 消息内容

消息起始位置添加特殊消息头，消息长度 Length字段 后移。

| 参数名              | 取值                          |
| ------------------- | ----------------------------- |
| lengthFieldOffset   | 2                             |
| lengthFieldLength   | 3                             |
| lengthAdjustment    | 0                             |
| initialBytesToStrip | 0（表示解码结果不截断消息头） |

解码示例：

![Netty常用招式——ChannelHandler与编解码](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202303301117707.png)

 

4）解码方案四：基于消息长度 + 消息头 + 消息内容

消息起始位置为消息长度 Length字段，后面并不直接添加 消息内容，而是先添加 消息头header，再添加 消息内容。

| 参数名              | 取值                          |
| ------------------- | ----------------------------- |
| lengthFieldOffset   | 0                             |
| lengthFieldLength   | 3                             |
| lengthAdjustment    | 2 （Header1的长度）           |
| initialBytesToStrip | 0（表示解码结果不截断消息头） |

解码示例：

![Netty常用招式——ChannelHandler与编解码](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202303301117052.png)

 

由于 Length 后面不是马上添加content，所以需要加上 lengthAdjustment（2 字节）才能得到 Header + Content 的内容（14 字节）。

## 4.小结

来简单回顾下吧。

本文主要介绍了ChannelHandler的一种典型应用场景——编解码器。

编解码器核心关注点在于「粘包/拆包」的处理，我们介绍了「粘包/拆包」产生的原因以及常用解决方案。然后说明了如何使用Netty框架实现自定义编解码器。

最后，介绍了Netty中非常好用的几个开箱即用的编解码器。