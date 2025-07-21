---
title: Elasticsearch-2
tags: 
  - Elasticsearch
date: 2023-03-30 09:00:00
categories:
  - Elasticsearch
---

原文链接：https://blog.csdn.net/ganquanzhong/article/details/108633025

## ElasticSearch（二）

### 学习目标：

能够使用java客户端完成创建、删除索引的操作
能够使用java客户端完成文档的增删改的操作
能够使用java客户端完成文档的查询操作
能够完成文档的分页操作
能够完成文档的高亮查询操作
能够搭建Spring Data ElasticSearch的环境
能够完成Spring Data ElasticSearch的基本增删改查操作
能够掌握基本条件查询的方法命名规则

### 第一章 ElasticSearch编程操作

#### 1.1 创建工程，导入坐标

pom.xml坐标

```xml
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
```

#### 1.2 创建索引index

```java
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
```

![](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202303311514661.png)

![](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202303311515862.png)

#### 1.3 创建映射mapping

```java
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
```

![](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202303311516885.png)

![](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202303311516167.png)

#### 1.4 建立文档document

##### 1.4.1 建立文档（通过XContentBuilder）

```java
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
```

![](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202303311516096.png)

##### 1.4.2 建立文档（使用Jackson转换实体）

1）创建Article实体

```java
public class Article {
	private Integer id;
	private String title;
	private String content;
    getter/setter...
}
```

2）添加jackson坐标

```xml
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
```

3）代码实现

```java
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
```
![](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202303311518220.png)

#### 1.5 查询文档操作

##### 1.5.1关键词查询

```java
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
```

##### 1.5.2 字符串查询

```java
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
```

##### 1.5.3 使用文档ID查询文档

```java
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
```

#### 1.6 查询文档分页操作

##### 1.6.1 批量插入数据

```java
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
```
##### 1.6.2 分页查询

```java
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
```
#### 1.7 查询结果高亮操作

##### 1.7.1 什么是高亮显示

在进行关键字搜索时，搜索出的内容中的关键字会显示不同的颜色，称之为高亮

百度搜索关键字"ElasticSearch"

![](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202303311534261.png)


京东商城搜索"笔记本"

![](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202303311535122.png)

#### 2.7.2 高亮显示的html分析

通过开发者工具查看高亮数据的html代码实现：

![](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202303311536158.png)

ElasticSearch可以对查询出的内容中关键字部分进行标签和样式的设置，但是你需要告诉ElasticSearch使用什么标签对高亮关键字进行包裹

![](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202303311536428.png)

#### 2.7.3 高亮显示代码实现

```java
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
```
![](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202303311538382.png)

### 第三章 Spring Data ElasticSearch 使用

#### 3.1 Spring Data ElasticSearch简介

##### 3.1.1 什么是Spring Data

Spring Data是一个用于简化数据库访问，并支持云服务的开源框架。其主要目标是使得对数据的访问变得方便快捷，并支持map-reduce框架和云计算数据服务。 Spring Data可以极大的简化JPA的写法，可以在几乎不用写实现的情况下，实现对数据的访问和操作。除了CRUD外，还包括如分页、排序等一些常用的功能。

Spring Data的官网：http://projects.spring.io/spring-data/

Spring Data常用的功能模块如下：

![](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202303311549504.png)

![](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202303311608259.png)

#### 3.1.2 什么是Spring Data ElasticSearch

Spring Data ElasticSearch 基于 spring data API 简化 elasticSearch操作，将原始操作elasticSearch的客户端API 进行封装 。Spring Data为Elasticsearch项目提供集成搜索引擎。Spring Data Elasticsearch POJO的关键功能区域为中心的模型与Elastichsearch交互文档和轻松地编写一个存储库数据访问层。

官方网站：http://projects.spring.io/spring-data-elasticsearch/

### 3.2 Spring Data ElasticSearch入门

1）导入Spring Data ElasticSearch坐标


```xml
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
```

2）创建applicationContext.xml配置文件，引入elasticsearch命名空间

```xml
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
```

3）编写实体Article

```java
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
```

4）编写Dao

```java
package com.itheima.dao;

import com.itheima.domain.Article;
import org.springframework.data.elasticsearch.repository.ElasticsearchRepository;

@Repository
public interface ArticleRepository extends ElasticsearchRepository<Article, Integer> {

}
```

5）编写Service

```java
package com.itheima.service;

import com.itheima.domain.Article;

public interface ArticleService {
	public void save(Article article);
}
```

```java
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
```

6） 配置applicationContext.xml

```xml
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
```

7）配置实体

基于spring data elasticsearch注解配置索引、映射和实体的关系

```java
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
```

```txt
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
```

8）创建测试类SpringDataESTest

```java
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
```

#### 3.3 Spring Data ElasticSearch的常用操作

##### 3.3.1 增删改查方法测试

```java
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
```

```java
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
```
```java
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
```
##### 3.3.2 常用查询命名规则

|    关键字     |       命名规则        |            解释            |         示例          |
| :-----------: | :-------------------: | :------------------------: | :-------------------: |
|      and      | findByField1AndField2 | 根据Field1和Field2获得数据 | findByTitleAndContent |
|      or       | findByField1OrField2  | 根据Field1或Field2获得数据 | findByTitleOrContent  |
|      is       |      findByField      |     根据Field获得数据      |      findByTitle      |
|      not      |    findByFieldNot     |   根据Field获得补集数据    |    findByTitleNot     |
|    between    |  findByFieldBetween   |     获得指定范围的数据     |  findByPriceBetween   |
| lessThanEqual |  findByFieldLessThan  |  获得小于等于指定值的数据  |  findByPriceLessThan  |

##### 3.3.3 查询方法测试

1）dao层实现

```java
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
```

2）service层实现

```java
public interface ArticleService {
    //根据标题查询
    List<Article> findByTitle(String condition);
    //根据标题查询(含分页)
    Page<Article> findByTitle(String condition, Pageable pageable);
}
```

```java
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
```

3）测试代码

```java
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
```

##### 3.3.4使用Elasticsearch的原生查询对象进行查询。


```java
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
```
#### 部分代码


```java
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
```


版权声明：本文为CSDN博主「ForFuture  Code」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。