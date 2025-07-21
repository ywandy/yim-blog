---
title: 多线程专题
tags: 
  - 面试
date: 2023-05-17 09:00:00
categories:
  - 面试
---

## 1. 线程的创建方式有几种

面试官:说下java中创建线程的方式有几种，以及推荐使用哪一种

我:创建线程的方式有两种，一种是继承Thread，一种是实现Runable

在这里推荐使用实现Runable接口，因为java是单继承的，一个类继承了Thread将无法继承其他的类，而java可以实现多个接口，所有如果实现了Runable接口后，还可以实现其他的接口

### 1.1 线程的几种状态

面试官:说下Thread的几种状态以及他们之间的一个状态转换

我:Thread有五种状态，一般说六种的都是错误的，因为这个是从java类中明确标明了，

#### 1.1.1 新建状态(NEW)

当程序使用 new 关键字创建了一个线程之后，该线程就处于新建状态，此时仅由 JVM 为其分配内存， 并初始化其成员变量的值

#### 1.1.2 就绪状态(RUNNABLE)

 当线程对象调用了 start()方法之后，该线程处于就绪状态，Java 虚拟机会为其创建方法调用栈和程序计

数器，等待调度运行

#### 1.1.3 运行状态(RUNNING)

如果处于就绪状态的线程获得了 CPU，开始执行 run()方法的线程执行体，则该线程处于运行状态

#### 1.1.4 阻塞状态(BLOCKED)

阻塞状态是指线程因为某种原因放弃了 cpu 使用权，也即让出了 cpu timeslice，暂时停止运行。直到 线程进入可运行(runnable)状态，才有机会再次获得 cpu timeslice 转到运行(running)状

##### 等待阻塞

运行(running)的线程执行 o.wait()方法，JVM 会把该线程放入等待队列(waitting queue)中

##### 同步阻塞

运行(running)的线程在获取对象的同步锁时，若该同步锁被别的线程占用，则 JVM 会把该线程放入锁 池(lock pool)中。

##### 其他阻塞

运行(running)的线程执行 Thread.sleep(long ms)或 t.join()方法，或者发出了 I/O 请求时，JVM 会把该线程置为阻塞状态。当 sleep()状态超时、join()等待线程终止或者超时、或者 I/O 

#### 1.1.5 线程死亡(DEAD)

线程会以下面三种方式结束，结束后就是死亡状态。 

##### 正常结束

run()或 call()方法执行完成，线程正常结束

##### 异常结束

线程抛出一个未捕获的 Exception 或 Error

##### 调用 stop

直接调用该线程的 stop()方法来结束该线程—该方法通常容易导致死锁，不推荐使用

### 1.2 线程之间的状态转换

![](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202305231537606.png)

### 1.3 sleep与wait 区别

1. 对于 sleep()方法，我们首先要知道该方法是属于 Thread 类中的，而 wait()方法，则是属于Object 类中的。

2. sleep()方法导致了程序暂停执行指定的时间，让出 cpu 该其他线程，但是他的监控状态依然保持 者，当指定的时间到了又会自动恢复运行状态。

3. 在调用 sleep()方法的过程中，线程不会释放对象锁。

4. 而当调用 wait()方法的时候，线程会放弃对象锁，进入等待此对象的等待锁定池，只有针对此对象

   调用 notify()方法后本线程才进入对象锁定池准备获取对象锁进入运行状态

---

## 2. 为什么要使用线程池

面试官:为什么要使用线程池，使用线程池的好处是什么

我:线程池的工作主要是控制运行线程的数量，处理过程中将任务放入队列，然后在线程创建后启动这些任务，如果线程数量超过了最大数量，超出数量的线程排队等候，等其他线程执行完毕，再从队列中取出任务来执行。

![](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202305231538731.png)

特点：线程复用；控制最大并发数；管理线程。

1. 降低资源消耗。
2. 提高响应速度。
3. 提高线程的可管理性。

### 2.1 说下线程池的执行流程

面试官:看过线程池的源码，说下线程池的执行流程

我：

![](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202305231539664.png)

### 2.2 线程池常用参数

面试官:说下线程池的常用参数

我:线程池常用的参数如下

```
corePoolSize:核心线程数量，会一直存在，除非allowCoreThreadTimeOut设置为true
maximumPoolSize:线程池允许的最大线程池数量
keepAliveTime:线程数量超过corePoolSize，空闲线程的最大超时时间
unit:超时时间的单位
workQueue:工作队列，保存未执行的Runnable任务
threadFactory:创建线程的工厂类
handler:当线程已满，工作队列也满了的时候，会被调用。被用来实现各种拒绝策略。
```

### 2.3 为什么不建议使用Executors静态工厂构建线程池

面试官:阿里巴巴Java开发手册，明确指出不允许使用Executors静态工厂构建线程池 

#### 2.3.1 Executors 是什么

Executors工具类的不同方法按照我们的需求创建了不同的线程池，来满足业务的需求,有以下几种创建工具类的方式

1. newFixedThreadPool(int Threads)创建固定数目的线程池
2. newSingleThreadPoolExecutor():创建一个单线程化的Executor
3. newCacheThreadPool():创建一个可缓存的线程池，调用execute将重用以前构成的线程(如果线程可用)，如果没有可用的线程，则创建一个新线程并添加到池中。终止并从缓存中移出那些已有60秒钟未被使用的线程。
4. newScheduledThreadPool(int corePoolSize)创建一个支持定时及周期性的任务执行的线程池

#### 2.3.2 为什么不允许使用Executors创建线程

线程池不允许使用Executors去创建，而是通过ThreadPoolExecutor的方式，这样的处理方式让写的目的是为了更加明确线程池的运行规则，规避资源耗尽的风险

![](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202305231600067.png)

#### 2.3.3 合理地配置线程池

面试官:如何合理的定制线程池 

我:

1. CPU密集型任务:应配置尽可能小的线程，如配置Ncpu+1个线程的线程池。

2. IO密集型任务:线程并不是一直在执行任务，则应配置尽可能多的线程，如2*Ncpu。

3. 混合型的任务，如果可以拆分，将其拆分成一个CPU密集型任务和一个IO密集型任务，只要这两个任务执行的时间相差不是太大，那么分解后执行的吞吐量将高于串行执行的吞吐量。如果这两个任务执行时间相差太大，则没必要进行分解。可以通过Runtime.getRuntime().availableProcessors() 方法获得当前设备的CPU个数。

---

## 3. Synchronized的常见用法以及区别

面试官:能说下Synchronized的用法以及互相之间的区别

![](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202305231601663.png)

我:Synchronized是jvm提供的可重入的互斥锁，用法有以下几种

1. 修饰实例方法，作用于当前实例加锁，进入同步代码前要获得当前实例的锁。
2. 修饰静态方法，作用于当前类对象加锁，进入同步代码前要获得当前类对象的锁。 

3. 修饰代码块，指定加锁对象，对给定对象加锁，进入同步代码库前要获得给定对象。

### 3.1 Synchronized的实现原理?

面试官:能说下Synchronized的实现原理吗

我: synchronized同步块使用了monitorenter和monitorexit指令实现同步，这两个指令，本质上都是 对一个对象的监视器(monitor)进行获取，这个过程是排他的，也就是说同一时刻只能有一个线程获取 到由synchronized所保护对象的监视器。

线程执行到monitorenter指令时，会尝试获取对象所对应的monitor所有权，也就是尝试获取对象的锁，而执行monitor exit，就是释放monitor的所有权。

### 3.2 能说下Synchronized的效率真的很低吗

面试官:都说Synchronized锁的效率很低能说下原因吗

我:Synchronized在jdk1.6以下效率很低的，因为直接使用了重量级锁，而到了1.6及以上，经过编译 器优化以及jvm的锁优化效率几乎接近到了lock的水平，但是Synchronized用起来是比较简单，不需要 关心锁释放问题，所以大多数场景下可以直接使用Synchronized

JDK1.6对锁的实现引入了大量的优化，如自旋锁、适应性自旋锁、锁消除、锁粗化、偏向锁、轻量级锁 等技术来减少锁操作的开销。

#### 3.2.1 锁消除

为了保证数据的完整性，我们在进行操作时需要对这部分操作进行同步控制，但是在有些情况下，JVM 检测到不可能存在共享数据竞争，这是JVM会对这些同步锁进行锁消除，锁消除的依据是逃逸分析的数 据支持。

#### 3.2.2 锁粗化

就是将多个连续的加锁、解锁操作连接在一起，扩展成一个范围更大的锁们知道在使用同步锁的时候，需要让同步块的作用范围尽可能小，仅在共享数据的实际作用域中才进行同步。这样做的目的是为了使需要同步的操作数量尽可能缩小，如果存在锁竞争，那么等待锁的线程也能尽快拿到锁。

但是如果一系列的连续加锁解锁操作，可能会导致不必要的性能损耗，所以引入锁粗化的概念

### 3.3 能说下Synchronized的升级过程吗

面试官:能说下Synchronized锁的升级过程吗

我:Synchronized经过优化后，升级步骤如下:偏向锁、无锁、轻量级锁、重量级锁

![](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202305231603518.png)

具体升级步骤如下图

![](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202305231604834.png)

### 3.4 说下Synchronized和Lock的区别吗

面试官:能说下Synchronized和Lock的区别吗? 

我:

#### 3.4.1 底层工作机制不同

synchronized关键字是属于JVM层面实现的，它的底层是通过monitor对象来完成的，其中 wait/notify等方法也依赖monitor对象只有在同步代码块和同步方法中才能调用wait/notify等方 法。

Lock与synchronized不同，它是一个具体的类，它是java api层面的锁

#### 3.4.2 使用方式的区别

Synchronized关键字运行后是不需要用户去手动释放锁的，在synchronized代码执行成功后系统 会自动让线程释放对锁的占据。

ReentrantLock锁运行后需要用户手动去释放锁，如若用户没有主动去释放锁，就有可能导致出现 死锁现象。ReentrantLock需要使用lock()和unlock()方法配合try finally语句块来完成。

#### 3.4.3 是否可中断

synchronized不能中断，除非抛出异常或者正常运行完成。 

ReetrantLock可中断，无影响。

#### 3.4.4 是否公平

 synchronized是一个非公平锁

ReetrantLock可以实现公平也可以实现非公平 

#### 3.4.5 是否支持条件唤醒

synchronized不支持多条件

如果使用ReentrantLock来实现分组唤醒需要唤醒的线程们，就可以精确唤醒，不会如synchronized样，要么随机唤醒一个，要么唤醒全部线程。

---

## 4. 为什么要使用多线程编程

面试官:为什么要使用多线程编程呢?使用后有什么有点以及缺点

我:

1. 多线程编程可以发挥出现多核CPU的优势
2. 提高系统的性能以及响应时间
3. 多线程编程需要时刻关注线程安全问题

### 4.1 进程线程携程的区别

面试官:能说下进程线程以及协程的区别吗? 我:进程是资源分配的最小单位，线程是CPU调度的最小单位 一个进程可以包含很多个线程，进程之间不能共享数据，而线程之间可以进程共享数据

协程是一种用户态的轻量级线程，协程的调度完全由用户控制。协程拥有自己的寄存器上下文和栈，协 程避免了无意义的调度，由此可以提高性能，但也因此，程序员必须自己承担调度的责任，同时协程也失去了标准线程使用多CPU的能力。

### 4.2 如何正确的停止一个线程

面试官:如何正确的停止一个线程

我:使用interrupt方法中断线程，

interrupt()方法的使用效果并不像for+break语句那样，马上就停止循环，调用interrupt方法是在当前线程中打了一个停止标志，并不是真的停止线程，而停止线程是交给线程自己来停止interrupt只是一个停止信号

### 4.3 在多线程情况下如何保证线程安全

面试官:多线程的情况下如何保证线程安全?

我:线程安全里面有几个安全等级

#### 4.3.1 线程安全等级

##### 不可变

在java语言中，不可变的对象一定是线程安全的，无论是对象的方法实现还是方法的调用者，都不需要 再采取任何的线程安全保障措施。如final关键字修饰的数据不可修改，可靠性最高。

##### 绝对线程安全

绝对的线程安全完全满足Brian GoetZ给出的线程安全的定义，这个定义其实是很严格的，一个类要达 到“不管运行时环境如何，调用者都不需要任何额外的同步措施”通常需要付出很大的代价。

##### 相对线程安全

相对线程安全就是我们通常意义上所讲的一个类是“线程安全”的。

它需要保证对这个对象单独的操作是线程安全的，我们在调用的时候不需要做额外的保障措施，但是对于一些特定顺序的连续调用，就可能需要在调用端使用额外的同步手段来保证调用的正确性。

在java语言中，大部分的线程安全类都属于相对线程安全的，例如Vector、HashTable、Collections的 synchronizedCollection()方法保证的集合。

##### 线程兼容

线程兼容就是我们通常意义上所讲的一个类不是线程安全的。

线程兼容是指对象本身并不是线程安全的，但是可以通过在调用端正确地使用同步手段来保证对象在并发环境下可以安全地使用。

Java API中大部分的类都是属于线程兼容的。如与前面的Vector和HashTable相对应的集合类ArrayList 和HashMap等。

##### 线程对立

线程对立是指无论调用端是否采取了同步策略，都无法在多线程环境中并发使用的代码，由于java语言天生就具有多线程特性，线程对立这种排斥多线程的代码是很少出现的。

一个线程对立的例子是Thread类的supend()和resume()方法。如果有两个线程同时持有一个线程对 象，一个尝试去中断线程，另一个尝试去恢复线程，如果并发进行的话，无论调用时是否进行了同步， 目标线程都有死锁风险。正因此如此，这两个方法已经被废弃啦。

#### 4.3.2 实现线程安全的手段

##### 线程栈封闭

什么是栈封闭呢?简单的说就是局部变量。多个线程访问一个方法，此方法中的局部变量都会被拷贝一份到线程栈中。所以局部变量是不会被多个线程所共享的，也就不会出现并发问题。所以能用局部变量就别用全局的变量，全局变量容易引起并发问题。

##### 无状态的类 

没有任何成员变量的类，就叫无状态的类，这种类一定是线程安全的。如果这个类的方法参数中使用了对象，也是线程安全的吗?比如

```java
public class StatelessClass {
    public int service(int a,int b){
				return a+b; 
    }
    public void serviceUser(UserVo user){
        //do sth user
		} 
}
```

当然也是，因为多线程下的使用，固然user这个对象的实例会不正常，但是对于StatelessClass这个类 的对象实例来说，它并不持有UserVo的对象实例，它自己并不会有问题，有问题的是UserVo这个类， 而非StatelessClass本身。

##### 加final关键字

加final关键字，对于一个类，所有的成员变量应该是私有的，同样的只要有可能，所有的成员变 量应该加上final关键字，但是加上final，要注意如果成员变量又是一个对象时，这个对象所对应 的类也要是不可变，才能保证整个类是不可变的。参见代码

```java
/**
* 类不可变 */
public class ImmutableClass {
    private final int a;
    private final UserVo user = new UserVo();//不安全
    public int getA() {
        return a;
    }
    public UserVo getUser() {
       return user;
    }
    public ImmutableClass(int a) {
        this.a = a;
    }
    public static class User{
       	private int age;
        public int getAge() {
            return age;
    		}
        public void setAge(int age) {
           this.age = age;
   			} 
    }
}
```

##### 不提供返回值 

根本就不提供任何可供修改成员变量的地方，同时成员变量也不作为方法的返回值。

```java
/**
 * 类不可变--事实不可变 
 */
public class ImmutableClassToo {
    private final List<Integer> list = new ArrayList<>(3);
    public ImmutableClassToo() {
        list.add(1);
        list.add(2);
        list.add(3);
    }
    public boolean isContain(int i){
        return list.contains(i);
    }
}
```

但是要注意，一旦类的成员变量中有对象，上述的final关键字保证不可变并不能保证类的安全性，为 何?因为在多线程下，虽然对象的引用不可变，但是对象在堆上的实例是有可能被多个线程同时修改 的，没有正确处理的情况下，对象实例在堆中的数据是不可预知的。这就牵涉到了如何安全的发布对象 这个问题。

##### 安全发布

类中持有的成员变量，如果是基本类型，发布出去，并没有关系，因为发布出去的其实是这个变量的一个副本，参见代码

```java
/**
 * 演示基本类型的发布 
 */
public class SafePublish {
    private int i;
    public SafePublish() {
        i = 2;
    }
  	public int getI() {
        return i;
  	}
  	public static void main(String[] args) {
        SafePublish safePublish = new SafePublish();
        int j = safePublish.getI();
        System.out.println("before j="+j);
        j = 3;
        System.out.println("after j="+j);
        System.out.println("getI = "+safePublish.getI());
    }
}    
```

但是如果类中持有的成员变量是对象的引用，如果这个成员对象不是线程安全的，通过get等方法发布出去，会造成这个成员对象本身持有的数据在多线程下不正确的修改，从而造成整个类线程不安全的问 题。参见以下代码可以看见:

```java
/**
* 不安全的发布 */
public class UnSafePublish {
    private List<Integer> list = new ArrayList<>(3);
    public UnSafePublish() {
        list.add(1);
        list.add(2);
        list.add(3);
    }
    public List getList() {
        return list;
		}
    public static void main(String[] args) {
        UnSafePublish unSafePublish = new UnSafePublish();
        List<Integer> list = unSafePublish.getList();
        System.out.println(list);
        list.add(4);
        System.out.println(list);
        System.out.println(unSafePublish.getList());
		} 
}
```

这个list发布出去后，是可以被外部线程之间修改，那么在多个线程同时修改的情况下不安全问题是肯定存在的，怎么修正这个问题呢?我们在发布这对象的时候，就应该用线程安全的方式包装这个对象。 我们将list用Collections.synchronizedList进行包装以后，无论多少线程使用这个list，就都是线程安全 的了。

```java
/**
* 安全的发布 */
public class SafePublishToo {
    private List<Integer> list
            = Collections.synchronizedList(new ArrayList<>(3));
    public SafePublishToo() {
        list.add(1);
        list.add(2);
        list.add(3);
    }
    public List getList() {
        return list;
    }
 		public static void main(String[] args) {
        SafePublishToo safePublishToo = new SafePublishToo();
        List<Integer> list = safePublishToo.getList();
        System.out.println(list);
        list.add(4);
        System.out.println(list);
        System.out.println(safePublishToo.getList());
		}
}
```

##### TheadLocal方式

ThreadLocal是实现线程封闭的最好方法，ThreadLocal内部维护了一个Map，Map的key是每个线程的 名称，而Map的值就是我们要封闭的对象。每个线程中的对象都对应着Map中一个值，也就是 ThreadLocal利用Map实现了对象的线程封闭。

### 4.4 讲一下volatile关键字的作用

面试官:说下volatile关键字的作用

我:volatile能够保证变量在多线程环境下执行的可见行以及有序性，但是不保证原子性，并通过引入内存屏障，防止volatile附近的变量执行重排

### 4.5 如何排查一个死锁

面试官:如何排查一个死锁呢? 

我:死锁是一种由于多个进程竞争资源而陷入的一种僵局，若无外力作用，所有进程都将无法向前推进。

![](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202305231635165.png)

#### 4.5.1 死锁的四个必要条件?

1、互斥条件:一个资源每次只能被一个线程使用

2、请求与保持条件:一个线程程因请求资源而阻塞时,对已获得的资源保持不放

3、不剥夺条件:进程已获得的资源,在末使用完之前,不能强行剥夺

4、循环等待条件:若干线程之间形成一种头尾相接的循环等待资源关系

#### 4.5.2 写一个死锁的例子



---

## 5. 什么是CAS

面试官:什么是CAS，CAS的原理是什么

CAS(compare and swap)的缩写，中文翻译成比较并交换

![](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202305231655165.png)

CAS 不通过JVM,直接利用java本地方法JNI(Java Native Interface为JAVA本地调用),直接调用CPU的 cmpxchg (是汇编指令)指令。

利用CPU的CAS指令，同时借助JNI来完成Java的非阻塞算法,实现原子操作，其它原子操作都是利用类 似的特性完成的。

整个JUC包都是建立在CAS之上的，因此对于synchronized阻塞算法，J.U.C在性能上有了很大的提升。 CAS是项乐观锁技术，当多个线程尝试使用CAS同时更新同一个变量时，只有其中一个线程能更新变量

的值，而其它线程都失败，失败的线程并不会被挂起，而是被告知这次竞争中失败，并可以再次尝试。 

### 5.1 常见的CAS操作类有哪些

CAS有3个操作数，内存值V，旧的预期值A，要修改的新值B。当且仅当预期值A和内存值V相同时，将内存值V修改为B，否则什么都不做。

#### 5.1.1 JUC包中的常用原子操作类

java.util.concurrent.atomic包提供了原子操作的类，例如

##### 1.AtomicInteger

Integer类型的CAS原子操作

##### 2.AtomicLong

Long类型的CAS原子操作

##### 3.AtomicBoolean

Boolean类型的CAS原子操作

##### 4.AtomicReference

引用类型的CAS原子操作

#### 5.1.2 AtomicInteger使用

以 AtomicInteger，它提供了支持原子操作的方法，包括

```java
//获取一个值
int get()
//获取并设置值
int getAndSet(int newValue)
//获取并进行+1操作
int getAndIncrement()
//进行+1后获取值
int incrementAndGet()
//获取并-1操作
int getAndDecrement()
//进行-1操作后在获取
int decrementAndGet()
```

我们使用 AtomicInteger 替代了 int ，这样可以确保最后的结果是 100。

```java
public class Atom_Test {
    private static AtomicInteger i = new AtomicInteger(0);

    public static void main(String[] args) {
        for (int j = 0; j < 100; j++) {
            new Thread() {
                public void run() {
                    try {
                        Thread.sleep(1);
                    } catch (InterruptedException e) {
                    }
                    i.getAndIncrement();
                }
            }.start();
        }
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
        }
        System.out.println(i);
    }
}
```

### 5.2 什么是原子操作

原子操作--不可中断的一个或者一系列操作, 也就是不会被线程调度机制打断的操作, 运行期间不会有任何的上下文切换(context switch)。

i++并不是一个原子操作，所以当一个线程读取它的值并加1时，另外一个线程有可能会读到之前的值，这就会引发错误。 为了解决这个问题，必须保证增加操作是原子的，在JDK1.5之前我们可以使用同步技术来做到这一点。

到JDK1.5，java.util.concurrent.atomic包提供了int和long类型的装类，它们可以自动的保证对于他们 的操作是原子的并且不需要使用同步。 

### 5.3 CAS操作的优缺点

#### 5.3.1 优点

确保对内存的读-改-写操作都是原子操作执行 

#### 5.3.2 缺点

##### CPU开销过大

在并发量比较高的情况下，如果许多线程反复尝试更新某一个变量，却又一直更新不成功，循环往复，会给CPU带来很到的压力。 

##### 不能保证代码块的原子性

CAS机制所保证的只是一个变量的原子性操作，而不能保证整个代码块的原子性。比如需要保证3个变量共同进行原子性的更新，就不得不使用synchronized了。

##### ABA问题

如果另一个线程修改V值假设原来是A，先修改成B，再修改回成A。当前线程的CAS操作无法分辨当前V值是否发生过变化。

![](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202305231722432.png)

真正要做到严谨的CAS机制，我们在compare阶段不仅需要比较内存地址V中的值是否和旧的期望值A相 同，还需要比较变量的版本号是否一致。

### 5.4 什么是乐观锁和悲观锁

#### 5.4.1 悲观锁

Java在JDK1.5之前都是靠synchronized关键字保证同步的，这种通过使用一致的锁定协议来协调对共享 状态的访问，可以确保无论哪个线程持有共享变量的锁，都采用独占的方式来访问这些变量。独占锁其 实就是一种悲观锁，所以可以说synchronized是悲观锁。

#### 5.4.2 乐观锁

乐观锁( Optimistic Locking)其实是一种思想，相对悲观锁而言，乐观锁假设认为数据一般情况下不 会造成冲突，所以在数据进行提交更新的时候，才会正式对数据的冲突与否进行检测，如果发现冲突 了，则让返回用户错误的信息，让用户决定如何去做。

### 5.5 什么是AQS

面试官:说下AQS是什么?

AbstractQueuedSynchronizer简称AQS，是一个用于构建锁和同步容器的框架。事实上concurrent 包内许多类都是基于AQS构建，例如ReentrantLock，Semaphore，CountDownLatch， ReentrantReadWriteLock，FutureTask等。AQS解决了在实现同步容器时设计的大量细节问题。

AQS使用一个FIFO的队列表示排队等待锁的线程，队列头节点称作“哨兵节点”或者“哑节点”，它不与任 何线程关联。其他的节点与等待线程关联，每个节点维护一个等待状态waitStatus。

AQS核心思想是，如果被请求的共享资源空闲，则将当前请求资源的线程设置为有效的工作线程，并且 将共享资源设置为锁定状态。如果被请求的共享资源被占用，那么就需要一套线程阻塞等待以及被唤醒 时锁分配的机制，这个机制AQS是用CLH队列锁实现的，即将暂时获取不到锁的线程加入到队列中。

#### 5.5.1 Lock接口是什么? 

面试官:说一下Lock接口是什么

Lock和ReadWriteLock是两大锁的根接口，Lock代表实现类是ReentrantLock(可重入锁)， ReadWriteLock(读写锁)的代表实现类是ReentrantReadWriteLock。

Lock 接口支持那些语义不同(重入、公平等)的锁规则，可以在非阻塞式结构的上下文中使用这些规 则，主要的实现是 ReentrantLock。

ReadWriteLock接口以类似方式定义了一些读取者可以共享而写入者独占的锁。此包只提供了一个实现，即 ReentrantReadWriteLock，因为它适用于大部分的标准用法上下文。但程序员可以创建自己的、适用于非标准要求的实现。

##### Lock接口和synchronized的区别

Lock不是Java语言内置的，synchronized是Java语言的关键字，因此是内置特性。Lock是一个类，通过这个类可以实现同步方法

Lock和synchronized有一点非常大的不同，采用synchronized不需要用户去手动释放锁，当 synchronized方法或者synchronized代码块执行完之后，系统会自动让线程释放对锁的占用;而 Lock则必须要用户去手动释放锁，如果没有主动释放锁，就有可能导致出现死锁现象。

##### Lock锁具有的优点

| 特性               | 描述                                                         |
| ------------------ | ------------------------------------------------------------ |
| 尝试非阻塞地获取锁 | 当前线程尝试获取锁，如果此时锁没被其他线程获取到，则当前线程获取 锁成功 |
| 响应中断地获取锁   | 当获取到锁的线程被中断时，中断异常将抛出，同时释放锁         |
| 超时地获取锁       | 在给定的时间内获取锁，如果超出给定时间仍未获取到锁，方法仍要返回 |

#### 5.5.2 LockSupport是什么?

LockSupport是一个非常方便实用的线程阻塞工具，它可以在线程内任意位置让线程阻塞。

它的内部其实两类主要的方法:park(停车阻塞线程)和unpark(启动唤醒线程)

##### 与wait/notify对比

LockSupport的park/unpark更符合这个语义，以“线程”作为方法的参数，语义更清晰，使用起来也更方便。而wait/notify的实现使得“线程”的阻塞/唤醒对线程本身来说是被动的，要准确的控制哪个线程、 什么时候阻塞/唤醒很困难， 要不随机唤醒一个线程(notify)要不唤醒所有的(notifyAll)

wait、notify方法有一个不好的地方，就是我们在编程的时候必须能保证wait方法比notify方法先执行。 如果notify方法比wait方法晚执行的话，就会导致因wait方法进入休眠的线程接收不到唤醒通知的问 题，而park、unpark则不会有这个问题

1.wait和notify都是Object中的方法,在调用这两个方法前必须先获得锁对象，但是park不需要获取某个对象的锁就可以锁住线程。

2.notify只能随机选择一个线程唤醒，无法唤醒指定的线程，unpark却可以唤醒一个指定的线程。

3.和wait方法不同，执行park进入休眠后并不会释放持有的锁

### 5.6 什么是阻塞队列

面试官:能说一下什么是阻塞队列吗?

阻塞队列是一个在队列基础上又支持了两个附加操作的队列。

BlockingQueue继承了Queue接口，是队列的一种。Queue和BlockingQueue都是在Java 5中加入的。

BlockingQueue 是线程安全的，在很多场景下都可以利用线程安全的队列来优雅地解决业务自身的线程安全问题。比如说，使用生产者/消费者模式的时候，生产者只需要往队列里添加元素，而消费者只需 要从队列里取出它们就可以了。

![](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202305241122278.png)

阻塞插入:队列满时，队列会阻塞插入元素的线程，直到队列不满阻塞移除

队列空时，获取元素的线程会等待队列变为非空

#### 5.6.1 应用场景

阻塞队列常用于生产者和消费者的场景，生产者是向队列里添加元素的线程，消费者是从队列里取元素的线程。简而言之，阻塞队列是生产者用来存放元素、消费者获取元素的容器。

#### 5.6.2 说下常用的阻塞队列

| 队列                  | 有界性 | 锁   | 结构 | 队列类型 |
| --------------------- | ------ | ---- | ---- | -------- |
| ArrayBlockingQueue    | 有界   | 加锁 | 数组 | 阻塞     |
| LinkedBlockingQueue   | 可选   | 加锁 | 链表 | 阻塞     |
| ConcurrentLinkedQueue | 无界   | 无锁 | 链表 | 非阻塞   |
| LinkedTransferQueue   | 无界   | 无锁 | 链表 | 阻塞     |
| PriorityBlockingQueue | 无界   | 加锁 | 堆   | 阻塞     |
| DelayQueue            | 无界   | 加锁 | 堆   | 阻塞     |

#### 5.6.3 说下如何用阻塞队列实现生产消费者模型

##### 什么是生产者消费者模型

生产者和消费者问题是线程模型中的经典问题:生产者和消费者在同一时间段内共用同一个存储空间，生产者往存储空间中添加产品，消费者从存储空间中取走产品，当存储空间为空时，消费者阻塞，当存储空间满时，生产者阻塞。

![](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202305241134743.png)

java.util.concurrent.BlockingQueue的特性是:当队列是空的时，从队列中获取或删除元素的操作将会 被阻塞，或者当队列是满时，往队列里添加元素的操作会被阻塞。

### 5.7 听说过Disruptor吗，和阻塞队列有什么区别

Disruptor框架是由LMAX公司开发的一款高效的无锁内存队列，单线程能支撑每秒600万订单。

Disruptor的最大特点就是高性能，它的内部与众不同的使用了环形队列(RingBuffer)来代替普通的 线型队列，相比普通队列环形队列不需要针对性的同步head和tail头尾指针，减少了线程协作的复杂 度，再加上它本身基于无锁操作的特性，并且解决了伪共享的问题，从而可以达到了非常高的性能;

#### 5.7.1 什么是伪共享 面试官:能说一下什么是伪共享吗?

当CPU访问某一个变量时候，首先会去看CPU Cache内是否有该变量，如果有则直接从中获取，否者就去主内存里面获取该变量，然后把该变量所在内存区域的一个Cache行大小的内存拷贝到 Cache(cache行是Cache与主内存进行数据交换的单位)。

由于存放到Cache行的的是内存块而不是单个变量，所以可能会把多个变量存放到了一个cache行。当多个线程同时修改一个缓存行里面的多个变量时候，由于同时只能有一个线程操作缓存行，所以相比每个变量放到一个缓存行性能会有所下降，这就是伪共享。

![](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202305241410513.png)

#### 5.7.2 如何解决 

##### 字段填充

解决伪共享最直接的方法就是填充(padding)，例如下面的VolatileLong，一个long占8个字节，Java 的对象头占用8个字节(32位系统)或者12字节(64位系统，默认开启对象头压缩，不开启占16字 节)。一个缓存行64字节，那么我们可以填充6个long(6 * 8 = 48 个字节)。

```java
/**
	* 缓存行填充父类 
	*/
public class DataPadding {
		//填充 5个long类型字段 8*5 = 40 个字节
		private long p1, p2, p3, p4, p5; //jvm 优化 删除无用代码 //需要操作的数据
		volatile long value;
}
```

##### 继承的方式

```java
/**
	* 缓存行填充父类 
	*/
public class DataPadding {
		//填充 5个long类型字段 8*5 = 40 个字节 
  	private long p1, p2, p3, p4, p5;
}
```

```java
/**
 * 继承DataPadding
 */
public class VolatileData extends DataPadding { 
    // 占用 8个字节 +48 + 对象头 = 64字节
    public VolatileData() {
    }

    public VolatileData(long defValue) {
        value = defValue;
    }

    public long accumulationAdd() { 
        //因为单线程操作不需要加锁 value++;
        return value;
    }

    public long getValue() {
        return value;
    }
}
```



---

## 6.能介绍下JUC包吗

面试官:说下你了解的JUC包有那些类以及作用 JUC全称:java.util.concurrent，是JDK提供的一个处理并发的工具包。

在此包中增加了在并发编程中很常用的实用工具类，用于定义类似于线程的自定义子系统，包括线程 池、异步IO 和轻量级任务框架。提供可调的、灵活的线程池。还提供了设计用于多线程上下文中的 Collection 实现等

### 6.1 JUC分类

#### 6.1.1 atomic类

集中在Atomic包下面实现了原子化操作的数据类型，包括 Boolean, Integer, Long, 和Referrence这四种类型以及这四种类型的数组类型。 

#### 6.1.2 锁类

这部分都被放在lock这个包里面，实现了并发操作中的几种类型的锁，如ReentrantLock类、 ReentrantReadWriteLock类等

#### 6.1.3 集合框架的并发类

这部分主要介绍实现线程安全的集合类，如CopyOnWriteArrayList类、CopyOnWriteArraySet类等

#### 6.1.4 线程管理类

这部分主要是对线程集合的管理的实现，有CyclicBarrier, CountDownLatch,Exchanger等一些类 

#### 6.1.5 阻塞队列类

阻塞队列是线程池实现的重要组成部分，如LinkedBlockingQueue类、PriorityBlockingQueue类等

### 6.2 CyclicBarrier和CountDownLatch的用法及区别 

#### 6.2.1 CountDownLatch

CountDownLatch是一个非常实用的多线程控制工具类，称之为“倒计时器”，它允许一个或多个线程一 直等待，直到其他线程的操作执行完后再执行。

CountDownLatch是通过一个计数器来实现的，计数器的初始值为线程的数量。每当一个线程完成了自 己的任务后，计数器的值就会减1。当计数器值到达0时，它表示所有的线程已经完成了任务，然后在闭 锁上等待的线程就可以恢复执行任务。

#### 6.2.2 CyclicBarrier

CyclicBarrier，是JDK1.5的java.util.concurrent并发包中提供的一个并发工具类。

CyclicBarrier和CountDownLatch是非常类似的，CyclicBarrier核心的概念是在于设置一个等待程的数量边界，到达了此边界之后进行执行。

#### 6.2.3 CyclicBarrier 与 CountDownLatch 区别

CountDownLatch 是一次性的，CyclicBarrier 是可循环利用的 CountDownLatch.await一般阻塞工作线程，所有的进行预备工作的线程执行countDown，而 CyclicBarrier通过工作线程调用await从而自行阻塞，直到所有工作线程达到指定屏障，再大家一 起往下走。

CountDownLatch 参与的线程的职责是不一样的，有的在倒计时，有的在等待倒计时结束。 CyclicBarrier 参与的线程职责是一样的。 在控制多个线程同时运行上，CountDownLatch可以不限线程数量，而CyclicBarrier是固定线程 数。

同时，CyclicBarrier还可以提供一个barrierAction，合并多线程计算结果。 

#### 6.3 Semaphore有什么作用?

Semaphore也叫信号量，在JDK1.5被引入，可以用来控制同时访问特定资源的线程数量，通过协调各个线程，以保证合理的使用资源。

Semaphore 是 synchronized 的加强版，作用是控制线程的并发数量。就这一点而言，单纯的 synchronized 关键字是实现不了的。

Semaphore可以用于做流量控制，特别是公用资源有限的应用场景，比如数据库连接。假如有一个需求，要读取几万个文件的数据，因为都是IO密集型任务，我们可以启动几十个线程并发地读取，但是如 果读到内存后，还需要存储到数据库中，而数据库的连接数只有10个，这时我们必须控制只有10个线程 同时获取数据库连接保存数据，否则会报错无法获取数据库连接。这个时候，就可以使用Semaphore 来做流量控制。

### 6.4 什么是Callable和Future

#### 6.4.1 Callable接口

Callable位于JUC包下，它也是一个接口，在它里面也只声明了一个方法叫做call():

```java
@FunctionalInterface
public interface Callable<V> {
    /**
     * Computes a result, or throws an exception if unable to do so.
     *
     * @return computed result
     * @throws Exception if unable to compute a result
     */
    V call() throws Exception;
}
```

Callable接口代表一段可以调用并返回结果的代码。 

#### 6.4.2 Future接口

Future就是对于具体的Runnable或者Callable任务的执行结果进行取消、查询是否完成、获取结果。必 要时可以通过get方法获取执行结果，该方法会阻塞直到任务返回结果。

Future接口是用来获取异步计算结果的，说白了就是对具体的Runnable或者Callable对象任务执行的结 果进行获取(get())，取消(cancel())，判断是否完成等操作



Future提供了三种功能

1.判断任务是否完成

2.能够中断任务

3.能够获取任务执行结果

#### 6.4.3 FutureTask

Future是一个接口，是无法生成一个实例的，所以又有了FutureTask。FutureTask实现了 RunnableFuture接口，RunnableFuture接口又实现了Runnable接口和Future接口。所以FutureTask 既可以被当做Runnable来执行，也可以被当做Future来获取Callable的返回结果。

### 6.5 ThreadLocal原理，使用注意点，应用场景有哪些?

#### 6.5.1 ThreadLocal是什么?

ThreadLocal，即线程本地变量，如果你创建了一个ThreadLocal变量，那么访问这个变量的每个线程 都会有这个变量的一个本地拷贝，多个线程操作这个变量的时候，实际是操作自己本地内存里面的变 量，从而起到线程隔离的作用，避免了线程安全问题。

ThreadLocal 用作保存每个线程独享的对象，为每个线程都创建一个副本，这样每个线程都可以修改自 己所拥有的副本, 而不会影响其他线程的副本，确保了线程安全。

#### 6.5.2 ThreadLocal原理

Thread类有一个类型为ThreadLocal.ThreadLocalMap的实例变量threadLocals，即每个线程都有 一个属于自己的ThreadLocalMap。 ThreadLocalMap内部维护着Entry数组，每个Entry代表一个完整的对象，key是ThreadLocal本 身，value是ThreadLocal的泛型值。 每个线程在往ThreadLocal里设置值的时候，都是往自己的ThreadLocalMap里存，读也是以某个 ThreadLocal作为引用，在自己的map里找对应的key，从而实现了线程隔离。

#### 6.5.3 ThreadLocal内存泄漏问题

根据我们前面对ThreadLocal的分析，我们可以知道每个Thread 维护一个 ThreadLocalMap，这个映射 表的 key 是 ThreadLocal实例本身，value 是真正需要存储的 Object，也就是说 ThreadLocal 本身并不 存储值，它只是作为一个 key 来让线程从 ThreadLocalMap 获取 value。仔细观察ThreadLocalMap， 这个map是使用 ThreadLocal 的弱引用作为 Key 的，弱引用的对象在 GC 时会被回收。

![](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202305241435519.png)

##### 四种引用类型

强引用: 就是指在程序代码之中普遍存在的，类似“Object obj=new Object()”这类的引用，只要强引用还存在，垃圾收集器永远不会回收掉被引用的对象实例。

软引用: 是用来描述一些还有用但并非必需的对象。对于软引用关联着的对象，在系统将要发生内存溢出异常之前，将会把这些对象实例列进回收范围之中进行第二次回收。如果这次回收还没有足够的内存，才会抛出内存溢出异常。在JDK 1.2之后，提供了SoftReference类来实现软引用。 

弱引用: 也是用来描述非必需对象的，但是它的强度比软引用更弱一些，被弱引用关联的对象实例 只能生存到下一次垃圾收集发生之前。当垃圾收集器工作时，无论当前内存是否足够，都会回收掉只被弱引用关联的对象实例。在JDK 1.2之后，提供了WeakReference类来实现弱引用。

虚引用: 也称为幽灵引用或者幻影引用，它是最弱的一种引用关系。一个对象实例是否有虚引用的存在，完全不会对其生存时间构成影响，也无法通过虚引用来取得一个对象实例。为一个对象设置 虚引用关联的唯一目的就是能在这个对象实例被收集器回收时收到一个系统通知。在之后，提供了类来实现虚引用

### 6.6 Fork/Join框架的理解?

#### 6.6.1 什么是Fork/Join

Fork/Join框架是一个实现了ExecutorService接口的多线程处理器。它可以把一个大的任务划分为若干个小的任务并发执行，充分利用可用的资源，进而提高应用的执行效率。

#### 6.6.2 什么是工作窃取算法

工作窃取算法是指线程从其他任务队列中窃取任务执行。考虑下面这种场景

有一个很大的计算任务，为了减少线程的竞争，会将这些大任务切分为小任务并分在不同的队列等待执行，然后为每个任务队列创建一个线程执行队列的任务。那么问题来了，有的线程可能很快就执行完了，而其他线程还有任务没执行完，执行完的线程与其空闲下来不如帮助其他线程执行任务，这样也能加快执行进程。所以，执行完的空闲线程从其他队列的尾部窃取任务执行，而被窃取任务的线程则从队列的头部取任务执行(这里使用了双端队列，既不影响被窃取任务的执行过程又能加快执行进度)。

### 6.7 ConcurrentHashMap并发度是什么

在JDK1.7中ConcurrentHashMap把实际map划分成若干部分来实现它的可扩展性和线程安全。这种划 分是使用并发度获得的，它是 ConcurrentHashMap类构造函数的一个可选参数，默认值为16，这样在 多线程情况下就能避免争，在 JDK8 后，它摒弃了 Segment(锁段)的概念，而是启用了一种全新的方 式实现,利用 CAS 算法。同时加入了更多的辅助变量来提高并发度。

#### 6.7.1 JDK 1.7 结构

![](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202305241436506.png)

#### 6.7.2 JDK 1.8 结构

![](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202305241436909.png)

#### 6.7.3 ConcurrentHashMap和HashTable的区别

ConcurrentHashMap在JDK1.7时，底层采用分段数组+链表形式

JDK1.8以后和HashMap一样采用数 组+链表/红黑二叉树，HashTable和JDK1.8以前的HashMap一样的底层数据结构:数组+链表，数组是 HashMap的主体，链表是为了解决哈希冲突问题
