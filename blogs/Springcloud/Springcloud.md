---
title: Springcloud总结
tags: 
  - Springcloud
date: 2021-10-12 20:56:08
categories:
  - Springcloud
---

### **1.Nacos**

集成了注册中心和配置中心，为服务提供可视化监控

有单点启动和集群启动方式，防止单点故障导致服务不可用。

主要通过心跳机制发现可用的服务

心跳包会包含当前服务实例的名称、IP、端口、集群名、权重等信息。

### **2.OpenFeign**

提供服务间的远程调用，主要使用到的技术是rpc

底层基于 Netflix Feign，Netflix Feign 是 Netflix 设计的开源的声明式 WebService 客户端，用于简化服务间通信。

Netflix Feign 采用“接口+注解”的方式开发，通过模仿 RPC 的客户端与服务器模式（CS），采用接口方式开发来屏蔽网络通信的细节。

OpenFeign 则是在 Netflix Feign 的基础上进行封装，结合原有 Spring MVC 的注解，对 Spring Cloud 微服务通信提供了良好的支持。

### **3.Ribbon**

负载均衡器，通过raft算法进行一个选举，是通过一个票数进行选举

负载均衡

是指通过软件或者硬件措施。它将来自客户端的请求按照某种策略平均的分配到集群的每一个节点上，保证这些节点的 CPU、内存等设备负载情况大致在一条水平线，避免由于局部节点负载过高产生宕机，再将这些处理压力传递到其他节点上产生系统性崩溃。

负载均衡按实现方式分类可区分为：服务端负载均衡与客户端负载均衡。

服务端负载均衡顾名思义，在架构中会提供专用的负载均衡器，由负载均衡器持有后端节点的信息，服务消费者发来的请求经由专用的负载均衡器分发给服务提供者，进而实现负载均衡的作用。目前常用的负载均衡器软硬件有：F5、Nginx、HaProxy 等

客户端负载均衡是指，在架构中不再部署额外的负载均衡器，在每个服务消费者内部持有客户端负载均衡器，由内置的负载均衡策略决定向哪个服务提供者发起请求。说到这，我们的主角登场了，Netfilx Ribbon 是 Netflix 公司开源的一个负载均衡组件，是属于客户端负载均衡器。目前Ribbon 已被 Spring Cloud 官方技术生态整合，运行时以 SDK 形式内嵌到每一个微服务实例中，为微服务间通信提供负载均衡与高可用支持。为了更容易理解，我们通过应用场景说明 Ribbon 的执行流程。假设订单服务在查询订单时需要附带对应商品详情，这就意味着订单服务依赖于商品服务，两者必然产生服务间通信，此时 Ribbon 的执行过程如下图所示：

![](https://pic-1304161434.cos.ap-guangzhou.myqcloud.com/img/202201271032677.png)

1. 订单服务（order-service）与商品服务（goods-service）实例在启动时向 Nacos 注册；
2. 订单服务向商品服务发起通信前，Ribbon 向 Nacos 查询商品服务的可用实例列表；
3. Ribbon 根据设置的负载策略从商品服务可用实例列表中选择实例；
4. 订单服务实例向商品服务实例发起请求，完成 RESTful 通信；

如何配置 Ribbon 负载均衡策略

Ribbon 内置多种负载均衡策略，常用的分为以下几种。

RoundRobinRule：

轮询策略，Ribbon 默认策略。默认超过 10 次获取到的 server 都不可用，会返回⼀个空的 server。

RandomRule：

随机策略，如果随机到的 server 为 null 或者不可用的话。会不停地循环选取。

RetryRule：

重试策略，⼀定时限内循环重试。默认继承 RoundRobinRule，也⽀持自定义注⼊，RetryRule 会在每次选取之后，对选举的 server 进⾏判断，是否为 null，是否 alive，并且在 500ms 内会不停地选取判断。而 RoundRobinRule 失效的策略是超过 10 次，RandomRule 没有失效时间的概念，只要 serverList 没都挂。

BestAvailableRule：

最小连接数策略，遍历 serverList，选取出可⽤的且连接数最小的⼀个 server。那么会调用 RoundRobinRule 重新选取。

AvailabilityFilteringRule：

可用过滤策略。扩展了轮询策略，会先通过默认的轮询选取⼀个 server，再去判断该 server 是否超时可用、当前连接数是否超限，都成功再返回。

ZoneAvoidanceRule：

区域权衡策略。扩展了轮询策略，除了过滤超时和链接数过多的 server，还会过滤掉不符合要求的 zone 区域⾥⾯的所有节点，始终保证在⼀个区域/机房内的服务实例进行轮询。

配置的时候，只需要在application.yml中进行配置

provider-service: #服务提供者的微服务id  ribbon:    NFLoadBalancerRuleClassName: com.netflix.loadbalancer.RandomRule 

### **4.Gateway**

网关服务，避免服务直接对接请求，因为不安全。通常配合spring security使用，为了保持高可用，一般会做一个集群的配置，注册到nacos中。

网关的作用

1. 服务将所有 API 接口对外直接暴露给用户端，这本身就是不安全和不可控的，用户可能越权访问不属于它的功能，例如普通的用户去访问管理员的高级功能。
2. 后台服务可能采用不同的通信方式，如服务 A 采用 RESTful 通信，服务 B 采用 RPC 通信，不同的接入方式让用户端接入困难。尤其是 App 端接入 RPC 过程更为复杂。
3. 在服务访问前很难做到统一的前置处理，如服务访问前需要对用户进行鉴权，这就必须将鉴权代码分散到每个服务模块中，随着服务数量增加代码将难以维护。

![](https://pic-1304161434.cos.ap-guangzhou.myqcloud.com/img/202201271032905.png)

用户端直接访问微服务

为了解决以上问题，API 网关应运而生，加入网关后应用架构变为下图所示。

![](https://pic-1304161434.cos.ap-guangzhou.myqcloud.com/img/202201271032092.png)

当引入 API 网关后，在用户端与微服务之间建立了一道屏障，通过 API 网关为微服务访问提供了统一的访问入口，所有用户端的请求被 API 网关拦截并在此基础上可以实现额外功能，例如：

- 针对所有请求进行统一鉴权、熔断、限流、日志等前置处理，让微服务专注自己的业务。
- 统一调用风格，通常 API 网关对外提供 RESTful 风格 URL 接口。用户传入请求后，由 API 网关负责转换为后端服务需要的 RESTful、RPC、WebService 等方式，这样便大幅度简化用户的接入难度。
- 更好的安全性，在通过 API 网关鉴权后，可以控制不同角色用户访问后端服务的权利，实现了服务更细粒度的权限控制。
- API 网关是用户端访问 API 的唯一入口，从用户的角度来说只需关注 API 网关暴露哪些接口，至于后端服务的处理细节，用户是不需要知道的。从这方面讲，微服务架构通过引入 API 网关，将用户端与微服务的具体实现进行了解耦。

Springcloud gateway

与 Zuul 是“别人家的孩子”不同，Spring Cloud Gateway 是 Spring 自己开发的新一代 API 网关产品。它基于 NIO 异步处理，摒弃了 Zuul 基于 Servlet 同步通信的设计，因此拥有更好的性能。同时，Spring Cloud Gateway 对配置进行了进一步精简，比 Zuul 更加简单实用。

**以下是 Spring Cloud Gateway 的关键特征：**

- 基于 JDK 8+ 开发；
- 基于 Spring Framework 5 + Project Reactor + Spring Boot 2.0 构建；
- 支持动态路由，能够匹配任何请求属性上的路由；
- 支持基于 HTTP 请求的路由匹配（Path、Method、Header、Host 等）；
- 过滤器可以修改 HTTP 请求和 HTTP 响应（增加/修改 Header、增加/修改请求参数、改写请求 Path 等等）；

![](https://pic-1304161434.cos.ap-guangzhou.myqcloud.com/img/202201271032209.png)

执行流程如下

1. Gateway、service-a 这些都是微服务实例，在启动时向 Nacos 注册登记；
2. 用户端向 Gateway 发起请求，请求地址http://192.168.31.103:80/service-a/list；
3. Gateway 网关实例收到请求，解析其中第二部分 service-a，即微服务 Id，第三部分 URI 为“/list”。之后向 Nacos 查询 service-a 可用实例列表；
4. Nacos 返回 120 与 121 两个可用微服务实例信息；
5. Spring Cloud Gateway 内置 Ribbon，根据默认轮询策略将请求转发至 120 实例，转发的完整 URL 将附加用户的 URI，即http://192.168.31.120:80/list；
6. 120 实例处理后返回 JSON 响应数据给 Gateway；
7. Gateway 返回给用户端，完成一次完整的请求路由转发过程。

**Gateway的执行原理**

![](https://pic-1304161434.cos.ap-guangzhou.myqcloud.com/img/202201271032709.png)

按执行顺序可以拆解以下几步：

1. Spring Cloud Gateway 启动时基于 Netty Server 监听指定的端口（该端口可以通过 server.port 属性自定义）。当前端应用发送一个请求到网关时，进入 Gateway Handler Mapping 处理过程，网关会根据当前 Gateway 所配置的谓词（Predicate）来决定是由哪个微服务进行处理。

1. 确定微服务后，请求向后进入 Gateway Web Handler 处理过程，该过程中 Gateway 根据过滤器（Filters）配置，将请求按前后顺序依次交给 Filter 过滤链进行前置（Pre）处理，前置处理通常是对请求进行前置检查，例如：判断是否包含某个指定请求头、检查请求的 IP 来源是否合法、请求包含的参数是否正确等。

1. 当过滤链前置（Pre）处理完毕后，请求会被 Gateway 转发到真正的微服务实例进行处理，微服务处理后会返回响应数据，这些响应数据会按原路径返回被 Gateway 配置的过滤链进行后置处理（Post），后置处理通常是对响应进行额外处理，例如：将处理过程写入日志、为响应附加额外的响应头或者流量监控等。

可以看到，在整个处理过程中谓词（Predicate）与过滤器（Filter）起到了重要作用，谓词决定了路径的匹配规则，让 Gateway 确定应用哪个微服务，而 Filter 则是对请求或响应作出实质的前置、后置处理。

**按执行流程，Sentinel 的执行流程分为三个阶段：**

1. Sentinel Core 与 Sentinel Dashboard 建立连接；
2. Sentinel Dashboard 向 Sentinel Core 下发新的保护规则；
3. Sentinel Core 应用新的保护规则，实施限流、熔断等动作。

**第一步，建立连接。**

Sentine Core 在初始化的时候，通过 application.yml 参数中指定的 Dashboard 的 IP地址，会主动向 dashboard 发起连接的请求。

\#Sentinel Dashboard通信地址 spring:   cloud:    sentinel:       transport:        dashboard: 192.168.31.10:9100

该请求是以心跳包的方式定时向 Dashboard 发送，包含 Sentinel Core 的 AppName、IP、端口信息。这里有个重要细节：Sentinel Core为了能够持续接收到来自 Dashboard的数据，会在微服务实例设备上监听 8719 端口，在心跳包上报时也是上报这个 8719 端口，而非微服务本身的 80 端口。

![](https://pic-1304161434.cos.ap-guangzhou.myqcloud.com/img/202201271032874.png)

在 Sentinel Dashboard 接收到心跳包后，来自 Sentinel Core的AppName、IP、端口信息会被封装为 MachineInfo 对象放入 ConcurrentHashMap 保存在 JVM的内存中，以备后续使用。

**第二步，推送新规则。**

如果在 Dashboard 页面中设置了新的保护规则，会先从当前的 MachineInfo 中提取符合要求的微服务实例信息，之后通过 Dashboard内置的 transport 模块将新规则打包推送到微服务实例的 Sentinel Core，Sentinel Core收 到新规则在微服务应用中对本地规则进行更新，这些新规则会保存在微服务实例的 JVM 内存中。

![](https://pic-1304161434.cos.ap-guangzhou.myqcloud.com/img/202201271032130.png)

**第三步，处理请求。**

Sentinel Core 为服务限流、熔断提供了核心拦截器 SentinelWebInterceptor，这个拦截器默认对所有请求 /** 进行拦截，然后开始请求的链式处理流程，在对于每一个处理请求的节点被称为 Slot（槽），通过多个槽的连接形成处理链，在请求的流转过程中，如果有任何一个 Slot 验证未通过，都会产生 BlockException，请求处理链便会中断，并返回“Blocked by sentinel" 异常信息。

![](https://pic-1304161434.cos.ap-guangzhou.myqcloud.com/img/202201271032570.png)

那这些 Slot 都有什么作用呢？我们需要了解一下，默认 Slot 有7 个，前 3 个 Slot为前置处理，用于收集、统计、分析必要的数据；后 4 个为规则校验 Slot，从Dashboard 推送的新规则保存在“规则池”中，然后对应 Slot 进行读取并校验当前请求是否允许放行，允许放行则送入下一个 Slot 直到最终被 RestController 进行业务处理，不允许放行则直接抛出 BlockException 返回响应。

**以下是每一个 Slot 的具体职责：**

- NodeSelectorSlot 负责收集资源的路径，并将这些资源的调用路径，以树状结构存储起来，用于根据调用路径来限流降级；
- ClusterBuilderSlot 则用于存储资源的统计信息以及调用者信息，例如该资源的 RT（运行时间）, QPS, thread count（线程总数）等，这些信息将用作为多维度限流，降级的依据；
- StatistcSlot 则用于记录，统计不同维度的runtime 信息；
- SystemSlot 则通过系统的状态，例如CPU、内存的情况，来控制总的入口流量；
- AuthoritySlot 则根据黑白名单，来做黑白名单控制；
- FlowSlot 则用于根据预设的限流规则，以及前面 slot 统计的状态，来进行限流；
- DegradeSlot 则通过统计信息，以及预设的规则，来做熔断降级。

### **5.skywalking（Sleuth）配合zipkin**

作为链路追踪

### **6.seata**

实现分布式事务。

**分布式事务2pc和3pc**

seata

![img](E:\soft\m18029363848@163.com\4d71c11143cd4b70b56b0d73e4cacc07\580dd77c4ab3b91f76aab866ce67cbb.png)

解决的是在2pc和3pc中的问题，当3pc做最后一步提交的时候，如果突然断点，就会产生数据锁定的问题，多线程环境下，会导致资源阻塞

而反向sql则不会

<img src="https://pic-1304161434.cos.ap-guangzhou.myqcloud.com/img/202201271034313.png" alt="at" style="zoom:75%;" />

### **7.RocketMQ**

详见RocketMQ文档

### **8.Alibaba Sentinel**

微服务限流架构

学习限流之前先要清楚什么是微服务的雪崩效应和如何避免与处理这个雪崩效应

**雪崩效应**

“雪崩”一词指的是山地积雪由于底部溶解等原因造成的突然大块塌落的现象，具有很强的破坏力。

在微服务项目中指由于突发流量导致某个服务不可用，从而导致上游服务不可用，并产生级联效应，最终导致整个系统不可用，使用雪崩这个词来形容这一现象最合适不过。

**解决办法一般有**

- 提高可用性，将单台设备转为多台负载均衡集群。
- 提高性能，检查慢 SQL、优化算法、引入缓存来缩短单笔业务的处理时间。
- 预防瞬时 TPS 激增，将系统限流作为常态加入系统架构。
- 完善事后处理，遇到长响应，一旦超过规定窗口时间，服务立即返回异常，中断当前处理。
- 加强预警与监控，引入 ELK，进行流量实时监控与风险评估，及时发现系统风险。

**为什么会产生雪崩效应**

如下图所示，假如我们开发了一套分布式应用系统，前端应用分别向 A/H/I/P 四个服务发起调用请求：

![](https://pic-1304161434.cos.ap-guangzhou.myqcloud.com/img/202202142055898.png)

前端应用需要四个服务

但随着时间推移，假如服务 I 因为优化问题，导致需要 20 秒才能返回响应，这就必然会导致 20 秒内该请求线程会一直处于阻塞状态。

![](https://pic-1304161434.cos.ap-guangzhou.myqcloud.com/img/202201271029466.png)

其中一个出现长延时，会导致前端应用线程阻塞

但是，如果这种状况放在高并发场景下，就绝对不允许出现，假如在 20 秒内有 10 万个请求通过应用访问到后端微服务。容器会因为大量请求阻塞积压导致连接池爆满，而这种情况后果极其严重！轻则“服务无响应”，重则前端应用直接崩溃。

以上这种因为某一个节点长时间处理导致应用请求积压崩溃的现象被称为微服务的“雪崩效应”。

![](https://pic-1304161434.cos.ap-guangzhou.myqcloud.com/img/202202142055121.png)

当大量线程积压后，前端应用崩溃，雪崩效应产生

**如何有效避免雪崩效应？**

1. **采用限流方式进行预防：**可以采用限流方案，控制请求的流入，让流量有序的进入应用，保证流量在一个可控的范围内。
2. **采用服务降级与熔断进行补救：**针对响应慢问题，可以采用服务降级与熔断进行补救。服务降级是指当应用处理时间超过规定上限后，无论服务是否处理完成，便立即触发服务降级，响应返回预先设置的异常信息。

下图为例，在用户支付完成后，通过消息通知服务向用户邮箱发送“订单已确认”的邮件。假设消息通知服务出现异常，需要 10 秒钟才能完成发送请求， 这是不能接受的。为了预防雪崩，我们可以在微服务体系中增加服务降级的功能，预设 2 秒钟有效期，如遇延迟便最多允许 2 秒，2 秒内服务未处理完成则直接降级并返回响应，此时支付服务会收到“邮件发送超时”的错误信息。这也就意味着消息通知服务最多只有两秒钟的处理时间。处理结果，要么发送成功，要么超时降级。 因此阻塞时间缩短，产生雪崩的概率会大大降低。

![](https://pic-1304161434.cos.ap-guangzhou.myqcloud.com/img/202201271029903.png)

所以Alibaba Sentinel就很好的解决了这方面的问题

Sentinel 以流量为切入点，从流量控制、熔断降级、系统负载保护等多个维度保护服务的稳定性。

Sentinel 具有以下特征。

- **丰富的应用场景：**Sentinel 承接了阿里巴巴近 10 年的双十一大促流量的核心场景，例如秒杀（即突发流量控制在系统容量可以承受的范围）、消息削峰填谷、集群流量控制、实时熔断下游不可用应用等。
- **完备的实时监控：**Sentinel 同时提供实时的监控功能。您可以在控制台中看到接入应用的单台机器秒级数据，甚至 500 台以下规模的集群的汇总运行情况。
- **广泛的开源生态：**Sentinel 提供开箱即用的与其他开源框架/库的整合模块，例如与 Spring Cloud、Dubbo、gRPC 整合只需要引入相应的依赖并进行简单的配置即可快速地接入 Sentinel。
- **完善的 SPI 扩展点：**Sentinel 提供简单易用、完善的 SPI 扩展接口。您可以通过实现扩展接口来快速地定制逻辑。例如定制规则管理、适配动态数据源等。

### 9.**Nacos配置中心详解**

![](https://pic-1304161434.cos.ap-guangzhou.myqcloud.com/img/202201271029709.png)

第一步，打开 Nacos 配置中心页面，点击右上角“+”号新建配置。

在新建配置页面包含六个选项：Data ID、Group、描述、说明、配置格式与配置内容，我们分别了解下这些选项的作用。

![](https://pic-1304161434.cos.ap-guangzhou.myqcloud.com/img/202201271029348.png)

- Data ID：配置的唯一标识，格式固定为：{微服务id}-{环境名}.yml，这里填写 order-service-dev.yml，其中 dev 就是环境名代表这个配置文件是 order-service 的开发环境配置文件。
- Group：指定配置文件的分组，这里设置默认分组 DEFAULT_GROUP 即可。
- 描述：说明 order-service-dev.yml 配置文件的用途。
- 配置格式：指定“配置内容”的类型，这里选择 YAML 即可。
- 配置内容：将 order-service 工程的 application.yml 文件内容粘贴过来。

之后点击右下角的“发布”按钮完成设置。

springcloud alibaba对于dubbo

1.springcloud更确切地说是开发的脚手架，而dubbo只是一个远程调用服务框架，没有可比性

2.使用远程调用的话只能谈到open Feign和dubbo

A：open Feign

openfeign是一个基于feign的远程通讯框架

B：dubbo

dubbo是apache公司开源的一个远程通讯框架，主要的实现技术是rpc