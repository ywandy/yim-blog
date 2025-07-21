---
title: Redis-jedis
tags: 
  - Redis
date: 2023-04-17 20:51:00
categories:	
  - Redis
---

## Jedis

本篇对jedis 即用java操作redis的常用方法用法整理

redis是一种高级的key-value的存储系统

其中的key是字符串类型，尽可能满足如下几点：

1）key不要太长，最好不要操作1024个字节，这不仅会消耗内存还会降低查找 效率
2）key不要太短，如果太短会降低key的可读性
3）在项目中，key最好有一个统一的命名规范（根据企业的需求）

其中value 支持五种数据类型：

1）字符串型 string
2）字符串列表 lists
3）字符串集合 sets
4）有序字符串集合 sorted sets
5）哈希类型 hashs

> jedis 就是 java redis 的简写

### jedis语法总结

```cobol
Jedis jedis = new Jedis(String ip , String port)
```

#### 1. jedis中对键通用的操作

| 方法                               | 描述                              | 返回值 /补充说明           |
| :--------------------------------- | :-------------------------------- | :------------------------- |
| jedis.flushAll()                   | 清空所有数据库的数据              |                            |
| jedis.flushDB()                    | 清空当前数据库的数据              |                            |
| boolean jedis.exists(String key)   | 判断某个键是否存在                | true = 存在，false= 不存在 |
| jedis.set(String key,String value) | 新增键值对（key,value）           | 返回String类型的OK代表成功 |
| Set<String> jedis.keys(\*)         | 获取所有key                       | 返回set 无序集合           |
| jedis.del(String key)              | 删除键为key的数据项               |                            |
| jedis.expire(String key,int i)     | 设置键为key的过期时间为i秒        |                            |
| int jedis.ttl(String key)          | 获取键为key数据项的剩余时间（秒） |                            |
| jedis.persist(String key)          | 移除键为key属性项的生存时间限制   |                            |
| jedis.type(String key)             | 查看键为key所对应value的数据类型  |                            |

#### 2. jedis中 字符串的操作

字符串类型是Redis中最为基础的数据存储类型，它在Redis中是二进制安全的，这 便意味着该类型可以接受任何格式的数据，如JPEG图像数据或Json对象描述信息等。 在Redis中字符串类型的Value最多可以容纳的数据长度是512M。

| 语法                                                     | 描述                                           |
| :------------------------------------------------------- | :--------------------------------------------- |
| jedis.set(String key,String value)                       | 增加（或覆盖）数据项                           |
| jedis.setnx(String key,String value)                     | 不覆盖增加数据项（重复的不插入）               |
| jedis.setex(String ,int t,String value)                  | 增加数据项并设置有效时间                       |
| jedis.del(String key)                                    | 删除键为key的数据项                            |
| jedis.get(String key)                                    | 获取键为key对应的value                         |
| jedis.append(String key, String s)                       | 在key对应value 后边扩展字符串 s                |
| jedis.mset(String k1,String V1,String K2,String V2,…)    | 增加多个键值对                                 |
| String[] jedis.mget(String K1,String K2,…)               | 获取多个key对应的value                         |
| **`jedis.del(new String[](String K1,String K2,.... ))`** | 删除多个key对应的数据项                        |
| String jedis.getSet(String key,String value)             | 获取key对应value并更新value                    |
| String jedis.getrang(String key , int i, int j)          | 获取key对应value第i到j字符 ，从0开始，包头包尾 |

#### 3.jedis中对整数和[浮点数](https://so.csdn.net/so/search?q=浮点数&spm=1001.2101.3001.7020)操作

| 语法                             | 描述                  |
| :------------------------------- | :-------------------- |
| jedis.incr(String key)           | 将key对应的value 加1  |
| jedis.incrBy(String key,int n)   | 将key对应的value 加 n |
| jedis.decr(String key)           | 将key对应的value 减1  |
| jedis.decrBy(String key , int n) | 将key对应的value 减 n |

#### 4. jedis中对列表（list）操作

在Redis中，List类型是按照插入顺序排序的字符串链表。和数据结构中的普通链表 一样，我们可以在其头部(left)和尾部(right)添加新的元素。在插入时，如果该键并不存在，Redis将为该键创建一个新的链表。如果链表中所有的元素均被移除，那么该键也将会被从数据库中删除。List中可以包含的最大元素数量是 4294967295。
从元素插入和删除的效率视角来看，如果我们是在链表的两头插入或删除元素，这将 会是非常高效的操作，即使链表中已经存储了百万条记录，该操作也可以在常量时间内完成。然而需要说明的是，如果元素插入或删除操作是作用于链表中间，那将会是非常低效的。

![在这里插入图片描述](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202304171632903.png)

> list 元素的下表从0开始

| 语法                                                 | 描述                                                         |
| :--------------------------------------------------- | :----------------------------------------------------------- |
| `jedis.lpush(String key, String v1, String v2,....)` | 添加一个List , 注意：如果已经有该List对应的key, 则按顺序在左边追加 一个或多个 |
| jedis.rpush(String key , String vn)                  | key对应list右边插入元素                                      |
| jedis.lrange(String key,int i,int j)                 | 获取key对应list区间[i,j]的元素，注：从左边0开始，包头包尾    |
| jedis.lrem(String key,int n , String val)            | 删除list中 n个元素val                                        |
| jedis.ltrim(String key,int i,int j)                  | 删除list区间[i,j] 之外的元素                                 |
| jedis.lpop(String key)                               | key对应list ,左弹出栈一个元素                                |
| jedis.rpop(String key)                               | key对应list ,右弹出栈一个元素                                |
| jedis.lset(String key,int index,String val)          | 修改key对应的list指定下标index的元素                         |
| jedis.llen(String key)                               | 获取key对应list的长度                                        |
| jedis.lindex(String key,int index)                   | 获取key对应list下标为index的元素                             |
| jedis.sort(String key)                               | 把key对应list里边的元素从小到大排序 （后边详细介绍）         |

#### 5. jedis 集合set 操作

在Redis中，我们可以将Set类型看作为**没有排序的字符集合**，和List类型一样，也可以在该类型的数据值上执行添加、删除或判断某一元素是否存在等操作。需要 说明的是，这些操作的时间是常量时间。Set可包含的最大元素数是4294967295。
和List类型不同的是，**Set集合中不允许出现重复的元素**。和List类型相比，Set类型在功能上还存在着一个非常重要的特性，即在服务器端完成多个Sets之间的聚合计 算操作，如unions、intersections和differences（就是交集并集那些了）。由于这些操作均在服务端完成， 因此效率极高，而且也节省了大量的网络IO开销

> set 的方法都以s开头

| 语法                                              | 操作                            |
| :------------------------------------------------ | :------------------------------ |
| jedis.sadd(String key,String v1,String v2,…)      | 添加一个set                     |
| jedis.smenbers(String key)                        | 获取key对应set的所有元素        |
| jedis.srem(String key,String val)                 | 删除集合key中值为val的元素      |
| jedis.srem(String key, Sting v1, String v2,…)     | 删除值为v1, v2 , …的元素        |
| jedis.spop(String key)                            | 随机弹出栈set里的一个元素       |
| jedis.scared(String key)                          | 获取set元素个数                 |
| jedis.smove(String key1, String key2, String val) | 将元素val从集合key1中移到key2中 |
| jedis.sinter(String key1, String key2)            | 获取集合key1和集合key2的交集    |
| jedis.sunion(String key1, String key2)            | 获取集合key1和集合key2的并集    |
| jedis.sdiff(String key1, String key2)             | 获取集合key1和集合key2的差集    |

#### 6.jedis中有序集合Zsort

Sorted-Sets和Sets类型极为相似，它们都是字符串的集合，都**不允许重复的成员出现在一个Set中**。它们之间的**主要差别是Sorted-Sets中的每一个成员都会有一个分数(score)与之关联**，Redis正是通过分数来为集合中的成员进行从小到大的排序。然 而需要额外指出的是，尽管Sorted-Sets中的成员必须是唯一的，但是分数(score) 却是可以重复的。
在Sorted-Set中添加、删除或更新一个成员都是非常快速的操作，其时间复杂度为集合中成员数量的对数。由于Sorted-Sets中的成员在集合中的位置是有序的，因此，即便是访问位于集合中部的成员也仍然是非常高效的。事实上，Redis所具有的这一特征在很多其它类型的数据库中是很难实现的，换句话说，在该点上要想达到和Redis同样的高效，在其它数据库中进行建模是非常困难的。
例如：游戏排名、微博热点话题等使用场景。

| 语法                                             | 描述                                            |
| :----------------------------------------------- | :---------------------------------------------- |
| jedis.zadd(String key,Map map)                   | 添加一个ZSet                                    |
| jedis.hset(String key,int score , int val)       | 往 ZSet插入一个元素（Score-Val）                |
| jedis.zrange(String key, int i , int j)          | 获取ZSet 里下表[i,j] 区间元素Val                |
| jedis. zrangeWithScore(String key,int i , int j) | 获取ZSet 里下表[i,j] 区间元素Score - Val        |
| jedis.zrangeByScore(String , int i , int j)      | 获取ZSet里score[i,j]分数区间的元素（Score-Val） |
| jeids.zscore(String key,String value)            | 获取ZSet里value元素的Score                      |
| jedis.zrank(String key,String value)             | 获取ZSet里value元素的score的排名                |
| jedis.zrem(String key,String value)              | 删除ZSet里的value元素                           |
| jedis.zcard(String key)                          | 获取ZSet的元素个数                              |
| jedis.zcount(String key , int i ,int j)          | 获取ZSet总score在[i,j]区间的元素个数            |
| jedis.zincrby(String key,int n , String value)   | 把ZSet中value元素的score+=n                     |

#### 7. jedis中 哈希（[Hash](https://so.csdn.net/so/search?q=Hash&spm=1001.2101.3001.7020)）操作

Redis中的Hashes类型可以看成具有String Key和String Value的map容器。所以该类型非常适合于存储值对象的信息。如Username、Password和Age等。如果Hash中包含很少的字段，那么该类型的数据也将仅占用很少的磁盘空间。每一个Hash可以存储4294967295个键值对。
![在这里插入图片描述](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202304171632074.png)

> 规律： 哈希的 方法 都以 h 开头，含有m字符的一般是多个的， （multiple： 多个的）

| 语法                                              | 描述                           |
| :------------------------------------------------ | :----------------------------- |
| jedis.hmset(String key,Map map)                   | 添加一个Hash                   |
| jedis.hset(String key , String key, String value) | 向Hash中插入一个元素（K-V）    |
| jedis.hgetAll(String key)                         | 获取Hash的所有（K-V） 元素     |
| jedis.hkeys（String key）                         | 获取Hash所有元素的key          |
| jedis.hvals(String key)                           | 获取Hash所有元素 的value       |
| jedis.hincrBy(String key , String k, int i)       | 把Hash中对应的k元素的值 val+=i |
| jedis.hdecrBy(String key,String k, int i)         | 把Hash中对应的k元素的值 val-=i |
| jedis.hdel(String key , String k1, String k2,…)   | 从Hash中删除一个或多个元素     |
| jedis.hlen(String key)                            | 获取Hash中元素的个数           |
| jedis.hexists(String key,String K1)               | 判断Hash中是否存在K1对应的元素 |
| jedis.hmget(String key,String K1,String K2)       | 获取Hash中一个或多个元素value  |

#### 8. 排序操作

使用排序， 首先需要生成一个排序对象

```java
SortingParams sortingParams = new SortingParams();
```

| 语法                                          | 描述                 |
| :-------------------------------------------- | :------------------- |
| jedis.sort(String key,sortingParams.alpha())  | 队列按首字母a-z 排序 |
| jedis.sort(String key, sortingParams.asc() )  | 队列按数字升序排列   |
| jedis.sort(String key , sortingParams.desc()) | 队列按数字降序排列   |

使用示例：

```java
 Jedis jedis = JedisPoolUtils.getJedis();
 SortingParams sortingParams = new SortingParams();
 List<String> sort = jedis.sort("list02", sortingParams.desc());
```

> 这里排序指的是返回的sort是有序的，而之前的list02 依然是以前的顺序。