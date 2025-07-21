---
title: Elasticsearch学习笔记
tags: 
  - Elasticsearch
date: 2023-04-04 09:00:00
categories:
  - Elasticsearch
---

## ES学习笔记

### 1.搭建ELK集群

后续补充

### 2.ES基础

#### 2.1 基本操作

http的操作需要大写（PUT/GET/POST/DELETE/HEAD/OPTIONS）

```url
# 使用PUT向 dbindex/_doc/1中插入数据，指定id=1
PUT dbindex/_doc/1
{
  "name":"yim",
  "age":25,
  "actionTime": "2023-04-04",
  "id":2,
  "addr": "随便"
}

# 在 dbindex/_doc/中获取id=1的数据
GET dbindex/_doc/1

# 使用GET在dbindex/_doc/中获取所有数据
GET dbindex/_doc/_search

# 使用POST向 dbindex/_doc中插入数据，不用指定文档id
POST dbindex/_doc
{
  "name":"123"
}

# 使用POST修改文档，语法 POST dbindex/_doc/文档_id/_update
POST dbindex/_doc/yyhOSocB7VY9_-BPWtxj/_update
{
  "doc":{
    "name":"223"
  }
}

# PUT执行修改操作时，会对文档的整个内容进行替换
# POST执行修改操作时，如果字段在文档中，则修改此字段的值；如果字段不在文档中，则把此字段加入文档信息中

# 使用POST查询索引库中的所有数据
POST dbindex/_doc/_search

# 使用DELTE删除dbindex/_doc中的文档
DELETE dbindex/_doc/某一_id
```

#### 2.2 使用GET查询

常用的GET查询语句

```url
# 根据_id查询文档详情
GET 索引库名称/_doc/文档_id
# 查询指定索引库中所有的文档信息
GET 索引库名称/_doc/_search
# 查询当前集群中所有的别名索引库信息
GET /_cat/indices
# 查询当前集群中所有的别名索引信息
GET /_cat/aliases
# 查询当前集群的颜色信息
GET /_cat/health
# 查询当前集群中主节点的信息
GET /_cat/master
# 查询当前集群中从节点的信息
GET /_cat/node
# 查询当前集群中索引分片的信息
GET /_cat/shards
# 查询当前集群的健康状态
GET _cluster/health?pretty=true
# 查询当前集群的运行状态信息
GET _cluster/stats?pretty
# 查询当前集群中所有节点的监控信息
GET _nodes/stats?pretty
# 查询当前集群中所有索引的监控信息
GET _stats?pretty
```

注：

```json
集群的健康状态：
{
  "cluster_name" : "elasticsearch-cluster",
  "status" : "green",
  "timed_out" : false,
  "number_of_nodes" : 3,
  "number_of_data_nodes" : 3,
  "active_primary_shards" : 8,
  "active_shards" : 16,
  "relocating_shards" : 0,
  "initializing_shards" : 0,
  "unassigned_shards" : 0,
  "delayed_unassigned_shards" : 0,
  "number_of_pending_tasks" : 0,
  "number_of_in_flight_fetch" : 0,
  "task_max_waiting_in_queue_millis" : 0,
  "active_shards_percent_as_number" : 100.0
}
参数：
status：表示集群状态，green、yellow、red
number_of_nodes/number_of_data_nodes：表示集群的节点数和数据节点数
active_primary_shards：表示集群中所有活跃的主分片数
active_shards：表示集群中所有活跃的分片数
relocating_shards：表示当前节点迁往其他节点的分片数量，通常为0，当有节点加入或者退出时该值会有相应变化
initializing_shards：表示正在初始化的分片
unassigned_shards：表示未分配的分片数，通常为0。当有某个节点的副本分片丢失时，该值会相应的变化
number_of_pending_tasks：表示主节点创建索引并分配了分片任务，如果该指标数值一直未减少，则代表集群存在不稳定因素，需要排除找到具体问题。而等待的任务只能由主节点进行处理，这些任务包括创建索引并将分片任务分配给节点
active_shards_percent_as_number：表示当前集群中分片的健康度（活跃分片数占总分片数的比例）
```

```json
查询集群节点的监控信息
参数：
name：表示节点名称。
roles：表示节点角色类型。
indices.docs.count：表示索引文档总数量。
segments.count：表示集群中段（数据存储单位）的总数量。
jvm.heap_used_percent：表示内存的使用百分比。
thread_pool.{bulk,index,get,search}.{active,queue,rejectd}：表示线程池的一些信息
主要指标有激活的(active)线程数，线程队列(queue)数和被拒绝(rejected)线程数量
以下指标是累加值，当节点重启之后会清零：
indices.indexing.index_total：表示索引文档数道。
indices.indexing.index_time_in_millis：表示使用索引总耗时。
indices.gt.total：表示get请求的数量。
indices.get.time_in_millis：表示get请求总耗时。
indices.scarch.query_total：表示search请求的总数。
indices.search.query_time_in_milis：表示search求的总耗时。
indices.search.fetch_total：表示fetch请求的总数量。
indices.search.fetch_time_in_millis: 表示fetch请求的总耗时。
jvm.gc.collectors.young.collection count：表示年轻代垃圾回收次数。
jvm.ge.colleaors young.colection_time_in_millis:表示年轻代垃圾回收总耗时。
jvm.gc.collectors.old.collection_count：表示老年代垃圾回收次数。
jvm.ge.collectors old.collection_time_in_milis:表示老年代垃圾回收总耗时。

需要计算的参数：
index_per_min：每分钟索引请求数量。计算公式如下：
案引请求率=index_total（指标的两次采集差值）/系统时间差值 (ms) × 60000 （公式1）
indexAverge_per_min：索引请求处理延迟。计算公式如下：
索引延迟率=index_time_in_millis（两次耗时的差值）/index_total (两次索引数量的差值) （公式2）
get_per_min：表示每分钟get请求的数量，参照计算公式1。
getAverage_per_min：表示get请求的处理延迟率，参照计算公式2。
merge_per_min：表示每分钟merge请求的数量，参照计算公式1。
mergeAverage_per_min：表示merge请求的处理延迟率，参照计算公式2。
searchQuery_per_min：表示每分钟query请求的数童，参照计算公式1。
searchQueryAverage_per_min：表示query请求的延迟率，参照计算公式2。
searchFetcb_per_min：表示每分钟fetch请求的数童，参照计算公式1。
searchFetchAverage_per_min：表示fetch请求的延迟率，参照计算公式2。
youngGc_per_min：表示每分钟young sc的数量，参照计算公式1。
youngGcAverage_per_min：表示母分钟young gC请求的延迟率，参照计算公式2。
oldGc_per_min：表示每分钟old gc的数量，参照计算公式1。
oldGcAverage_per_min：表示每分钟old gc请求的延迟率，参照计算公式2。
```

#### 2.3 es的映射

映射是定义文档及其包含的字段如何存储在索引库中的过程。每个文档都是一个字段的集合，每个字段都有各自的数据类型，映射数据时，可以创建一个映射的定义，其中包含与文档相关字段的列表。映射定义还包括元数据字段（_source），他定义如何处理文档关联的元数据，而es支持动态映射和显式映射。每种方法根据不同的使用情况有不同的好处。如果要明确定义映射或者更好地控制创建字段和字段类型，则可以利用es的显示映射。动态映射和显式映射的详细说明如下

- 动态映射方便用户快速写入索引数据，es会根据写入的文档内容自动添加字段和字段类型，用户只负责写入具体的数据即可
- 显式映射需要用户精确地选择如何定义映射，例如哪些字符串字段应被视为全文字段，哪些字段是数字或者日期格式。显式索引是用于控制动态添加字段映射的自定义规则，而且在显示映射中使用runtime（运行时映射）时无需重新创建索引就可以更改数据存储的规则

##### 2.3.1 动态映射

动态映射是es非常重要的功能之一，使用户不需要关注存储结果，能够在业务中尽快使用数据进行搜索。使用动态银蛇在写入索引文档数据时，不需要先创建索引和定义字段，索引中的字段和字段类型都将自动创建

```url
PUT datadb/_doc/1
{"count": 10}
```

以上语句创建了一个名为datadb的索引库，然后为此索引库添加count字段，该字段的数据类型是long，而long类型都是动态映射自动生成的。我们可以使用GET datadb语句进行查询。

elasticsearch在用户写入数据时检测到新的字段时，会动态地将该字段添加到类型映射中，由dynamic参数来控制此行为。用户可以通过将dynamic参数设置为true或明确指示elasticsearch根据传入的文档来动态创建字段。

映射规则

| JSON数据类型         | "dynamic":"ture"         | "dynamic":"runtime"      |
| -------------------- | ------------------------ | ------------------------ |
| true或者false        | boolean                  | boolean                  |
| double类型           | float                    | double                   |
| integer类型          | long                     | long                     |
| object类型           | object                   | object                   |
| array类型            | 取决于数组的第一个元素值 | 取决于数组的第一个元素值 |
| 日期类型             | date                     | date                     |
| 数字类型             | float或者long            | double或者long           |
| 日期和数字之外的类型 | 带有.keyword子字段的文本 | keyword                  |

###### 2.3.1.1 日期检测

###### 2.3.1.2 禁用日期检测

###### 2.3.1.3 自定义日期检测格式

###### 2.3.1.4 数字检测



#### 2.3.2 动态映射模板

动态映射模板使得在默认的动态字段映射规则之外能够更好地控制es如何映射数据。用户可以通过将dynamic参数设置为true来启用动态映射模式，然后使用动态模板来定义自定义映射，这些映射可以根据匹配条件应用于动态添加的字段。相关参数如下

- match_mapping_type：表示对es检测到的数据类型进行操作
- match和unmatch：表示使用模式匹配来匹配字段名称



### 3.es字段类型

==常用类型==

- binary：表示可以存储编码为base64的字符串或者二进制值
- boolean：表示可以存储true和false的布尔值
- keyword：该字段类型的数据再存储时不会进行分词处理，适合进行统计分析，不能进行全文搜索
- numbers：用于表示数字类型，例如long和double
- date：表示可以存储日期类型的数据
- alias：表示为现有字段定义别名
- text：该字段类型的数据再存储时会进行分词并建立索引，适合进行全文搜索，不能进行统计分析

==对象和关系类型==

- object：表示一个json对象
- nested：嵌套类型，对象中可以嵌套对象
- array：在es中，数组不需要专用的字段类型，默认情况下，任何字段都可以包含0个或多个值，但是数组中的所有值必须具有相同的字段类型

==其余类型==

- range：表示范围类型
- rank_feature：表示排名类型
- token_count：表示令牌计数类型
- ip：用于ip地址的存储和查询的类型
- geo_point，geo_shape：用于地理位置和空间位置存储和搜索的类型

#### 3.1 alias类型

```txt
# 创建名为userinfo的索引库并为其创建映射关系
PUT userinfo
{
	"mappings":{
		"properties":{
			"age":{
				"type":"long"
			},
			"aliasage":{
				"type":"alias",
				"path":"age"
			},
			"transit_mode":{
				"type":"keyword"
			}
		}
	}
}
```

以上语句创建了userinfo索引库，而且为age字段创建了名为aliasage的别名

```txt
# 在索引库userinfo中插入一条文档数据
PUT userinfo/_doc/1
{
	"age":39,
	"transit_mode":"transit_mode"
}
# 通过别名查询年龄大于30的用户信息
GET userinfo/_doc/_search
{
	"query":{
		"range":{
			"aliasage":{
				"gte": 30
			}
		}
	}
}
```

在使用别名的时候需要注意，可以使用别名进行数据的搜索，但是不能使用别名进行数据的写入

#### 3.2 数组类型

在es中，没有专用的数组数据类型。默认情况下，任何字段都可以包含0个或多个值，不过数组中的所有值必须具有相同的数据类型

#### 3.3 binary类型

binary定义的字段类型可以存储编码为Base64的字符串或者二进制值

#### 3.4 布尔类型

#### 3.5 日期类型

#### 3.6 nested类型

