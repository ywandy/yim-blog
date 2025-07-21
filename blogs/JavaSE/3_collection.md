---
title: Java基础3-集合
tags: 
  - Java基础
date: 2024-02-20 20:39:49
categories:
  - Java基础
---

## 集合

### 前言

集合存放的都是对象，即引用数据类型，基础数据类型不能放到集合中。

-  List，Set，Map都是接口，前两个继承至Collection接口，Map为独立接口

- List下有ArrayList、Vector、LinkedList

- Set下有HashSet、LinkedHashSet、TreeSet

- Map下有Hashtable、LinkedHashMap、HashMap、TreeMap

- Collection接口下还有个Queue接口，有PriorityQueue类。

---

### List

ArrayList、Vector、LinkedList

特点：

无序、可重复、可null

ArrayList：线程不安全，增删改查快，数据结构是单向链表，初始化长度一般是10，扩容倍数是1.5

Vector：线程安全，但查询比ArrayList慢很多；可以设置增长因子，而ArrayList不可以。

LinkedList：线程不安全，增删改查快，双向链表，其他特性与ArrayList一致



==fail-fast机制==

在for中删除元素就会触发fail-fast避免被修改

==fail-save机制==

为避免fail-fast可使用fail-save集合类 fail-save的原理是不是直接在原有集合上操作而是将原有集合进行拷贝在拷贝上的集合进行增删改的操作，完成后将拷贝的集合引用到原集合上 使用了读写分离的思想

---

### Set

HashSet、TreeSet、LinkedHashSet

特点：

无序、不可重复、可null

HashSet：在JDK1.7之前，基于哈希表实现，增删改查快；1.8之后是链表+二叉树，链表长度>8，将链表换成二叉树

TreeSet：基于红黑树实现，有排序功能，有序

排序：自然排序，比较器排序。可以自定义规则，默认是自然排序，按hashCode值升序插入

Compare(Obj o1, Obj o2);

LinkedHashSet：线程不安全，双向链表，但是有序



### Map

HashMap、TreeMap、LinkedHashMap、ConcurrentHashMap

特点：

数组，数组内部是一个链表；无序、key不可重复、重复插入value会被覆盖，key、value可null

HashMap：

1.数组的每个元素都是一个链表（或红黑树，当链表长度超过一定阈值时会转化为红黑树）的头节点

2.当 `HashMap` 中的元素数量超过数组大小的一定比例（默认为0.75）时，`HashMap` 会进行扩容。扩容操作会创建一个新的数组，其大小是原数组大小的两倍，然后将原数组中的元素重新哈希并插入到新数组中

3.如果链表长度超过一定的阈值（默认为8），链表会转化为红黑树，

哈希函数在HashMap中的作用

1.确定存储位置：用对应的hashCode%arraySize确定存储位置

2.快速查找：每个key都有对应的hash值。

3.碰撞处理：如果有重复，则用用开放寻址法处理，放到红黑树的最后一个

4.均匀分布：尽可能把不同的键映射到不同的索引上，并且减少碰撞的几率



TreeMap：基于红黑树实现，可以对键进行排序

LinkedHashMap：按照插入的顺序排序，双向链表

ConcurrentHashMap: 

1.在JDK 1.8中，ConcurrentHashMap的实现方式进行了改进，使用分段锁（思想）和“CAS+Synchronized”的机制来保证线程安全。

2.如何保证fail-safe

在 JDK 1.8 中，ConcurrentHashMap 中的 Segment 被移除了，取而代之的是使用类似于cas+synchronized的机制来实现并发访问。在遍历 ConcurrentHashMap 时，只需要获取每个桶的头结点即可，因为每个桶的头结点是原子更新的，不会被其他线程修改。这个设计允许多个线程同时修改不同的桶，这减少了并发修改的概率，从而降低了冲突和数据不一致的可能性。



==HashMap与HashTable的区别==

1.HashMap线程不安全，HashTable线程安全

2.HashMap的初始容量默认为16，Hashtable的初始容量默认为11，两者的填充因子默认都是0.75。

3.HashMap和Hashtable的底层实现都是数组+链表结构，但它们计算hash的方式不同

4.HashMap允许null作为键或值，Hashtable不允许null作为键或值



==HashMap在并发插入中会出现什么问题==

1.会导致cpu占用短暂飙升

2.在put操作时会出现覆盖问题

3.HashMap在扩容的时候，会将元素插入链表头部 
