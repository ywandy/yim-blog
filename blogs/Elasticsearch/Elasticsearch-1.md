---
title: Elasticsearch-1
tags: 
  - Elasticsearch
date: 2023-03-30 09:00:00
categories:
  - Elasticsearch
---

原文链接：https://blog.csdn.net/ganquanzhong/article/details/108633025

ElasticSearch
倒排索引

Logstash
有用的 Logstash 链接

Logstash 频道博客文章
https://blog.csdn.net/ubuntutouch/category_9335275.html
如何安装 Elastic 栈中的 Logstash
https://blog.csdn.net/UbuntuTouch/article/details/99655350
Logstash：Logstash 入门教程 （一）
https://elasticstack.blog.csdn.net/article/details/105973985
Logstash：Logstash 入门教程 （二）
https://elasticstack.blog.csdn.net/article/details/105979677
Logstash：Data转换，分析，提取，丰富及核心操作
https://blog.csdn.net/UbuntuTouch/article/details/100770828
Logstash：把Apache日志导入到 Elasticsearch
https://blog.csdn.net/UbuntuTouch/article/details/100727051
Logstash: 启动监控及集中管理
https://blog.csdn.net/UbuntuTouch/article/details/103767088
Logstash 培训视频
https://www.elastic.co/cn/webinars/getting-started-logstash

![](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202303301720506.png)

![](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202303301720786.png)

设计方案
ES 订单数据的同步方案
MySQL数据同步到ES中，大致总结可以分为两种方案：
方案1：监听MySQL的Binlog，分析Binlog将数据同步到ES集群中。
方案2：直接通过ES API将数据写入到ES集群中。
考虑到订单系统ES服务的业务特殊性，对于订单数据的实时性较高，显然监听Binlog的方式相当于异步同步，有可能会产生较大的延时性。且方案1实质上跟方案2类似，但又引入了新的系统，维护成本也增高。所以订单中心ES采用了直接通过ES API写入订单数据的方式，该方式简洁灵活，能够很好的满足订单中心数据同步到ES的需求。

由于ES订单数据的同步采用的是在业务中写入的方式，当新建或更新文档发生异常时，如果重试势必会影响业务正常操作的响应时间。

所以每次业务操作只更新一次ES，如果发生错误或者异常，在数据库中插入一条补救任务，有Worker任务会实时地扫这些数据，以数据库订单数据为基准来再次更新ES数据。通过此种补偿机制，来保证ES数据与数据库订单数据的最终一致性。

```txt
搜索所有index（慎用）：
GET  /_search

搜一个索引下，所有type，（不指定type即可）
GET /beauties/_search

搜多个索引，则多个索引间，用逗号(,)分隔开
GET /beauties,test_index/_search

使用通配符，搜索所有匹配的index内容
GET /*e*,b*/_search

搜索一个index下，多个type
GET /beauties/my,cn/_search

搜索所有index下，指定type，（用 _all 代指所有index）
GET /_all/my,cn/_search

/_search：所有索引，所有type下的所有数据都搜索出来
/index1/_search：指定一个index，搜索其下所有type的数据
/index1,index2/_search：同时搜索两个index下的数据
/*1,*2/_search：按照通配符去匹配多个索引
/index1/type1/_search：搜索一个index下指定的type的数据
/index1/type1,type2/_search：可以搜索一个index下多个type的数据
/index1,index2/type1,type2/_search：搜索多个index下的多个type的数据
/_all/type1,type2/_search：_all，可以代表搜索所有index下的指定type的数据
```

## ElasticSearch（一）

### 学习目标：

能够理解ElasticSearch的作用
能够安装ElasticSearch服务
能够理解ElasticSearch的相关概念
能够使用Postman发送Restful请求操作ElasticSearch
能够理解分词器的作用
能够使用ElasticSearch集成IK分词器
能够完成es集群搭建

### 第一章 ElasticSearch简介

#### 1.1 什么是ElasticSearch

Elaticsearch，简称为es， es是一个开源的高扩展的分布式全文检索引擎，它可以近乎实时的存储、检索数据；本身扩展性很好，可以扩展到上百台服务器，处理PB级别的数据。es也使用Java开发并使用Lucene作为其核心来实现所有索引和搜索的功能，但是它的目的是通过简单的RESTful API来隐藏Lucene的复杂性，从而让全文搜索变得简单。

#### 1.2 ElasticSearch的使用案例

2013年初，GitHub抛弃了Solr，采取ElasticSearch 来做PB级的搜索。 “GitHub使用ElasticSearch搜索20TB的数据，包括13亿文件和1300亿行代码”
维基百科：启动以elasticsearch为基础的核心搜索架构
SoundCloud：“SoundCloud使用ElasticSearch为1.8亿用户提供即时而精准的音乐搜索服务”
百度：百度目前广泛使用ElasticSearch作为文本数据分析，采集百度所有服务器上的各类指标数据及用户自定义数据，通过对各种数据进行多维分析展示，辅助定位分析实例异常或业务层面异常。目前覆盖百度内部20多个业务线（包括casio、云分析、网盟、预测、文库、直达号、钱包、风控等），单集群最大100台机器，200个ES节点，每天导入30TB+数据
新浪使用ES 分析处理32亿条实时日志
阿里使用ES 构建挖财自己的日志采集和分析体系

#### 1.3 ElasticSearch对比Solr

Solr 利用 Zookeeper 进行分布式管理，而 Elasticsearch 自身带有分布式协调管理功能;
Solr 支持更多格式的数据，而 Elasticsearch 仅支持json文件格式；
Solr 官方提供的功能更多，而 Elasticsearch 本身更注重于核心功能，高级功能多有第三方插件提供；
Solr 在传统的搜索应用中表现好于 Elasticsearch，但在处理实时搜索应用时效率明显低于 Elasticsearch

### 第二章 ElasticSearch安装与启动

#### 2.1 下载ES压缩包

ElasticSearch分为Linux和Windows版本，基于我们主要学习的是ElasticSearch的Java客户端的使用，所以我们课程中使用的是安装较为简便的Window版本，项目上线后，公司的运维人员会安装Linux版的ES供我们连接使用。

ElasticSearch的官方地址： https://www.elastic.co/products/elasticsearch

在资料中已经提供了下载好的5.6.8的压缩包：
这里下载6.5版本，当然你也可以下载最新版本，注意新版本有许多新的特性！！

#### 2.2 安装ES服务

Windows版的ElasticSearch的安装很简单，类似Windows版的Tomcat，解压开即安装完毕，解压后的ElasticSearch的目录结构如下：

![](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202303311002558.png)

修改elasticsearch配置文件：config/elasticsearch.yml，增加以下两句命令：

```txt
http.cors.enabled: true
http.cors.allow-origin: "*"
```


此步为允许elasticsearch跨越访问，如果不安装后面的elasticsearch-head是可以不修改，直接启动。

#### 2.3 启动ES服务

点击ElasticSearch下的bin目录下的elasticsearch.bat启动，控制台显示的日志信息如下：

![](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202303311003118.png)



注意：9300是tcp通讯端口，集群间和TCPClient都执行该端口，9200是http协议的RESTful接口 。

通过浏览器访问ElasticSearch服务器，看到如下返回的json信息，代表服务启动成功：

![](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202303311021662.png)

注意：ElasticSearch是使用java开发的，且本版本的es需要的jdk版本要是1.8以上，所以安装ElasticSearch之前保证JDK1.8+安装完毕，并正确的配置好JDK环境变量，否则启动ElasticSearch失败。

#### 2.4 安装ES的图形化界面插件

ElasticSearch不同于Solr自带图形化界面，我们可以通过安装ElasticSearch的head插件，完成图形化界面的效果，完成索引数据的查看。安装插件的方式有两种，在线安装和本地安装。本文档采用本地安装方式进行head插件的安装。elasticsearch-5-*以上版本安装head需要安装node和grunt

1）下载head插件：https://github.com/mobz/elasticsearch-head

在资料中已经提供了elasticsearch-head-master插件压缩包：

2）将elasticsearch-head-master压缩包解压到任意目录，但是要和elasticsearch的安装目录区别开

3）下载nodejs：https://nodejs.org/en/download/

在资料中已经提供了nodejs安装程序：

双击安装程序，步骤截图如下：

安装完毕，可以通过cmd控制台输入：node -v 查看版本号

5）将grunt安装为全局命令 ，Grunt是基于Node.js的项目构建工具

在cmd控制台中输入如下执行命令：

```shell
npm install -g grunt-cli
```

6）进入elasticsearch-head-master目录启动head，在命令提示符下输入命令：

```shell
npm install
grunt server
```


7）打开浏览器，输入 http://localhost:9100，看到如下页面：

如果不能成功连接到es服务，需要修改ElasticSearch的config目录下的配置文件：config/elasticsearch.yml，增加以下两句命令：

```shell
http.cors.enabled: true
http.cors.allow-origin: "*"
```


然后重新启动ElasticSearch服务。

我这里使用的是谷歌浏览器插件

![](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202303311023474.png)

### 第三章 ElasticSearch相关概念(术语)

#### 3.1 概述

Elasticsearch是面向文档(document oriented)的，这意味着它可以存储整个对象或文档(document)。然而它不仅仅是存储，还会索引(index)每个文档的内容使之可以被搜索。在Elasticsearch中，你可以对文档（而非成行成列的数据）进行索引、搜索、排序、过滤。Elasticsearch比传统关系型数据库如下：

```txt
Relational DB -> Databases -> Tables -> Rows -> Columns
Elasticsearch -> Indices   -> Types  -> Documents -> Fields
```

#### 3.2 Elasticsearch核心概念

##### 3.2.1 索引 index

一个索引就是一个拥有几分相似特征的文档的集合。比如说，你可以有一个客户数据的索引，另一个产品目录的索引，还有一个订单数据的索引。一个索引由一个名字来标识（必须全部是小写字母的），并且当我们要对对应于这个索引中的文档进行索引、搜索、更新和删除的时候，都要使用到这个名字。在一个集群中，可以定义任意多的索引。

##### 3.2.2 类型 type

在一个索引中，你可以定义一种或多种类型。一个类型是你的索引的一个逻辑上的分类/分区，其语义完全由你来定。通常，会为具有一组共同字段的文档定义一个类型。比如说，我们假设你运营一个博客平台并且将你所有的数据存储到一个索引中。在这个索引中，你可以为用户数据定义一个类型，为博客数据定义另一个类型，当然，也可以为评论数据定义另一个类型。

##### 3.2.3 字段Field

相当于是数据表的字段，对文档数据根据不同属性进行的分类标识

##### 3.2.4 映射 mapping

mapping是处理数据的方式和规则方面做一些限制，如某个字段的数据类型、默认值、分析器、是否被索引等等，这些都是映射里面可以设置的，其它就是处理es里面数据的一些使用规则设置也叫做映射，按着最优规则处理数据对性能提高很大，因此才需要建立映射，并且需要思考如何建立映射才能对性能更好。

##### 3.2.5 文档 document

一个文档是一个可被索引的基础信息单元。比如，你可以拥有某一个客户的文档，某一个产品的一个文档，当然，也可以拥有某个订单的一个文档。文档以JSON（Javascript Object Notation）格式来表示，而JSON是一个到处存在的互联网数据交互格式。

在一个index/type里面，你可以存储任意多的文档。注意，尽管一个文档，物理上存在于一个索引之中，文档必须被索引/赋予一个索引的type。

##### 3.2.6 接近实时 NRT

Elasticsearch是一个接近实时的搜索平台。这意味着，从索引一个文档直到这个文档能够被搜索到有一个轻微的延迟（通常是1秒以内）

##### 3.2.7 集群 cluster

一个集群就是由一个或多个节点组织在一起，它们共同持有整个的数据，并一起提供索引和搜索功能。一个集群由一个唯一的名字标识，这个名字默认就是“elasticsearch”。这个名字是重要的，因为一个节点只能通过指定某个集群的名字，来加入这个集群

##### 3.2.8 节点 node

一个节点是集群中的一个服务器，作为集群的一部分，它存储数据，参与集群的索引和搜索功能。和集群类似，一个节点也是由一个名字来标识的，默认情况下，这个名字是一个随机的漫威漫画角色的名字，这个名字会在启动的时候赋予节点。这个名字对于管理工作来说挺重要的，因为在这个管理过程中，你会去确定网络中的哪些服务器对应于Elasticsearch集群中的哪些节点。

一个节点可以通过配置集群名称的方式来加入一个指定的集群。默认情况下，每个节点都会被安排加入到一个叫做“elasticsearch”的集群中，这意味着，如果你在你的网络中启动了若干个节点，并假定它们能够相互发现彼此，它们将会自动地形成并加入到一个叫做“elasticsearch”的集群中。

在一个集群里，只要你想，可以拥有任意多个节点。而且，如果当前你的网络中没有运行任何Elasticsearch节点，这时启动一个节点，会默认创建并加入一个叫做“elasticsearch”的集群。

##### 3.2.9 分片和复制 shards&replicas

一个索引可以存储超出单个结点硬件限制的大量数据。比如，一个具有10亿文档的索引占据1TB的磁盘空间，而任一节点都没有这样大的磁盘空间；或者单个节点处理搜索请求，响应太慢。为了解决这个问题，Elasticsearch提供了将索引划分成多份的能力，这些份就叫做分片。当你创建一个索引的时候，你可以指定你想要的分片的数量。每个分片本身也是一个功能完善并且独立的“索引”，这个“索引”可以被放置到集群中的任何节点上。分片很重要，主要有两方面的原因：
1）允许你水平分割/扩展你的内容容量。
2）允许你在分片（潜在地，位于多个节点上）之上进行分布式的、并行的操作，进而提高性能/吞吐量。

至于一个分片怎样分布，它的文档怎样聚合回搜索请求，是完全由Elasticsearch管理的，对于作为用户的你来说，这些都是透明的。

在一个网络/云的环境里，失败随时都可能发生，在某个分片/节点不知怎么的就处于离线状态，或者由于任何原因消失了，这种情况下，有一个故障转移机制是非常有用并且是强烈推荐的。为此目的，Elasticsearch允许你创建分片的一份或多份拷贝，这些拷贝叫做复制分片，或者直接叫复制。

复制之所以重要，有两个主要原因： 在分片/节点失败的情况下，提供了高可用性。因为这个原因，注意到复制分片从不与原/主要（original/primary）分片置于同一节点上是非常重要的。扩展你的搜索量/吞吐量，因为搜索可以在所有的复制上并行运行。总之，每个索引可以被分成多个分片。一个索引也可以被复制0次（意思是没有复制）或多次。一旦复制了，每个索引就有了主分片（作为复制源的原来的分片）和复制分片（主分片的拷贝）之别。分片和复制的数量可以在索引创建的时候指定。在索引创建之后，你可以在任何时候动态地改变复制的数量，但是你事后不能改变分片的数量。

默认情况下，Elasticsearch中的每个索引被分片5个主分片和1个复制，这意味着，如果你的集群中至少有两个节点，你的索引将会有5个主分片和另外5个复制分片（1个完全拷贝），这样的话每个索引总共就有10个分片。

### 第四章 ElasticSearch的客户端操作

实际开发中，主要有三种方式可以作为elasticsearch服务的客户端：

第一种，elasticsearch-head插件
第二种，使用elasticsearch提供的Restful接口直接访问
第三种，使用elasticsearch提供的API进行访问

#### 4.1 安装Postman工具

Postman中文版是postman这款强大网页调试工具的windows客户端，提供功能强大的Web API & HTTP 请求调试。软件功能非常强大，界面简洁明晰、操作方便快捷，设计得很人性化。Postman中文版能够发送任何类型的HTTP 请求 (GET, HEAD, POST, PUT…)，且可以附带任何数量的参数。

#### 4.1 下载Postman工具

Postman官网：https://www.getpostman.com

#### 4.2 注册Postman工具

使用Postman工具进行Restful接口访问

##### 4.2.1 ElasticSearch的接口语法

```shell
curl -X<VERB> '<PROTOCOL>://<HOST>:<PORT>/<PATH>?<QUERY_STRING>' -d '<BODY>'
```

其中：

| 参数         | 解释                                                         |
| ------------ | ------------------------------------------------------------ |
| VERB         | 适当的 HTTP 方法 或 谓词 : GET、 POST、 PUT、 HEAD 或者 DELETE。 |
| PROTOCOL     | http 或者 https（如果你在 Elasticsearch 前面有一个 https 代理） |
| HOST         | Elasticsearch 集群中任意节点的主机名，或者用 localhost 代表本地机器上的节点。 |
| PORT         | 运行 Elasticsearch HTTP 服务的端口号，默认是 9200 。         |
| PATH         | API 的终端路径（例如 _count 将返回集群中文档数量）。Path 可能包含多个组件，例如：_cluster/stats 和 _nodes/stats/jvm 。 |
| QUERY_STRING | 任意可选的查询字符串参数 (例如 ?pretty 将格式化地输出 JSON 返回值，使其更容易阅读) |
| BODY         | 一个 JSON 格式的请求体 (如果请求需要的话)                    |

##### 4.2.2 创建索引index和映射mapping

请求url：

```http
PUT		localhost:9200/blog1
```


请求体：

```json
{
    "mappings": {
        "article": {
            "properties": {
                "id": {
                	"type": "long",
                    "store": true,
                    "index":"not_analyzed"
                },
                "title": {
                	"type": "text",
                    "store": true,
                    "index":"analyzed",
                    "analyzer":"standard"
                },
                "content": {
                	"type": "text",
                    "store": true,
                    "index":"analyzed",
                    "analyzer":"standard"
                }
            }
        }
    }
}
```


elasticsearch-head查看：

##### 4.2.3 创建索引后设置Mapping

我们可以在创建索引时设置mapping信息，当然也可以先创建索引然后再设置mapping。

在上一个步骤中不设置maping信息，直接使用put方法创建一个索引，然后设置mapping信息。

请求的url：

```http
POST	http://127.0.0.1:9200/blog2/hello/_mapping
```

请求体：

```json
{
    "hello": {
            "properties": {
                "id":{
                	"type":"long",
                	"store":true
                },
                "title":{
                	"type":"text",
                	"store":true,
                	"index":true,
                	"analyzer":"standard"
                },
                "content":{
                	"type":"text",
                	"store":true,
                	"index":true,
                	"analyzer":"standard"
                }
            }
        }
  }
```

##### 4.2.4 删除索引index

请求url：

```http
DELETE		localhost:9200/blog1
```

##### 4.2.5 创建文档document

请求url：

```http
POST	localhost:9200/blog1/article/1
```

请求体：

```json
{
	"id":1,
	"title":"ElasticSearch是一个基于Lucene的搜索服务器",
	"content":"它提供了一个分布式多用户能力的全文搜索引擎，基于RESTful web接口。Elasticsearch是用Java开发的，并作为Apache许可条款下的开放源码发布，是当前流行的企业级搜索引擎。设计用于云计算中，能够达到实时搜索，稳定，可靠，快速，安装使用方便。"
}
```

##### 4.2.6 修改文档document

请求url：

```http
POST	localhost:9200/blog1/article/1
```


请求体：

```json
{
	"id":1,
	"title":"【修改】ElasticSearch是一个基于Lucene的搜索服务器",
	"content":"【修改】它提供了一个分布式多用户能力的全文搜索引擎，基于RESTful web接口。Elasticsearch是用Java开发的，并作为Apache许可条款下的开放源码发布，是当前流行的企业级搜索引擎。设计用于云计算中，能够达到实时搜索，稳定，可靠，快速，安装使用方便。"
}
```

#####  4.2.7 删除文档document

请求url：

```http
DELETE	localhost:9200/blog1/article/1
```

##### 4.2.8 查询文档-根据id查询

请求url：

```http
GET	localhost:9200/blog1/article/1
```

##### 4.2.9 查询文档-querystring查询

请求url：

```http
POST	localhost:9200/blog1/article/_search
```


请求体：

```json
{
    "query": {
        "query_string": {
            "default_field": "title",
            "query": "搜索服务器"
        }
    }
}
```


注意：

将搜索内容"搜索服务器"修改为"钢索"，同样也能搜索到文档，该原因会在下面讲解中得到答案

```json
{
    "query": {
        "query_string": {
            "default_field": "title",
            "query": "钢索"
        }
    }
}
```

##### 4.2.10 查询文档-term查询

请求url：

```http
POST	localhost:9200/blog1/article/_search
```

请求体：

```json
{
    "query": {
        "term": {
            "title": "搜索"
        }
    }
}
```

### 第五章 IK 分词器和ElasticSearch集成使用

#### 5.1 上述查询存在问题分析

在进行字符串查询时，我们发现去搜索"搜索服务器"和"钢索"都可以搜索到数据；

而在进行词条查询时，我们搜索"搜索"却没有搜索到数据；

究其原因是ElasticSearch的标准分词器导致的，当我们创建索引时，字段使用的是标准分词器：

```json
{
    "mappings": {
        "article": {
            "properties": {
                "id": {
                	"type": "long",
                    "store": true,
                    "index":"not_analyzed"
                },
                "title": {
                	"type": "text",
                    "store": true,
                    "index":"analyzed",
                    "analyzer":"standard"	//标准分词器
                },
                "content": {
                	"type": "text",
                    "store": true,
                    "index":"analyzed",
                    "analyzer":"standard"	//标准分词器
                }
            }
        }
    }
}
```


例如对 “我是程序员” 进行分词

标准分词器分词效果测试：

```url
http://127.0.0.1:9200/_analyze?analyzer=standard&pretty=true&text=我是程序员
```

分词结果：

```json
{
    "tokens" : [
    {
      "token" : "我",
      "start_offset" : 0,
      "end_offset" : 1,
      "type" : "<IDEOGRAPHIC>",
      "position" : 0
    },
    {
      "token" : "是",
      "start_offset" : 1,
      "end_offset" : 2,
      "type" : "<IDEOGRAPHIC>",
      "position" : 1
    },
    {
      "token" : "程",
      "start_offset" : 2,
      "end_offset" : 3,
      "type" : "<IDEOGRAPHIC>",
      "position" : 2
    },
    {
      "token" : "序",
      "start_offset" : 3,
      "end_offset" : 4,
      "type" : "<IDEOGRAPHIC>",
      "position" : 3
    },
    {
      "token" : "员",
      "start_offset" : 4,
      "end_offset" : 5,
      "type" : "<IDEOGRAPHIC>",
      "position" : 4
    }
  ]
}
```



而我们需要的分词效果是：我、是、程序、程序员

这样的话就需要对中文支持良好的分析器的支持，支持中文分词的分词器有很多，word分词器、庖丁解牛、盘古分词、Ansj分词等，但我们常用的还是下面要介绍的IK分词器。

#### 5.2 IK分词器简介

IKAnalyzer是一个开源的，基于java语言开发的轻量级的中文分词工具包。从2006年12月推出1.0版开始，IKAnalyzer已经推出 了3个大版本。最初，它是以开源项目Lucene为应用主体的，结合词典分词和文法分析算法的中文分词组件。新版本的IKAnalyzer3.0则发展为 面向Java的公用分词组件，独立于Lucene项目，同时提供了对Lucene的默认优化实现。

IK分词器3.0的特性如下：

1）采用了特有的“正向迭代最细粒度切分算法“，具有60万字/秒的高速处理能力。
2）采用了多子处理器分析模式，支持：英文字母（IP地址、Email、URL）、数字（日期，常用中文数量词，罗马数字，科学计数法），中文词汇（姓名、地名处理）等分词处理。
3）对中英联合支持不是很好,在这方面的处理比较麻烦.需再做一次查询,同时是支持个人词条的优化的词典存储，更小的内存占用。
4）支持用户词典扩展定义。
5）针对Lucene全文检索优化的查询分析器IKQueryParser；采用歧义分析算法优化查询关键字的搜索排列组合，能极大的提高Lucene检索的命中率。

#### 5.3 ElasticSearch集成IK分词器

##### 5.3.1 IK分词器的安装

1）下载地址：https://github.com/medcl/elasticsearch-analysis-ik/releases

课程资料也提供了IK分词器的压缩包：

2）解压，将解压后的elasticsearch文件夹拷贝到elasticsearch-5.6.8\plugins下，并重命名文件夹为analysis-ik

3）重新启动ElasticSearch，即可加载IK分词器

##### 5.3.2 IK分词器测试

IK提供了两个分词算法ik_smart 和 ik_max_word

其中 ik_smart 为最少切分，ik_max_word为最细粒度划分

我们分别来试一下

1）最小切分：在浏览器地址栏输入地址

```url
http://127.0.0.1:9200/_analyze?analyzer=ik_smart&pretty=true&text=我是程序员
```

输出的结果为：

```json
{
  "tokens" : [
    {
      "token" : "我",
      "start_offset" : 0,
      "end_offset" : 1,
      "type" : "CN_CHAR",
      "position" : 0
    },
    {
      "token" : "是",
      "start_offset" : 1,
      "end_offset" : 2,
      "type" : "CN_CHAR",
      "position" : 1
    },
    {
      "token" : "程序员",
      "start_offset" : 2,
      "end_offset" : 5,
      "type" : "CN_WORD",
      "position" : 2
    }
  ]
}
```

2）最细切分：在浏览器地址栏输入地址

```url
http://127.0.0.1:9200/_analyze?analyzer=ik_max_word&pretty=true&text=我是程序员
```

输出的结果为：

```json
{
  "tokens" : [
    {
      "token" : "我",
      "start_offset" : 0,
      "end_offset" : 1,
      "type" : "CN_CHAR",
      "position" : 0
    },
    {
      "token" : "是",
      "start_offset" : 1,
      "end_offset" : 2,
      "type" : "CN_CHAR",
      "position" : 1
    },
    {
      "token" : "程序员",
      "start_offset" : 2,
      "end_offset" : 5,
      "type" : "CN_WORD",
      "position" : 2
    },
    {
      "token" : "程序",
      "start_offset" : 2,
      "end_offset" : 4,
      "type" : "CN_WORD",
      "position" : 3
    },
    {
      "token" : "员",
      "start_offset" : 4,
      "end_offset" : 5,
      "type" : "CN_CHAR",
      "position" : 4
    }
  ]
}
```

#### 5.4 修改索引映射mapping

##### 5.4.1 重建索引

删除原有blog1索引

DELETE		localhost:9200/blog1
1
创建blog1索引，此时分词器使用ik_max_word

PUT		localhost:9200/blog1
1
{
    "mappings": {
        "article": {
            "properties": {
                "id": {
                	"type": "long",
                    "store": true,
                    "index":"not_analyzed"
                },
                "title": {
                	"type": "text",
                    "store": true,
                    "index":"analyzed",
                    "analyzer":"ik_max_word"
                },
                "content": {
                	"type": "text",
                    "store": true,
                    "index":"analyzed",
                    "analyzer":"ik_max_word"
                }
            }
        }
    }
}
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
创建文档

POST	localhost:9200/blog1/article/1
1
{
	"id":1,
	"title":"ElasticSearch是一个基于Lucene的搜索服务器",
	"content":"它提供了一个分布式多用户能力的全文搜索引擎，基于RESTful web接口。Elasticsearch是用Java开发的，并作为Apache许可条款下的开放源码发布，是当前流行的企业级搜索引擎。设计用于云计算中，能够达到实时搜索，稳定，可靠，快速，安装使用方便。"
}
1
2
3
4
5

##### 5.4.2 再次测试queryString查询

请求url：

POST	localhost:9200/blog1/article/_search
1
请求体：

{
    "query": {
        "query_string": {
            "default_field": "title",
            "query": "搜索服务器"
        }
    }
}
1
2
3
4
5
6
7
8
postman截图：

[外链图片转存失败,源站可能有防盗链机制,建议将图片保存下来直接上传(img-cbxx7eG9-1600272635312)(image/57.png)]

将请求体搜索字符串修改为"钢索"，再次查询：

{
    "query": {
        "query_string": {
            "default_field": "title",
            "query": "钢索"
        }
    }
}
1
2
3
4
5
6
7
8
postman截图：

[外链图片转存失败,源站可能有防盗链机制,建议将图片保存下来直接上传(img-jeapt0jR-1600272635313)(image\59.png)]

##### 5.4.3 再次测试term测试

请求url：

POST	localhost:9200/blog1/article/_search
1
请求体：

{
    "query": {
        "term": {
            "title": "搜索"
        }
    }
}
1
2
3
4
5
6
7
postman截图：

[外链图片转存失败,源站可能有防盗链机制,建议将图片保存下来直接上传(img-IzbRCXdE-1600272635314)(image\60.png)]

第六章 ElasticSearch集群
 ES集群是一个 P2P类型(使用 gossip 协议)的分布式系统，除了集群状态管理以外，其他所有的请求都可以发送到集群内任意一台节点上，这个节点可以自己找到需要转发给哪些节点，并且直接跟这些节点通信。所以，从网络架构及服务配置上来说，构建集群所需要的配置极其简单。在 Elasticsearch 2.0 之前，无阻碍的网络下，所有配置了相同 cluster.name 的节点都自动归属到一个集群中。2.0 版本之后，基于安全的考虑避免开发环境过于随便造成的麻烦，从 2.0 版本开始，默认的自动发现方式改为了单播(unicast)方式。配置里提供几台节点的地址，ES 将其视作 gossip router 角色，借以完成集群的发现。由于这只是 ES 内一个很小的功能，所以 gossip router 角色并不需要单独配置，每个 ES 节点都可以担任。所以，采用单播方式的集群，各节点都配置相同的几个节点列表作为 router 即可。

 集群中节点数量没有限制，一般大于等于2个节点就可以看做是集群了。一般处于高性能及高可用方面来考虑一般集群中的节点数量都是3个及3个以上。

#### 6.1 集群的相关概念

##### 6.1.1 集群 cluster

一个集群就是由一个或多个节点组织在一起，它们共同持有整个的数据，并一起提供索引和搜索功能。一个集群由一个唯一的名字标识，这个名字默认就是“elasticsearch”。这个名字是重要的，因为一个节点只能通过指定某个集群的名字，来加入这个集群

##### 6.1.2 节点 node

一个节点是集群中的一个服务器，作为集群的一部分，它存储数据，参与集群的索引和搜索功能。和集群类似，一个节点也是由一个名字来标识的，默认情况下，这个名字是一个随机的漫威漫画角色的名字，这个名字会在启动的时候赋予节点。这个名字对于管理工作来说挺重要的，因为在这个管理过程中，你会去确定网络中的哪些服务器对应于Elasticsearch集群中的哪些节点。

一个节点可以通过配置集群名称的方式来加入一个指定的集群。默认情况下，每个节点都会被安排加入到一个叫做“elasticsearch”的集群中，这意味着，如果你在你的网络中启动了若干个节点，并假定它们能够相互发现彼此，它们将会自动地形成并加入到一个叫做“elasticsearch”的集群中。

在一个集群里，只要你想，可以拥有任意多个节点。而且，如果当前你的网络中没有运行任何Elasticsearch节点，这时启动一个节点，会默认创建并加入一个叫做“elasticsearch”的集群。

6.1.3 分片和复制 shards&replicas
一个索引可以存储超出单个结点硬件限制的大量数据。比如，一个具有10亿文档的索引占据1TB的磁盘空间，而任一节点都没有这样大的磁盘空间；或者单个节点处理搜索请求，响应太慢。为了解决这个问题，Elasticsearch提供了将索引划分成多份的能力，这些份就叫做分片。当你创建一个索引的时候，你可以指定你想要的分片的数量。每个分片本身也是一个功能完善并且独立的“索引”，这个“索引”可以被放置到集群中的任何节点上。分片很重要，主要有两方面的原因：
1）允许你水平分割/扩展你的内容容量。
2）允许你在分片（潜在地，位于多个节点上）之上进行分布式的、并行的操作，进而提高性能/吞吐量。

至于一个分片怎样分布，它的文档怎样聚合回搜索请求，是完全由Elasticsearch管理的，对于作为用户的你来说，这些都是透明的。

在一个网络/云的环境里，失败随时都可能发生，在某个分片/节点不知怎么的就处于离线状态，或者由于任何原因消失了，这种情况下，有一个故障转移机制是非常有用并且是强烈推荐的。为此目的，Elasticsearch允许你创建分片的一份或多份拷贝，这些拷贝叫做复制分片，或者直接叫复制。

复制之所以重要，有两个主要原因： 在分片/节点失败的情况下，提供了高可用性。因为这个原因，注意到复制分片从不与原/主要（original/primary）分片置于同一节点上是非常重要的。扩展你的搜索量/吞吐量，因为搜索可以在所有的复制上并行运行。总之，每个索引可以被分成多个分片。一个索引也可以被复制0次（意思是没有复制）或多次。一旦复制了，每个索引就有了主分片（作为复制源的原来的分片）和复制分片（主分片的拷贝）之别。分片和复制的数量可以在索引创建的时候指定。在索引创建之后，你可以在任何时候动态地改变复制的数量，但是你事后不能改变分片的数量。

默认情况下，Elasticsearch中的每个索引被分片5个主分片和1个复制，这意味着，如果你的集群中至少有两个节点，你的索引将会有5个主分片和另外5个复制分片（1个完全拷贝），这样的话每个索引总共就有10个分片。

#### 6.2 集群的搭建

##### 6.2.1 准备三台elasticsearch服务器

创建elasticsearch-cluster文件夹，在内部复制三个elasticsearch服务

##### 6.2.2 修改每台服务器配置

修改elasticsearch-cluster\node*\config\elasticsearch.yml配置文件

node1节点：
#节点1的配置信息：
#集群名称，保证唯一
cluster.name: my-elasticsearch
#节点名称，必须不一样
node.name: node-1
#必须为本机的ip地址
network.host: 127.0.0.1
#服务端口号，在同一机器下必须不一样
http.port: 9200
#集群间通信端口号，在同一机器下必须不一样
transport.tcp.port: 9300
#设置集群自动发现机器ip集合
discovery.zen.ping.unicast.hosts: ["127.0.0.1:9300","127.0.0.1:9301","127.0.0.1:9302"]
1
2
3
4
5
6
7
8
9
10
11
12
13
node2节点：
#节点2的配置信息：
#集群名称，保证唯一
cluster.name: my-elasticsearch
#节点名称，必须不一样
node.name: node-2
#必须为本机的ip地址
network.host: 127.0.0.1
#服务端口号，在同一机器下必须不一样
http.port: 9201
#集群间通信端口号，在同一机器下必须不一样
transport.tcp.port: 9301
#设置集群自动发现机器ip集合
discovery.zen.ping.unicast.hosts: ["127.0.0.1:9300","127.0.0.1:9301","127.0.0.1:9302"]
1
2
3
4
5
6
7
8
9
10
11
12
13
node3节点：
#节点3的配置信息：
#集群名称，保证唯一
cluster.name: my-elasticsearch
#节点名称，必须不一样
node.name: node-3
#必须为本机的ip地址
network.host: 127.0.0.1
#服务端口号，在同一机器下必须不一样
http.port: 9202
#集群间通信端口号，在同一机器下必须不一样
transport.tcp.port: 9302
#设置集群自动发现机器ip集合
discovery.zen.ping.unicast.hosts: ["127.0.0.1:9300","127.0.0.1:9301","127.0.0.1:9302"]
1
2
3
4
5
6
7
8
9
10
11
12
13

##### 6.2.3 启动各个节点服务器

双击elasticsearch-cluster\node*\bin\elasticsearch.bat

启动节点1：
[外链图片转存失败,源站可能有防盗链机制,建议将图片保存下来直接上传(img-fGBaIS52-1600272635315)(image\21.png)]

启动节点2：
[外链图片转存失败,源站可能有防盗链机制,建议将图片保存下来直接上传(img-m9iBOBQR-1600272635316)(image\22.png)]

启动节点3：
[外链图片转存失败,源站可能有防盗链机制,建议将图片保存下来直接上传(img-5ZR8suZp-1600272635318)(image\23.png)]

##### 6.2.4 集群测试

添加索引和映射
PUT		localhost:9200/blog1
1
{
    "mappings": {
        "article": {
            "properties": {
                "id": {
                	"type": "long",
                    "store": true,
                    "index":"not_analyzed"
                },
                "title": {
                	"type": "text",
                    "store": true,
                    "index":"analyzed",
                    "analyzer":"standard"
                },
                "content": {
                	"type": "text",
                    "store": true,
                    "index":"analyzed",
                    "analyzer":"standard"
                }
            }
        }
    }
}
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
添加文档
POST	localhost:9200/blog1/article/1
1
{
	"id":1,
	"title":"ElasticSearch是一个基于Lucene的搜索服务器",
	"content":"它提供了一个分布式多用户能力的全文搜索引擎，基于RESTful web接口。Elasticsearch是用Java开发的，并作为Apache许可条款下的开放源码发布，是当前流行的企业级搜索引擎。设计用于云计算中，能够达到实时搜索，稳定，可靠，快速，安装使用方便。"
}
1
2
3
4
5

#### 使用elasticsearch-header查看集群情况

[外链图片转存失败,源站可能有防盗链机制,建议将图片保存下来直接上传(img-0JdmVaRB-1600272635319)(image\62.png)]

```txt
一、使用Java客户端管理ES
1、创建索引库
步骤：
	1）创建一个Java工程
	2）添加jar包，添加maven的坐标
	3）编写测试方法实现创建索引库
		1、创建一个Settings对象，相当于是一个配置信息。主要配置集群的名称。
		2、创建一个客户端Client对象
		3、使用client对象创建一个索引库
		4、关闭client对象
2、使用Java客户端设置Mappings
	步骤：
	1）创建一个Settings对象
	2）创建一个Client对象
	3）创建一个mapping信息，应该是一个json数据，可以是字符串，也可以是XContextBuilder对象
	4）使用client向es服务器发送mapping信息
	5）关闭client对象
3、添加文档
	步骤：
	1）创建一个Settings对象
	2）创建一个Client对象
	3）创建一个文档对象，创建一个json格式的字符串，或者使用XContentBuilder
	4）使用Client对象吧文档添加到索引库中
	5）关闭client
4、添加文档第二种方式
	创建一个pojo类
	使用工具类把pojo转换成json字符串
	把文档写入索引库

二、使用es客户端实现搜索
1、根据id搜索
	QueryBuilder queryBuilder = QueryBuilders.idsQuery().addIds("1", "2");
2、根据Term查询（关键词）
	QueryBuilder queryBuilder = QueryBuilders.termQuery("title", "北方");
3、QueryString查询方式（带分析的查询）
	QueryBuilder queryBuilder = QueryBuilders.queryStringQuery("速度与激情")
                .defaultField("title");
查询步骤：
	1）创建一个Client对象
	2）创建一个查询对象，可以使用QueryBuilders工具类创建QueryBuilder对象。
	3）使用client执行查询
	4）得到查询的结果。
	5）取查询结果的总记录数
	6）取查询结果列表
	7）关闭client
4、分页的处理
	在client对象执行查询之前，设置分页信息。
	然后再执行查询
	 //执行查询
        SearchResponse searchResponse = client.prepareSearch("index_hello")
                .setTypes("article")
                .setQuery(queryBuilder)
                //设置分页信息
                .setFrom(0)
                //每页显示的行数
                .setSize(5)
                .get();
	分页需要设置两个值，一个from、size
	from：起始的行号，从0开始。
	size：每页显示的记录数
5、查询结果高亮显示
（1）高亮的配置
	1）设置高亮显示的字段
	2）设置高亮显示的前缀
	3）设置高亮显示的后缀
（2）在client对象执行查询之前，设置高亮显示的信息。
（3）遍历结果列表时可以从结果中取高亮结果。
三、SpringDataElasticSearch
1、工程搭建
	1）创建一个java工程。
	2）把相关jar包添加到工程中。如果maven工程就添加坐标。
	3）创建一个spring的配置文件
		1、配置elasticsearch:transport-client
		2、elasticsearch:repositories：包扫描器，扫描dao
		3、配置elasticsearchTemplate对象，就是一个bean
2、管理索引库
	1、创建一个Entity类，其实就是一个JavaBean（pojo）映射到一个Document上
		需要添加一些注解进行标注。
	2、创建一个Dao，是一个接口，需要继承ElasticSearchRepository接口。
	3、编写测试代码。


3、创建索引
	直接使用ElasticsearchTemplate对象的createIndex方法创建索引，并配置映射关系。
4、添加、更新文档
	1）创建一个Article对象
	2）使用ArticleRepository对象向索引库中添加文档。
5、删除文档
	直接使用ArticleRepository对象的deleteById方法直接删除。
6、查询索引库
	直接使用ArticleRepository对象的查询方法。
7、自定义查询方法
	需要根据SpringDataES的命名规则来命名。
	如果不设置分页信息，默认带分页，每页显示10条数据。
	如果设置分页信息，应该在方法中添加一个参数Pageable
		Pageable pageable = PageRequest.of(0, 15);
	注意：设置分页信息，默认是从0页开始。
	

	可以对搜索的内容先分词然后再进行查询。每个词之间都是and的关系。

8、使用原生的查询条件查询
	NativeSearchQuery对象。
	使用方法：
		1）创建一个NativeSearchQuery对象
			设置查询条件，QueryBuilder对象
		2）使用ElasticSearchTemplate对象执行查询
		3）取查询结果
```



ElasticSearch（二）
学习目标：
能够使用java客户端完成创建、删除索引的操作
能够使用java客户端完成文档的增删改的操作
能够使用java客户端完成文档的查询操作
能够完成文档的分页操作
能够完成文档的高亮查询操作
能够搭建Spring Data ElasticSearch的环境
能够完成Spring Data ElasticSearch的基本增删改查操作
能够掌握基本条件查询的方法命名规则
第一章 ElasticSearch编程操作
1.1 创建工程，导入坐标
pom.xml坐标

<dependencies>
    <dependency>
        <groupId>org.elasticsearch</groupId>
        <artifactId>elasticsearch</artifactId>
        <version>5.6.8</version>
    </dependency>
    <dependency>
        <groupId>org.elasticsearch.client</groupId>
        <artifactId>transport</artifactId>
        <version>5.6.8</version>
    </dependency>
    <dependency>
        <groupId>org.apache.logging.log4j</groupId>
        <artifactId>log4j-to-slf4j</artifactId>
        <version>2.9.1</version>
    </dependency>
    <dependency>
        <groupId>org.slf4j</groupId>
        <artifactId>slf4j-api</artifactId>
        <version>1.7.24</version>
    </dependency>
    <dependency>
        <groupId>org.slf4j</groupId>
        <artifactId>slf4j-simple</artifactId>
        <version>1.7.21</version>
    </dependency>
    <dependency>
        <groupId>log4j</groupId>
        <artifactId>log4j</artifactId>
        <version>1.2.12</version>
    </dependency>
    <dependency>
        <groupId>junit</groupId>
        <artifactId>junit</artifactId>
        <version>4.12</version>
    </dependency>
</dependencies>
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
36
37
1.2 创建索引index
@Test
//创建索引
public void test1() throws Exception{
    // 创建Client连接对象
    Settings settings = Settings.builder().put("cluster.name", "my-elasticsearch").build();
    TransportClient client = new PreBuiltTransportClient(settings)
        .addTransportAddress(new InetSocketTransportAddress(InetAddress.getByName("127.0.0.1"), 9300));
    //创建名称为blog2的索引
    client.admin().indices().prepareCreate("blog2").get();
    //释放资源
    client.close();
}
1
2
3
4
5
6
7
8
9
10
11
12



1.3 创建映射mapping
@Test
//创建映射
public void test3() throws Exception{
    // 创建Client连接对象
    Settings settings = Settings.builder().put("cluster.name", "my-elasticsearch").build();
    TransportClient client = new PreBuiltTransportClient(settings)
        .addTransportAddress(new InetSocketTransportAddress(InetAddress.getByName("127.0.0.1"), 9300));
    
    // 添加映射
    /**
         * 格式：
         * "mappings" : {
             "article" : {
                "dynamic" : "false",
                 "properties" : {
                    "id" : { "type" : "string" },
                     "content" : { "type" : "string" },
                    "author" : { "type" : "string" }
                 }
            }
         }
         */
    XContentBuilder builder = XContentFactory.jsonBuilder()
        .startObject()
        .startObject("article")
        .startObject("properties")
        .startObject("id")
        .field("type", "integer").field("store", "yes")
        .endObject()
        .startObject("title")
        .field("type", "string").field("store", "yes").field("analyzer", "ik_smart")
        .endObject()
        .startObject("content")
        .field("type", "string").field("store", "yes").field("analyzer", "ik_smart")
        .endObject()
        .endObject()
        .endObject()
        .endObject();
    // 创建映射
    PutMappingRequest mapping = Requests.putMappingRequest("blog2")
        .type("article").source(builder);
    client.admin().indices().putMapping(mapping).get();
    //释放资源
    client.close();
}

1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
36
37
38
39
40
41
42
43
44
45



1.4 建立文档document
1.4.1 建立文档（通过XContentBuilder）
@Test
//创建文档(通过XContentBuilder)
public void test4() throws Exception{
    // 创建Client连接对象
    Settings settings = Settings.builder().put("cluster.name", "my-elasticsearch").build();
    TransportClient client = new PreBuiltTransportClient(settings)
        .addTransportAddress(new InetSocketTransportAddress(InetAddress.getByName("127.0.0.1"), 9300));

    //创建文档信息
    XContentBuilder builder = XContentFactory.jsonBuilder()
        .startObject()
        .field("id", 1)
        .field("title", "ElasticSearch是一个基于Lucene的搜索服务器")
        .field("content",
               "它提供了一个分布式多用户能力的全文搜索引擎，基于RESTful web接口。Elasticsearch是用Java开发的，并作为Apache许可条款下的开放源码发布，是当前流行的企业级搜索引擎。设计用于云计算中，能够达到实时搜索，稳定，可靠，快速，安装使用方便。")
        .endObject();
    
    // 建立文档对象
    /**
         * 参数一blog1：表示索引对象
         * 参数二article：类型
         * 参数三1：建立id
         */
    client.prepareIndex("blog2", "article", "1").setSource(builder).get();
    
    //释放资源
    client.close();
}

1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28


1.4.2 建立文档（使用Jackson转换实体）
1）创建Article实体

public class Article {
	private Integer id;
	private String title;
	private String content;
    getter/setter...
}
1
2
3
4
5
6
2）添加jackson坐标

<dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-core</artifactId>
    <version>2.8.1</version>
</dependency>
<dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-databind</artifactId>
    <version>2.8.1</version>
</dependency>
<dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-annotations</artifactId>
    <version>2.8.1</version>
</dependency>
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
3）代码实现

@Test
//创建文档(通过实体转json)
public void test5() throws Exception{
    // 创建Client连接对象
    Settings settings = Settings.builder().put("cluster.name", "my-elasticsearch").build();
    TransportClient client = new PreBuiltTransportClient(settings)
        .addTransportAddress(new InetSocketTransportAddress(InetAddress.getByName("127.0.0.1"), 9300));

    // 描述json 数据
    //{id:xxx, title:xxx, content:xxx}
    Article article = new Article();
    article.setId(2);
    article.setTitle("搜索工作其实很快乐");
    article.setContent("我们希望我们的搜索解决方案要快，我们希望有一个零配置和一个完全免费的搜索模式，我们希望能够简单地使用JSON通过HTTP的索引数据，我们希望我们的搜索服务器始终可用，我们希望能够一台开始并扩展到数百，我们要实时搜索，我们要简单的多租户，我们希望建立一个云的解决方案。Elasticsearch旨在解决所有这些问题和更多的问题。");
    
    ObjectMapper objectMapper = new ObjectMapper();
    
    // 建立文档
    client.prepareIndex("blog2", "article", article.getId().toString())
        //.setSource(objectMapper.writeValueAsString(article)).get();
        .setSource(objectMapper.writeValueAsString(article).getBytes(), XContentType.JSON).get();
    
    //释放资源
    client.close();
}
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25


1.5 查询文档操作
1.5.1关键词查询
@Test
public void testTermQuery() throws Exception{
    //1、创建es客户端连接对象
    Settings settings = Settings.builder().put("cluster.name", "my-elasticsearch").build();
    TransportClient client = new PreBuiltTransportClient(settings)
        .addTransportAddress(new InetSocketTransportAddress(InetAddress.getByName("127.0.0.1"), 9300));

    //2、设置搜索条件
    SearchResponse searchResponse = client.prepareSearch("blog2")
        .setTypes("article")
        .setQuery(QueryBuilders.termQuery("content", "搜索")).get();
    
    //3、遍历搜索结果数据
    SearchHits hits = searchResponse.getHits(); // 获取命中次数，查询结果有多少对象
    System.out.println("查询结果有：" + hits.getTotalHits() + "条");
    Iterator<SearchHit> iterator = hits.iterator();
    while (iterator.hasNext()) {
        SearchHit searchHit = iterator.next(); // 每个查询对象
        System.out.println(searchHit.getSourceAsString()); // 获取字符串格式打印
        System.out.println("title:" + searchHit.getSource().get("title"));
    }
    
    //4、释放资源
    client.close();

}
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
2.5.2 字符串查询
@Test
public void testStringQuery() throws Exception{
    //1、创建es客户端连接对象
    Settings settings = Settings.builder().put("cluster.name", "my-elasticsearch").build();
    TransportClient client = new PreBuiltTransportClient(settings)
        .addTransportAddress(new InetSocketTransportAddress(InetAddress.getByName("127.0.0.1"), 9300));

    //2、设置搜索条件
    SearchResponse searchResponse = client.prepareSearch("blog2")
        .setTypes("article")
        .setQuery(QueryBuilders.queryStringQuery("搜索")).get();
    
    //3、遍历搜索结果数据
    SearchHits hits = searchResponse.getHits(); // 获取命中次数，查询结果有多少对象
    System.out.println("查询结果有：" + hits.getTotalHits() + "条");
    Iterator<SearchHit> iterator = hits.iterator();
    while (iterator.hasNext()) {
        SearchHit searchHit = iterator.next(); // 每个查询对象
        System.out.println(searchHit.getSourceAsString()); // 获取字符串格式打印
        System.out.println("title:" + searchHit.getSource().get("title"));
    }
    
    //4、释放资源
    client.close();

}

1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
2.5.2 使用文档ID查询文档
@Test
    public void testIdQuery() throws Exception {
        //client对象为TransportClient对象
        SearchResponse response = client.prepareSearch("blog1")
                .setTypes("article")
                //设置要查询的id
                .setQuery(QueryBuilders.idsQuery().addIds("test002"))
                //执行查询
                .get();
        //取查询结果
        SearchHits searchHits = response.getHits();
        //取查询结果总记录数
        System.out.println(searchHits.getTotalHits());
        Iterator<SearchHit> hitIterator = searchHits.iterator();
        while(hitIterator.hasNext()) {
            SearchHit searchHit = hitIterator.next();
            //打印整行数据
            System.out.println(searchHit.getSourceAsString());
        }
    }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
2.6 查询文档分页操作
2.6.1 批量插入数据
 @Test
//批量插入100条数据
public void test9() throws Exception{
    	// 创建Client连接对象
        Settings settings = Settings.builder().put("cluster.name", "my-elasticsearch").build();
        TransportClient client = new PreBuiltTransportClient(settings)
                .addTransportAddress(new InetSocketTransportAddress(InetAddress.getByName("127.0.0.1"), 9300));

        ObjectMapper objectMapper = new ObjectMapper();
    
        for (int i = 1; i <= 100; i++) {
            // 描述json 数据
            Article article = new Article();
            article.setId(i);
            article.setTitle(i + "搜索工作其实很快乐");
            article.setContent(i
                    + "我们希望我们的搜索解决方案要快，我们希望有一个零配置和一个完全免费的搜索模式，我们希望能够简单地使用JSON通过HTTP的索引数据，我们希望我们的搜索服务器始终可用，我们希望能够一台开始并扩展到数百，我们要实时搜索，我们要简单的多租户，我们希望建立一个云的解决方案。Elasticsearch旨在解决所有这些问题和更多的问题。");
    
            // 建立文档
            client.prepareIndex("blog2", "article", article.getId().toString())
                    //.setSource(objectMapper.writeValueAsString(article)).get();
                    .setSource(objectMapper.writeValueAsString(article).getBytes(),XContentType.JSON).get();
        }
    
        //释放资源
        client.close();
}
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27


2.6.2 分页查询
@Test
//分页查询
public void test10() throws Exception{
    // 创建Client连接对象
    Settings settings = Settings.builder().put("cluster.name", "my-elasticsearch").build();
    TransportClient client = new PreBuiltTransportClient(settings)
        .addTransportAddress(new InetSocketTransportAddress(InetAddress.getByName("127.0.0.1"), 9300));

    // 搜索数据
    SearchRequestBuilder searchRequestBuilder = client.prepareSearch("blog2").setTypes("article")
        .setQuery(QueryBuilders.matchAllQuery());//默认每页10条记录
    
    // 查询第2页数据，每页20条
    //setFrom()：从第几条开始检索，默认是0。
    //setSize():每页最多显示的记录数。
    searchRequestBuilder.setFrom(0).setSize(5);
    SearchResponse searchResponse = searchRequestBuilder.get();
    
    SearchHits hits = searchResponse.getHits(); // 获取命中次数，查询结果有多少对象
    System.out.println("查询结果有：" + hits.getTotalHits() + "条");
    Iterator<SearchHit> iterator = hits.iterator();
    while (iterator.hasNext()) {
        SearchHit searchHit = iterator.next(); // 每个查询对象
        System.out.println(searchHit.getSourceAsString()); // 获取字符串格式打印
        System.out.println("id:" + searchHit.getSource().get("id"));
        System.out.println("title:" + searchHit.getSource().get("title"));
        System.out.println("content:" + searchHit.getSource().get("content"));
        System.out.println("-----------------------------------------");
    }
    
    //释放资源
    client.close();
}
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
[外链图片转存失败,源站可能有防盗链机制,建议将图片保存下来直接上传(img-9qAkQsOk-1600271923061)(image/13.png)]

2.7 查询结果高亮操作
2.7.1 什么是高亮显示
在进行关键字搜索时，搜索出的内容中的关键字会显示不同的颜色，称之为高亮

百度搜索关键字"ElasticSearch"


京东商城搜索"笔记本"


2.7.2 高亮显示的html分析
通过开发者工具查看高亮数据的html代码实现：


ElasticSearch可以对查询出的内容中关键字部分进行标签和样式的设置，但是你需要告诉ElasticSearch使用什么标签对高亮关键字进行包裹

2.7.3 高亮显示代码实现
@Test
//高亮查询
public void test11() throws Exception{
    // 创建Client连接对象
    Settings settings = Settings.builder().put("cluster.name", "my-elasticsearch").build();
    TransportClient client = new PreBuiltTransportClient(settings)
        .addTransportAddress(new InetSocketTransportAddress(InetAddress.getByName("127.0.0.1"), 9300));

    // 搜索数据
    SearchRequestBuilder searchRequestBuilder = client
        .prepareSearch("blog2").setTypes("article")
        .setQuery(QueryBuilders.termQuery("title", "搜索"));
    
    //设置高亮数据
    HighlightBuilder hiBuilder=new HighlightBuilder();
    hiBuilder.preTags("<font style='color:red'>");
    hiBuilder.postTags("</font>");
    hiBuilder.field("title");
    searchRequestBuilder.highlighter(hiBuilder);
    
    //获得查询结果数据
    SearchResponse searchResponse = searchRequestBuilder.get();
    
    //获取查询结果集
    SearchHits searchHits = searchResponse.getHits();
    System.out.println("共搜到:"+searchHits.getTotalHits()+"条结果!");
    //遍历结果
    for(SearchHit hit:searchHits){
        System.out.println("String方式打印文档搜索内容:");
        System.out.println(hit.getSourceAsString());
        System.out.println("Map方式打印高亮内容");
        System.out.println(hit.getHighlightFields());
    
        System.out.println("遍历高亮集合，打印高亮片段:");
        Text[] text = hit.getHighlightFields().get("title").getFragments();
        for (Text str : text) {
            System.out.println(str);
        }
    }
    
    //释放资源
    client.close();
}
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
36
37
38
39
40
41
42
43


第三章 Spring Data ElasticSearch 使用
3.1 Spring Data ElasticSearch简介
3.1.1 什么是Spring Data
Spring Data是一个用于简化数据库访问，并支持云服务的开源框架。其主要目标是使得对数据的访问变得方便快捷，并支持map-reduce框架和云计算数据服务。 Spring Data可以极大的简化JPA的写法，可以在几乎不用写实现的情况下，实现对数据的访问和操作。除了CRUD外，还包括如分页、排序等一些常用的功能。

Spring Data的官网：http://projects.spring.io/spring-data/

Spring Data常用的功能模块如下：



3.1.2 什么是Spring Data ElasticSearch
Spring Data ElasticSearch 基于 spring data API 简化 elasticSearch操作，将原始操作elasticSearch的客户端API 进行封装 。Spring Data为Elasticsearch项目提供集成搜索引擎。Spring Data Elasticsearch POJO的关键功能区域为中心的模型与Elastichsearch交互文档和轻松地编写一个存储库数据访问层。

官方网站：http://projects.spring.io/spring-data-elasticsearch/

3.2 Spring Data ElasticSearch入门
1）导入Spring Data ElasticSearch坐标

<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.itheima</groupId>
    <artifactId>itheima_elasticsearch_demo3</artifactId>
    <version>1.0-SNAPSHOT</version>


    <dependencies>
        <dependency>
            <groupId>org.elasticsearch</groupId>
            <artifactId>elasticsearch</artifactId>
            <version>5.6.8</version>
        </dependency>
        <dependency>
            <groupId>org.elasticsearch.client</groupId>
            <artifactId>transport</artifactId>
            <version>5.6.8</version>
        </dependency>
        <dependency>
            <groupId>org.apache.logging.log4j</groupId>
            <artifactId>log4j-to-slf4j</artifactId>
            <version>2.9.1</version>
        </dependency>
        <dependency>
            <groupId>org.slf4j</groupId>
            <artifactId>slf4j-api</artifactId>
            <version>1.7.24</version>
        </dependency>
        <dependency>
            <groupId>org.slf4j</groupId>
            <artifactId>slf4j-simple</artifactId>
            <version>1.7.21</version>
        </dependency>
        <dependency>
            <groupId>log4j</groupId>
            <artifactId>log4j</artifactId>
            <version>1.2.12</version>
        </dependency>
        <dependency>
            <groupId>junit</groupId>
            <artifactId>junit</artifactId>
            <version>4.12</version>
        </dependency>


        <dependency>
            <groupId>com.fasterxml.jackson.core</groupId>
            <artifactId>jackson-core</artifactId>
            <version>2.8.1</version>
        </dependency>
        <dependency>
            <groupId>com.fasterxml.jackson.core</groupId>
            <artifactId>jackson-databind</artifactId>
            <version>2.8.1</version>
        </dependency>
        <dependency>
            <groupId>com.fasterxml.jackson.core</groupId>
            <artifactId>jackson-annotations</artifactId>
            <version>2.8.1</version>
        </dependency>


        <dependency>
            <groupId>org.springframework.data</groupId>
            <artifactId>spring-data-elasticsearch</artifactId>
            <version>3.0.5.RELEASE</version>
            <exclusions>
                <exclusion>
                    <groupId>org.elasticsearch.plugin</groupId>
                    <artifactId>transport-netty4-client</artifactId>
                </exclusion>
            </exclusions>
        </dependency>
    
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-test</artifactId>
            <version>5.0.4.RELEASE</version>
        </dependency>
    
    </dependencies>

</project>
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
36
37
38
39
40
41
42
43
44
45
46
47
48
49
50
51
52
53
54
55
56
57
58
59
60
61
62
63
64
65
66
67
68
69
70
71
72
73
74
75
76
77
78
79
80
81
82
83
84
85
86
87
2）创建applicationContext.xml配置文件，引入elasticsearch命名空间

<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xmlns:context="http://www.springframework.org/schema/context"
	xmlns:elasticsearch="http://www.springframework.org/schema/data/elasticsearch"
	xsi:schemaLocation="
		http://www.springframework.org/schema/beans 
		http://www.springframework.org/schema/beans/spring-beans.xsd
		http://www.springframework.org/schema/context 
		http://www.springframework.org/schema/context/spring-context.xsd
		http://www.springframework.org/schema/data/elasticsearch
		http://www.springframework.org/schema/data/elasticsearch/spring-elasticsearch-1.0.xsd
		">
		
</beans>
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
3）编写实体Article

package com.itheima.domain;

public class Article {

    private Integer id;
    private String title;
    private String content;
    public Integer getId() {
        return id;
    }
    public void setId(Integer id) {
        this.id = id;
    }
    public String getTitle() {
        return title;
    }
    public void setTitle(String title) {
        this.title = title;
    }
    public String getContent() {
        return content;
    }
    public void setContent(String content) {
        this.content = content;
    }
    @Override
    public String toString() {
        return "Article [id=" + id + ", title=" + title + ", content=" + content + "]";
    }

}
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
4）编写Dao

package com.itheima.dao;

import com.itheima.domain.Article;
import org.springframework.data.elasticsearch.repository.ElasticsearchRepository;

@Repository
public interface ArticleRepository extends ElasticsearchRepository<Article, Integer> {

}
1
2
3
4
5
6
7
8
9
5）编写Service

package com.itheima.service;

import com.itheima.domain.Article;

public interface ArticleService {

    public void save(Article article);

}
1
2
3
4
5
6
7
8
9
package com.itheima.service.impl;

import com.itheima.dao.ArticleRepository;
import com.itheima.domain.Article;
import com.itheima.service.ArticleService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

@Service
public class ArticleServiceImpl implements ArticleService {

    @Autowired
    private ArticleRepository articleRepository;
    
    public void save(Article article) {
        articleRepository.save(article);
    }

}

1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
6） 配置applicationContext.xml

<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:elasticsearch="http://www.springframework.org/schema/data/elasticsearch"
       xsi:schemaLocation="
		http://www.springframework.org/schema/beans
		http://www.springframework.org/schema/beans/spring-beans.xsd
		http://www.springframework.org/schema/context
		http://www.springframework.org/schema/context/spring-context.xsd
		http://www.springframework.org/schema/data/elasticsearch
		http://www.springframework.org/schema/data/elasticsearch/spring-elasticsearch-1.0.xsd
		">
    
    <!-- 扫描Dao包，自动创建实例 -->
    <elasticsearch:repositories base-package="com.itheima.dao"/>
    
    <!-- 扫描Service包，创建Service的实体 -->
    <context:component-scan base-package="com.itheima.service"/>
    
    <!-- 配置elasticSearch的连接 -->
        <!-- 配置elasticSearch的连接 -->
    <elasticsearch:transport-client id="client" cluster-nodes="localhost:9300" cluster-name="my-elasticsearch"/>


    <!-- ElasticSearch模版对象 -->
    <bean id="elasticsearchTemplate" class="org.springframework.data.elasticsearch.core.ElasticsearchTemplate">
        <constructor-arg name="client" ref="client"></constructor-arg>
    </bean>

</beans>
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
7）配置实体

基于spring data elasticsearch注解配置索引、映射和实体的关系

package com.itheima.domain;

import org.springframework.data.annotation.Id;
import org.springframework.data.elasticsearch.annotations.Document;
import org.springframework.data.elasticsearch.annotations.Field;
import org.springframework.data.elasticsearch.annotations.FieldType;

//@Document 文档对象 （索引信息、文档类型 ）
@Document(indexName="blog3",type="article")
public class Article {

    //@Id 文档主键 唯一标识
    @Id
    //@Field 每个文档的字段配置（类型、是否分词、是否存储、分词器 ）
    @Field(store=true, index = false,type = FieldType.Integer)
    private Integer id;
    @Field(index=true,analyzer="ik_smart",store=true,searchAnalyzer="ik_smart",type = FieldType.text)
    private String title;
    @Field(index=true,analyzer="ik_smart",store=true,searchAnalyzer="ik_smart",type = FieldType.text)
    private String content;
    public Integer getId() {
        return id;
    }
    public void setId(Integer id) {
        this.id = id;
    }
    public String getTitle() {
        return title;
    }
    public void setTitle(String title) {
        this.title = title;
    }
    public String getContent() {
        return content;
    }
    public void setContent(String content) {
        this.content = content;
    }
    @Override
    public String toString() {
        return "Article [id=" + id + ", title=" + title + ", content=" + content + "]";
    }

}
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
36
37
38
39
40
41
42
43
44
其中，注解解释如下：
@Document(indexName="blob3",type="article")：
    indexName：索引的名称（必填项）
    type：索引的类型
@Id：主键的唯一标识
@Field(index=true,analyzer="ik_smart",store=true,searchAnalyzer="ik_smart",type = FieldType.text)
    index：是否设置分词
    analyzer：存储时使用的分词器
    searchAnalyze：搜索时使用的分词器
    store：是否存储
    type: 数据类型
1
2
3
4
5
6
7
8
9
10
11
8）创建测试类SpringDataESTest

package com.itheima.test;

import com.itheima.domain.Article;
import com.itheima.service.ArticleService;
import org.elasticsearch.client.transport.TransportClient;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.elasticsearch.core.ElasticsearchTemplate;
import org.springframework.test.context.ContextConfiguration;
import org.springframework.test.context.junit4.SpringJUnit4ClassRunner;

@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(locations="classpath:applicationContext.xml")
public class SpringDataESTest {

    @Autowired
    private ArticleService articleService;
    
    @Autowired
    private TransportClient client;
    
    @Autowired
    private ElasticsearchTemplate elasticsearchTemplate;
    
    /**创建索引和映射*/
    @Test
    public void createIndex(){
        elasticsearchTemplate.createIndex(Article.class);
        elasticsearchTemplate.putMapping(Article.class);
    }
    
    /**测试保存文档*/
    @Test
    public void saveArticle(){
        Article article = new Article();
        article.setId(100);
        article.setTitle("测试SpringData ElasticSearch");
        article.setContent("Spring Data ElasticSearch 基于 spring data API 简化 elasticSearch操作，将原始操作elasticSearch的客户端API 进行封装 \n" +
                "    Spring Data为Elasticsearch Elasticsearch项目提供集成搜索引擎");
        articleService.save(article);
    }

}

1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
36
37
38
39
40
41
42
43
44
45
3.3 Spring Data ElasticSearch的常用操作
3.3.1 增删改查方法测试
package com.itheima.service;

import com.itheima.domain.Article;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;

public interface ArticleService {

    //保存
    public void save(Article article);
    //删除
    public void delete(Article article);
    //查询全部
    public Iterable<Article> findAll();
    //分页查询
    public Page<Article> findAll(Pageable pageable);

}
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
package com.itheima.service.impl;

import com.itheima.dao.ArticleRepository;
import com.itheima.domain.Article;
import com.itheima.service.ArticleService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.stereotype.Service;


@Service
public class ArticleServiceImpl implements ArticleService {

    @Autowired
    private ArticleRepository articleRepository;
    
    public void save(Article article) {
        articleRepository.save(article);
    }
    
    public void delete(Article article) {
        articleRepository.delete(article);
    }
    
    public Iterable<Article> findAll() {
        Iterable<Article> iter = articleRepository.findAll();
        return iter;
    }
    
    public Page<Article> findAll(Pageable pageable) {
        return articleRepository.findAll(pageable);
    }
}
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
package com.itheima.test;

import com.itheima.domain.Article;
import com.itheima.service.ArticleService;
import org.elasticsearch.client.transport.TransportClient;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.PageRequest;
import org.springframework.data.domain.Pageable;
import org.springframework.data.elasticsearch.core.ElasticsearchTemplate;
import org.springframework.test.context.ContextConfiguration;
import org.springframework.test.context.junit4.SpringJUnit4ClassRunner;

@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(locations="classpath:applicationContext.xml")
public class SpringDataESTest {

    @Autowired
    private ArticleService articleService;
    
    @Autowired
    private TransportClient client;
    
    @Autowired
    private ElasticsearchTemplate elasticsearchTemplate;
    
    /**创建索引和映射*/
    @Test
    public void createIndex(){
        elasticsearchTemplate.createIndex(Article.class);
        elasticsearchTemplate.putMapping(Article.class);
    }
    
    /**测试保存文档*/
    @Test
    public void saveArticle(){
        Article article = new Article();
        article.setId(100);
        article.setTitle("测试SpringData ElasticSearch");
        article.setContent("Spring Data ElasticSearch 基于 spring data API 简化 elasticSearch操作，将原始操作elasticSearch的客户端API 进行封装 \n" +
                "    Spring Data为Elasticsearch Elasticsearch项目提供集成搜索引擎");
        articleService.save(article);
    }
    
    /**测试保存*/
    @Test
    public void save(){
        Article article = new Article();
        article.setId(1001);
        article.setTitle("elasticSearch 3.0版本发布");
        article.setContent("ElasticSearch是一个基于Lucene的搜索服务器。它提供了一个分布式多用户能力的全文搜索引擎，基于RESTful web接口");
        articleService.save(article);
    }
    
    /**测试更新*/
    @Test
    public void update(){
        Article article = new Article();
        article.setId(1001);
        article.setTitle("elasticSearch 3.0版本发布...更新");
        article.setContent("ElasticSearch是一个基于Lucene的搜索服务器。它提供了一个分布式多用户能力的全文搜索引擎，基于RESTful web接口");
        articleService.save(article);
    }
    
    /**测试删除*/
    @Test
    public void delete(){
        Article article = new Article();
        article.setId(1001);
        articleService.delete(article);
    }
    
    /**批量插入*/
    @Test
    public void save100(){
        for(int i=1;i<=100;i++){
            Article article = new Article();
            article.setId(i);
            article.setTitle(i+"elasticSearch 3.0版本发布..，更新");
            article.setContent(i+"ElasticSearch是一个基于Lucene的搜索服务器。它提供了一个分布式多用户能力的全文搜索引擎，基于RESTful web接口");
            articleService.save(article);
        }
    }
    
    /**分页查询*/
    @Test
    public void findAllPage(){
        Pageable pageable = PageRequest.of(1,10);
        Page<Article> page = articleService.findAll(pageable);
        for(Article article:page.getContent()){
            System.out.println(article);
        }
    }
}
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
36
37
38
39
40
41
42
43
44
45
46
47
48
49
50
51
52
53
54
55
56
57
58
59
60
61
62
63
64
65
66
67
68
69
70
71
72
73
74
75
76
77
78
79
80
81
82
83
84
85
86
87
88
89
90
91
92
93
94
95
96
3.3.2 常用查询命名规则
关键字	命名规则	解释	示例
and	findByField1AndField2	根据Field1和Field2获得数据	findByTitleAndContent
or	findByField1OrField2	根据Field1或Field2获得数据	findByTitleOrContent
is	findByField	根据Field获得数据	findByTitle
not	findByFieldNot	根据Field获得补集数据	findByTitleNot
between	findByFieldBetween	获得指定范围的数据	findByPriceBetween
lessThanEqual	findByFieldLessThan	获得小于等于指定值的数据	findByPriceLessThan
3.3.3 查询方法测试
1）dao层实现

package com.itheima.dao;

import com.itheima.domain.Article;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.elasticsearch.repository.ElasticsearchRepository;
import java.util.List;

public interface ArticleRepository extends ElasticsearchRepository<Article, Integer> {
    //根据标题查询
    List<Article> findByTitle(String condition);
    //根据标题查询(含分页)
    Page<Article> findByTitle(String condition, Pageable pageable);
}
1
2
3
4
5
6
7
8
9
10
11
12
13
14
2）service层实现

public interface ArticleService {
    //根据标题查询
    List<Article> findByTitle(String condition);
    //根据标题查询(含分页)
    Page<Article> findByTitle(String condition, Pageable pageable);
}
1
2
3
4
5
6
package com.itheima.service.impl;

import com.itheima.dao.ArticleRepository;
import com.itheima.domain.Article;
import com.itheima.service.ArticleService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.stereotype.Service;

import java.util.List;

@Service
public class ArticleServiceImpl implements ArticleService {

    @Autowired
    private ArticleRepository articleRepository;
    
    public List<Article> findByTitle(String condition) {
        return articleRepository.findByTitle(condition);
    }
    public Page<Article> findByTitle(String condition, Pageable pageable) {
        return articleRepository.findByTitle(condition,pageable);
    }

}

1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
3）测试代码

package com.itheima.test;

import com.itheima.domain.Article;
import com.itheima.service.ArticleService;
import org.elasticsearch.client.transport.TransportClient;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.PageRequest;
import org.springframework.data.domain.Pageable;
import org.springframework.data.elasticsearch.core.ElasticsearchTemplate;
import org.springframework.test.context.ContextConfiguration;
import org.springframework.test.context.junit4.SpringJUnit4ClassRunner;

import java.util.List;

@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(locations="classpath:applicationContext.xml")
public class SpringDataESTest {

    @Autowired
    private ArticleService articleService;
    
    @Autowired
    private TransportClient client;
    
    @Autowired
    private ElasticsearchTemplate elasticsearchTemplate;
    
    /**条件查询*/
    @Test
    public void findByTitle(){
        String condition = "版本";
        List<Article> articleList = articleService.findByTitle(condition);
        for(Article article:articleList){
            System.out.println(article);
        }
    }
    
    /**条件分页查询*/
    @Test
    public void findByTitlePage(){
        String condition = "版本";
        Pageable pageable = PageRequest.of(2,10);
        Page<Article> page = articleService.findByTitle(condition,pageable);
        for(Article article:page.getContent()){
            System.out.println(article);
        }
    }

}

1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
36
37
38
39
40
41
42
43
44
45
46
47
48
49
50
51
52
53
3.3.4使用Elasticsearch的原生查询对象进行查询。
@Test
    public void findByNativeQuery() {
        //创建一个SearchQuery对象
        SearchQuery searchQuery = new NativeSearchQueryBuilder()
                //设置查询条件，此处可以使用QueryBuilders创建多种查询
                .withQuery(QueryBuilders.queryStringQuery("备份节点上没有数据").defaultField("title"))
                //还可以设置分页信息
                .withPageable(PageRequest.of(1, 5))
                //创建SearchQuery对象
                .build();


        //使用模板对象执行查询
        elasticsearchTemplate.queryForList(searchQuery, Article.class)
                .forEach(a-> System.out.println(a));
    }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
部分代码
package com.sw.fam.service.ais.impl;

import com.sw.fam.service.ais.AisService;
import com.sw.fam.utils.FileComponent;
import com.sw.fam.vo.ais.IndexEntityVO;
import com.sw.fam.vo.ais.ReturnBean;
import com.sw.fam.vo.ais.SearchBean;
import com.sw.seeker.common.util.CurrentUser;
import org.apache.commons.collections.CollectionUtils;
import org.apache.commons.lang.StringUtils;
import org.elasticsearch.action.search.SearchRequest;
import org.elasticsearch.action.search.SearchResponse;
import org.elasticsearch.client.RequestOptions;
import org.elasticsearch.client.RestHighLevelClient;
import org.elasticsearch.common.text.Text;
import org.elasticsearch.index.query.BoolQueryBuilder;
import org.elasticsearch.index.query.QueryBuilders;
import org.elasticsearch.index.query.QueryStringQueryBuilder;
import org.elasticsearch.search.SearchHit;
import org.elasticsearch.search.SearchHits;
import org.elasticsearch.search.aggregations.Aggregation;
import org.elasticsearch.search.aggregations.AggregationBuilder;
import org.elasticsearch.search.aggregations.AggregationBuilders;
import org.elasticsearch.search.aggregations.bucket.terms.ParsedStringTerms;
import org.elasticsearch.search.aggregations.bucket.terms.Terms;
import org.elasticsearch.search.builder.SearchSourceBuilder;
import org.elasticsearch.search.fetch.subphase.highlight.HighlightBuilder;
import org.elasticsearch.search.fetch.subphase.highlight.HighlightField;
import org.elasticsearch.search.sort.SortOrder;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

import java.io.IOException;
import java.util.ArrayList;
import java.util.HashMap;
import java.util.Iterator;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;
import java.util.regex.Pattern;

/**
 * @author ganquanzhong
 * @version v1.0
 * @description 检索服务接口实现类
 * @since 2020-09-21 v1.0
 **/
    @Service
    public class AisServiceImpl implements AisService {

    @Autowired
    private RestHighLevelClient restHighLevelClient;

    @Autowired
    private FileComponent fileComponent;

    /**
     * @param param {@link com.sw.fam.vo.ais.SearchBean} 请求参数
     * @return ReturnBean {@link com.sw.fam.vo.ais.ReturnBean} 返回结果
     * @throws IOException 检索异常
     * @description 全部 全文检索
     * @create ganquanzhong 2020/09/21
     */
        @Override
        public ReturnBean search(SearchBean param) throws IOException {
        //最终的请求体
        SearchRequest searchRequest = null;
        String indexName = param.getClusterName();
        String searchValue = param.getSearchValue();
        // 1.构建SearchRequest检索请求
        // 专门用来进行全文检索、关键字检索的API
        if (indexName == null || "".equals(indexName)) {
            searchRequest = new SearchRequest();
        } else {
            searchRequest = new SearchRequest(indexName);
        }

        // 2.创建一个SearchSourceBuilder专门用于构建查询条件
        SearchSourceBuilder sourceBuilder = new SearchSourceBuilder();

        //使用BoolQueryBuilder进行复合查询  过滤字段，效率高于must
        BoolQueryBuilder boolQueryBuilder = QueryBuilders.boolQuery();
        // 输入框  查询条件
        if (searchValue == null || searchValue.isEmpty()) {
            sourceBuilder.query(QueryBuilders.matchAllQuery());
        } else {
            String reg = "^[a-z0-9A-Z]+$";
            // 判断输入的内容是否完全由数字或字母组成 大写转为小写
            boolean flag = Pattern.compile(reg).matcher(searchValue).matches();
            if (flag) {
                searchValue = "*" + searchValue + "*";
            }
            QueryStringQueryBuilder builder = new QueryStringQueryBuilder(searchValue)
                    .fields(IndexEntityVO.getSearchField().get(indexName))
                    .analyzer("ik_smart");
            boolQueryBuilder.must(builder);
        }


        // 分组条件  entrySet遍历，在键和值都需要时使用（最常用）   此处是遍历传入的字段限定,为完全符合
        BoolQueryBuilder mustQueryBuilder = QueryBuilders.boolQuery();
        if (param.getSearchMap() != null && param.getSearchMap().size() > 0) {
            for (Map.Entry<String, Object> entry : param.getSearchMap().entrySet()) {
                //判断组织id
                if ("orgId".equals(entry.getKey())) {
                    String orgIds = CurrentUser.getOrgIds();
                    String newOrgIds = orgIds.replace("'", "");
                    String[] split = newOrgIds.split(",");
                    if (split.length > 0) {
                        boolQueryBuilder.filter(QueryBuilders.termsQuery(entry.getKey(), split));
                    }
                    continue;
                }
                //处理排序
                if (entry.getValue() != null && "order".equals(entry.getKey())) {
                    // searchMap中键值为es中索引的字段，值为arrayList封装的
                    ArrayList<String> terms = (ArrayList<String>) entry.getValue();
                    if (terms.size() > 0) {
                        terms.forEach(item -> {
                            sourceBuilder.sort(item, SortOrder.DESC);
                        });
                    }
                } else {
                    //处理条件查询
                    if (entry.getValue() != null && !"".equals(entry.getValue())) {
                        if (entry.getValue() instanceof ArrayList) {
                            // searchMap中键值为es中索引的字段，值为arrayList封装的
                            ArrayList<String> terms = (ArrayList<String>) entry.getValue();
                            BoolQueryBuilder shouldQueryBuilder = QueryBuilders.boolQuery();
                            if (terms.size() > 0) {
                                terms.forEach(item -> {
                                    shouldQueryBuilder.should(QueryBuilders.termQuery(entry.getKey(), item));
                                });
                                mustQueryBuilder.must(shouldQueryBuilder);
                            }
                        } else {
                            boolQueryBuilder.filter(QueryBuilders.termQuery(entry.getKey(), entry.getValue()));
                        }
                    }
                }
            }
        }
        boolQueryBuilder.must(mustQueryBuilder);
        //匹配度倒数，数值越大匹配度越高
        sourceBuilder.sort("_score", SortOrder.DESC);
    
        // 给请求设置需要高亮显示的字段
        HighlightBuilder highlightBuilder = new HighlightBuilder();
        List highlightFieldList = IndexEntityVO.getHighlightField().get(indexName);
        if (CollectionUtils.isNotEmpty(highlightFieldList)) {
            for (int i = 0; i < highlightFieldList.size(); i++) {
                String name = highlightFieldList.get(i).toString();
                HighlightBuilder.Field highlight =
                        new HighlightBuilder.Field(name).preTags("<span>").postTags("</span>").fragmentSize(200).numOfFragments(1);
                highlight.highlighterType("unified");
                highlightBuilder.field(highlight);
            }
        }
        sourceBuilder.highlighter(highlightBuilder);
    
        //分页处理
        sourceBuilder.query(boolQueryBuilder).from((param.getPageNum() - 1) * param.getPageSize()).size(param.getPageSize());
    
        // 4.调用SearchRequest.source将查询条件设置到检索请求
        searchRequest.source(sourceBuilder);
    
        // 5.执行RestHighLevelClient.search发起请求
        SearchResponse response = restHighLevelClient.search(searchRequest, RequestOptions.DEFAULT);
    
        ReturnBean returnBean = new ReturnBean();
        List<Map<String, Object>> list = new ArrayList<>();
        //SearchHits提供有关所有匹配的全局信息，例如总命中数或最高分数：
        SearchHits hits = response.getHits();
        long totalHits = response.getHits().getTotalHits();
        returnBean.setTotal(totalHits);
        returnBean.setCurrent(param.getPageNum());
        returnBean.setSize(param.getPageSize());
        SearchHit[] searchHits = hits.getHits();
        ArrayList<String> fileInfoIdList = new ArrayList<>();
        for (SearchHit hit : searchHits) {
            //获取命中的数据源
            Map<String, Object> source = hit.getSourceAsMap();
            //文件类别需要处理
            if ("fam_file_idx".equals(indexName) || (source.get("flag") != null && source.get("flag").toString().equals("file"))) {
                if (source.get("fileInfoId") != null) {
                    fileInfoIdList.add(source.get("fileInfoId").toString());
                }
            }
            //高亮结果处理
            Map<String, HighlightField> highlightFields = hit.getHighlightFields();
            if (CollectionUtils.isNotEmpty(highlightFieldList)) {
                for (int i = 0; i < highlightFieldList.size(); i++) {
                    String name = highlightFieldList.get(i).toString();
                    HighlightField highlightField = highlightFields.get(name);
                    StringBuilder sb = new StringBuilder();
                    if (highlightField != null) {
                        Text[] fragments = highlightField.getFragments();
                        for (Text text : fragments) {
                            sb.append(text);
                        }
                        source.put(name, sb.toString());
                    }
                }
            }
            list.add(source);
        }
        if (CollectionUtils.isNotEmpty(fileInfoIdList)) {
            HashMap<String, Object> map = fileComponent.queryFileList(fileInfoIdList);
            for (Map<String, Object> objectMap : list) {
                if (map.get(objectMap.get("fileInfoId")) != null) {
                    LinkedHashMap fileInfoMap = (LinkedHashMap) map.get(objectMap.get("fileInfoId"));
                    objectMap.put("fileInfoId", fileInfoMap.get("filePath") + fileInfoMap.get("fileName").toString());
                    objectMap.put("filePath", fileInfoMap.get("filePath"));
                    objectMap.put("fileName", fileInfoMap.get("fileName"));
                    objectMap.put("serverPath", fileInfoMap.get("serverPath"));
                }
            }
        }
        returnBean.setRecords(list);
        return returnBean;
    }


    /**
     * @param param {@link com.sw.fam.vo.ais.SearchBean} 请求参数
     * @return ReturnBean {@link com.sw.fam.vo.ais.ReturnBean} 返回结果
     * @throws IOException 检索服务异常
     * @description 获取某个索引下面的分类
     * @create ganquanzhong 2020/09/22
     */
    @Override
    public ReturnBean queryGroupList(SearchBean param) throws IOException {
        SearchRequest searchRequest = null;
        String indexName = param.getClusterName();
    
        // 构建SearchRequest检索请求
        if (indexName == null || "".equals(indexName)) {
            //查全部
            searchRequest = new SearchRequest();
        } else {
            searchRequest = new SearchRequest(indexName);
        }
    
        // 创建一个SearchSourceBuilder专门用于构建查询条件
        SearchSourceBuilder sourceBuilder = new SearchSourceBuilder();


        // 分组条件时加入的条件，如项目等
        BoolQueryBuilder boolQueryBuilder = QueryBuilders.boolQuery();
        if (param.getSearchMap() != null && param.getSearchMap().size() > 0) {
            for (Map.Entry<String, Object> entry : param.getSearchMap().entrySet()) {
                //判断组织id
                if ("orgId".equals(entry.getKey())) {
                    String orgIds = CurrentUser.getOrgIds();
                    String newOrgIds = orgIds.replace("'", "");
                    String[] split = newOrgIds.split(",");
                    if (split.length > 0) {
                        boolQueryBuilder.filter(QueryBuilders.termsQuery(entry.getKey(), split));
                    }
                    continue;
                }
                //处理条件查询
                if (entry.getValue() != null && !"".equals(entry.getValue())) {
                    boolQueryBuilder.filter(QueryBuilders.termQuery(entry.getKey(), entry.getValue()));
                }
            }
        }
    
        //分组查询封装
        List searchGroup = IndexEntityVO.getSearchGroupTerm().get(indexName);
        ReturnBean returnBean = new ReturnBean();
        List<Map<String, Object>> list = new ArrayList<>();
        HashMap<String, Object> group = new HashMap<>();
        if (CollectionUtils.isNotEmpty(searchGroup)) {
            SearchResponse response = null;
            Map<String, Aggregation> aggMap = null;
            ParsedStringTerms stringTerms = null;
            Iterator<? extends Terms.Bucket> iterator = null;
            for (int i = 0; i < searchGroup.size(); i++) {
                //聚合查询
                AggregationBuilder groupAgg = AggregationBuilders
                        .terms("group_" + searchGroup.get(i)).field(searchGroup.get(i).toString()).size(Integer.MAX_VALUE);
                sourceBuilder.aggregation(groupAgg).query(boolQueryBuilder);
                searchRequest.source(sourceBuilder);
                response = restHighLevelClient.search(searchRequest, RequestOptions.DEFAULT);
                //处理返回的聚合结果
                aggMap = response.getAggregations().asMap();
                if (aggMap != null && aggMap.size() > 0) {
                    stringTerms = (ParsedStringTerms) aggMap.get("group_" + searchGroup.get(i));
                    //将集合转换成迭代器遍历桶
                    iterator = stringTerms.getBuckets().iterator();
                    ArrayList<Map> resultList = new ArrayList<>();
                    while (iterator.hasNext()) {
                        HashMap<String, Object> sourceMap = new HashMap<>();
                        Terms.Bucket bucket = iterator.next();
                        //bucket桶也是一个map对象， 我们取它的key值就可以
                        if (StringUtils.isNotEmpty(bucket.getKeyAsString())) {
                            sourceMap.put("id", bucket.getKeyAsString());
                            resultList.add(sourceMap);
                        }
                    }
                    group.put(searchGroup.get(i) + "List", resultList);
                }
            }
        }
        list.add(group);
        returnBean.setRecords(list);
        return returnBean;
    }
}

————————————————
版权声明：本文为CSDN博主「ForFuture  Code」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。