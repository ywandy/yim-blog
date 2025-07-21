---
title: JVM
tags:
  - JVM
date: 2022-02-14 22:35:05
catogeries:	
  - JVM
---

## 概述

![jvm基本架构](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202204130016018.png)

jvm（java virtual machine）是java运行不可或缺的一部分，由类加载器、运行时数据区（Runtime Data Areas）、字节码执行引擎组成

### 双亲委派模型

![](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202404151428069.png)

除了顶层的启动类加载器外，其它的类加载器都要有自己的父类加载器

父类加载失败，由子类加载器自己处理



#### 工作过程

1.一个类加载器收到了类加载的请求，它首先不会自己去尝试加载这个类，而是把这个请求委派给父类加载器去完成，每一个层次的类加载器都是如此

2.所有的加载请求最终都应该传送到顶层的启动类加载器中，只有当父加载器反馈自己无法完成这个加载请求（它的搜索范围中没有找到所需的类）时

3.子加载器才会尝试自己去加载



#### 好处

1.使得Java类随着它的类加载器一起具有一种带有优先级的层次关系，从而使得基础类得到统一，避免了多份同样字节码的加载(类的重复加载)

2.java核心api中定义类型不会被随意替换，可以防止核心API库被随意篡改。例如：通过网络传递一个名为java.lang.Integer的类，通过双亲委托模式传递到启动类 

加载器，而启动类加载器在核心Java API发现这个名字的类，发现该类已被加载，并不会重新加载网络传递的过来的java.lang.Integer，而直接返回已加载过的

Integer.class。

---

## 基本组件

### 堆

![堆](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202204182117083.png)

#### 堆跟栈的联系

栈的局部变量空间，放的是堆中对象的地址。

#### GC

![](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202204191947882.png)

可达性分析算法可以解决gc root的循环引用问题

当eden区存放满了之后，字节码执行引擎就会开启一个线程，执行minor gc。

在方法区里面找能作为gc root的局部变量，根据这个局部变量找到引用的堆中的实际变量，直至这个变量不再引用任何对象（不是从gc root出发的对象都是垃圾对象），并把非垃圾对象复制一份放到survivor区，并清空eden区

（对象一旦被挪到survivor区，他们的分代年龄就会+1）

当程序继续运行，如果eden区对象又满了，算法就会把非空的survivor区也一并回收。最后把存活的对象复制到另一份survivor区中

当对象的分代年龄到达15，则会在最后一次minor gc后把此对象放到老年代中。老年代一般是连接池、spring的bean、缓存对象、静态对象...

当老年代放满了之后，jvm会做一个full gc，直接把整个堆中的对象回收

jvm调优，就是为了减少gc的次数以及时间（包括full gc的执行次数以及执行时间）

==所以full gc是什么时候触发的？跟oom有啥关系？==

老年代放满了，产生full gc；如果老年代放满了，程序还一直在不停地放对象，就会产生oom。

==为什么jvm的设计人员要设计STW（stop the world）机制？==

先单线程把gc做了，把整个容器的性能提高。如果不这样做，jvm堆里面的对象很难判断哪些是垃圾哪些是非垃圾

==大对象、长期存活的对象会直接进入老年代==

![](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202211142116507.png)

![image-20221114211828242](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202211142118274.png)

##### 对象头

![](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202211012238878.png)



### 栈（FILO First In Last Of 先进后出）

每一个线程开始执行，java虚拟机就会分配一个内存空间（栈），用来存放局部变量。

==栈帧==是线程内部方法的内存空间， 一个方法对应一块栈帧内存空间，局部变量就放在栈帧内，遵循FILO规则。

栈帧内部包含5个组件

- 局部变量表
- 操作数栈
- 动态链接
- 方法出口

### 局部变量表

放局部变量表的地方

### 操作数栈

临时存放数值的内存空间。当做运算操作时，会把栈顶2个数弹出来，做操作后重新压回栈中

### 动态链接

放的是方法内部的代码在元空间中的入口地址

### 方法出口

记录的是主线程代码的执行位置、返回值等

补充：javap -c test.class 反编译

Main方法的局部变量是一个对象， 是一块内存空间，是一个对象在堆中的地址（引用）

### 本地方法栈

native修饰，访问C++的接口

### 方法区（元空间）

使用的是直接内存（物理内存），放的是常量、静态变量、类信息

方法区的类信息，存的是栈中实例的引用（地址）



### 程序计数器

记录代码执行的位置，每个线程独有的



栈、本地方法栈、程序计数器是每个线程私有的

堆、方法区是公有的

---

## 垃圾回收算法

### 1.Mark-Sweep(标记清除)

仅仅把标记的可回收对象回收了

![标记清除](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202212152256627.png)

### 2.复制

内存空间一切为二，把上面的可回收复制到下面

![复制算法](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202212152256880.png)

### 3.标记整理

![标记整理算法](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202212152256082.png)

![区别](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202212152256253.png)

---

## 垃圾收集器

![垃圾收集器](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202212152256465.png)

### Java8以及之前的版本--分代模型

新生代↓

- ParNew
- Serial
- Parallel Scavenge

老年代↓

- CMS
- Serial Old
- Parallel Old

#### 1.使用-XX:UseSerialGC，新生代使用Serial GC，老年代自动使用Serial Old GC（串行的收集器）

![Serial收集器](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202212152256686.png)

用户线程访问程序，当新生代要做垃圾回收的时候，线程终止的点叫做Safepoint，开启GC线程回收垃圾，使用Serial垃圾收集器，采用复制算法，串行收集垃圾；老年代使用Serial Old垃圾收集器，采用标记整理算法，暂停所有用户线程（STW），串行做垃圾收集。

#### 2.==Java8默认==使用-XX:UseParallelGC / -XX:UseParallelOldGC，新生代使用Parallel Scavenge GC，老年代使用Parallel Old GC（并行的收集器）

![ParallelGC](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202212152256318.png)

用户线程访问程序，当新生代要做垃圾回收的时候，线程终止的点叫做Safepoint，开启GC线程回收垃圾，使用Parallel Scavenge垃圾收集器，采用复制算法，并行收集垃圾；老年代使用Parallel Old垃圾收集器，采用标记整理算法，暂停所有用户线程（STW），并行做垃圾收集。

#### 3.使用-XX:UseConcMarkSweepGC，新生代使用ParNew GC，老年代使用CMS GC与Serial Old GC的收集器组合，Serial Old将作为CMS出错的后备收集器

新生代

![ParNewGC](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202212152256629.png)

老年代

![CMS+SerialOld](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202212152256833.png)

CMS(ConcurrentMarkSweep)步骤如下：

1.初始标记（STW）

gc root开始标记第一层对象

2.并发标记

顺着【gc root->第一层对象】继续标记后续的对象

3.重新标记（STW）

由于并发标记时，有用户线程的干扰，重新标记的作用是==重新把错标、漏标的垃圾重新标记==

4.并发清理

进行垃圾回收



==CMS什么时候有可能出问题==

有可能产生碎片，有新的对象进来老年代，老年代满了->垃圾处理程序出现异常，Serial Old就进场，执行STW机制

### Java9以及以后的版本--分区模型

- G1(Java9)
- ZGC(Java11)
- Epsilon(Java11)
- Shenandoah

---

## 问题分析

### 1.Java虚拟机为什么要设计一个程序计数器？

当一个程序被挂起然后再重新执行的时候，程序技术器的作用就是让虚拟机知道，挂起之前的代码执行到哪里

### 2.程序计数器的值是怎么修改的？

字节码执行引擎修改程序计数器的值

### 3.能不能对jvm调优，使其几乎不发生full gc?

![jvm参数](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202211182021367.png)

