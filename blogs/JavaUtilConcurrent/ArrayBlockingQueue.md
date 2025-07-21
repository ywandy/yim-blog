---
title: 并发阻塞队列-ArrayBlockingQueue
tags: 
  - ArrayBlockingQueue
date: 2022-04-10 18:28:00
categories:	
  - JUC
---

Java并发包中的阻塞队列一共7个，当然他们都是线程安全的。 

ArrayBlockingQueue：一个由数组结构组成的有界阻塞队列。 

　　LinkedBlockingQueue：一个由链表结构组成的有界阻塞队列。 

　　PriorityBlockingQueue：一个支持优先级排序的无界阻塞队列。 

　　DealyQueue：一个使用优先级队列实现的无界阻塞队列。 

　　SynchronousQueue：一个不存储元素的阻塞队列。 

　　LinkedTransferQueue：一个由链表结构组成的无界阻塞队列。 

　　LinkedBlockingDeque：一个由链表结构组成的双向阻塞队列。（摘自《Java并发编程的艺术》） 

　　在本文对ArrayBlockingQueue阻塞队列做一个简要解析 

　　对于ArrayLinkedQueue，放眼看过去其安全性的保证是由ReentrantLock保证的，有关ReentrantLock的解析可参考[《](http://www.cnblogs.com/yulinfeng/p/6906597.html)[5.Lock](http://www.cnblogs.com/yulinfeng/p/6906597.html)[接口及其实现](http://www.cnblogs.com/yulinfeng/p/6906597.html)[ReentrantLock]，在下文我也会适当的提及。 

　　首先来查看其构造函数： 

| 构造方法                                                     |                                                              |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| public ArrayBlockingQueue(int capacity)                      | 构造指定大小的有界队列                                       |
| public ArrayBlockingQueue(int capacity, boolean fair)        | 构造指定大小的有界队列，指定为公平或非公平锁                 |
| public ArrayBlockingQueue(int capacity, boolean fair, Collection<? extends E> c) | 构造指定大小的有界队列，指定为公平或非公平锁，指定在初始化时加入一个集合 |



```java
 public ArrayBlockingQueue(int capacity) { 
  　　this(capacity, false);//默认构造非公平锁的阻塞队列 
  } 
  public ArrayBlockingQueue(int capacity, boolean fair) { 
  　　if (capacity <= 0)  
  　　　　throw new IllegalArgumentException(); 
  　　this.items = new Object[capacity]; 
  　　lock = new ReentrantLock(fair);//初始化ReentrantLock重入锁，出队入队拥有这同一个锁 
  　　notEmpty = lock.newCondition;//初始化非空等待队列，有关Condition可参考《6.类似Object监视器方法的Condition接口》
 　　notFull = lock.newCondition;//初始化非满等待队列 
 } 
 public ArrayBlockingQueue(int capacity, boolean fair, Collecation<? extends E> c) { 
 　　this(capacity, fair); 
 　　final ReentrantLock lock = this.lock; 
 　　lock.lock();//注意在这个地方需要获得锁，这为什么需要获取锁的操作呢？ 
 　　try { 
 　　　　int i = 0; 
 　　　　try { 
 　　　　　　for (E e : c) { 
 　　　　　　　　checkNotNull(e); 
 　　　　　　　　item[i++] = e;//将集合添加进数组构成的队列中 
 　　　　　　} 
 　　　　} catch (ArrayIndexOutOfBoundsException ex) { 
 　　　　　　throw new IllegalArgumentException(); 
 　　　　} 
 　　　　count = i;//队列中的实际数据数量 
 　　　　putIndex = (i == capacity) ? 0 : i; 
 　　} finally { 
 　　　　lock.unlock(); 
 　　} 
 } 
```

　　在第15行，源码里给了一句注释： Lock only for visibility, not mutual exclusion。这句话的意思就是给出，这个锁的操作并不是为了互斥操作，而是保证其可见性。线程T1是实例化ArrayBlockingQueue对象，T2是对实例化的ArrayBlockingQueue对象做入队操作（当然要保证T1和T2的执行顺序），如果不对它进行加锁操作（加锁会保证其可见性，也就是写回主存），T1的集合c有可能只存在T1线程维护的缓存中，并没有写回主存，T2中实例化的ArrayBlockingQueue维护的缓存以及主存中并没有集合c，此时就因为可见性造成数据不一致的情况，引发线程安全问题。 

　　以下是ArrayBlockingQueue的一些出队入队操作。

## **队列元素的插入**

|      | 抛出异常                                                     | 返回值（非阻塞）                                             | 一定时间内返回值                                             | 返回值（阻塞）                                               |
| ---- | ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 插入 | add(e)//队列未满时，返回true；队列满则抛出IllegalStateException(“Queue full”)异常——AbstractQueue | offer(e)//队列未满时，返回true；队列满时返回false。非阻塞立即返回。 | offer(e, time, unit)//设定等待的时间，如果在指定时间内还不能往队列中插入数据则返回false，插入成功返回true。 | put(e)//队列未满时，直接插入没有返回值；队列满时会阻塞等待，一直等到队列未满时再插入。 |

```java
//ArrayBlockingQueue#add 
public boolean add(E e) { 
　　return super.add(e); 
} 
```

```java
//AbstractQueue#add，这是一个模板方法，只定义add入队算法骨架，成功时返回true，失败时抛出IllegalStateException异常，具体offer实现交给子类实现。 
public boolean add(E e) { 
　　if (offer(e))//offer方法由Queue接口定义 
　　　　return true; 
　　else 
　　　　throw new IllegalStateException(); 
}
```

```java
//ArrayBlockingQueue#offer，队列未满时返回true，满时返回false 
public boolean offer(E e) { 
　　checkNotNull(e);//检查入队元素是否为空 
　　final ReentrantLock lock = this.lock; 
　　lock.lock();//获得锁，线程安全 
　　try { 
　　　　if (count == items.length)//队列满时，不阻塞等待，直接返回false 
　　　　　　return false; 
　　　　else { 
　　　　　　insert(e);//队列未满，直接插入 
　　　　　　return true; 
　　　　} 
　　} finally {
　　　　lock.unlock();
　　}
} 
```

```java
//ArrayBlockingQueue#insert 
private void insert(E e) { 
　　items[putIndex] = x; 
　　putIndex = inc(putIndex); 
　　++count; 
　　notEmpty.signal();//唤醒非空等待队列中的线程，有关Condition可参考《6.类似Object监视器方法的Condition接口》
 }
```

　　在这里有几个ArrayBlockingQueue成员变量。items即队列的数组引用，putIndex表示等待插入的数组下标位置。当items[putIndex] = x将新元素插入队列中后，调用inc将数组下标向后移动，如果队列满则将putIndex置为0:

```
//ArrayBlockingQueue#inc 
private int inc(int i) { 
　　return (++i == items.length) ? 0 : i; 
} 
```

　　接着解析下put方法，阻塞插入队列，当队列满时不会返回false，也不会抛出异常，而是一直阻塞等待，直到有空位可插入，但它可被中断返回。

```java
//ArrayBlockingQueue#put 
public void put(E e) throws InterruptedException { 
　　checkNotNull(e);//同样检查插入元素是否为空 
　　final ReentrantLock lock = this.lock; 
　　lock.lockInterruptibly();//这里并没有调用lock方法，而是调用了可被中断的lockInterruptibly，该方法可被线程中断返回，lock不能被中断返回。 
　　try { 
　　　　while (count == items.length) 
　　　　　　notFull.await();//当队列满时，使非满等待队列休眠 
　　　　insert(e);//此时表示队列非满，故插入元素，同时在该方法里唤醒非空等待队列 
　　} finally { 
　　　　lock.unlock(); 
　　} 
}  
```

## **队列元素的删除** 

| 抛出异常                                                     | 返回值（非阻塞）                                             | 一定时间内返回值                                             | 返回值（阻塞）                                               |
| ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| remove()//队列不为空时，返回队首值并移除；队列为空时抛出NoSuchElementException()异常——AbstractQueue | poll()//队列不为空时返回队首值并移除；队列为空时返回null。非阻塞立即返回。 | poll(time, unit)//设定等待的时间，如果在指定时间内队列还未孔则返回null，不为空则返回队首值 | take(e)//队列不为空返回队首值并移除；当队列为空时会阻塞等待，一直等到队列不为空时再返回队首值。 |

```java
//AbstractQueue#remove，这也是一个模板方法，定义删除队列元素的算法骨架，队列中元素时返回具体元素，元素为空时抛出异常，具体实现poll由子类实现， 
public E remove() { 
　　E x = poll();//poll方法由Queue接口定义 
　　if (x != null) 
　　　　return x; 
　　else 
　　　　throw new NoSuchElementException(); 
} 
```

```java
//ArrayBlockingQueue#poll，队列中有元素时返回元素，不为空时返回null 
public E poll() { 
　　final ReentrantLock lock = this.lock; 
　　lock.lock(); 
　　try { 
　　　　return (count == 0) ? null : extract(); 
　　} finally { 
　　　　lock.unlock(); 
　　} 
}
```

```java
//ArrayBlockingQueue#extract 
private E extract() { 
　　final Object[] items = this.items; 
　　E x = this.<E>cast(items[takeIndex]);//移除队首元素 
　　items[takeIndex] = null;//将队列数组中的第一个元素置为null，便于GC回收 
　　takeIndex = inc(takeIndex); 
　　--count; 
　　notFull.signal();//唤醒非满等待队列线程 
　　return x; 
} 
```

　　对比add和offer方法，理解了上两个方法后remove和poll实际不难理解，同理在理解了put阻塞插入队列后，对比take阻塞删除队列元素同样也很好理解。

```java
//ArrayBlockQueue#take 
public E take() throws InterruptedException { 
　　final ReentrantLock lock = this.lock(); 
　　lock.lockInterrupted();//这里并没有调用lock方法，而是调用了可被中断的lockInterruptibly，该方法可被线程中断返回，lock不能被中断返回。 
　　try { 
　　　　while (count == 0)//队列元素为空 
　　　　　　notEmpty.await();//非空等待队列休眠 
　　　　return extract();//此时表示队列非空，故删除元素，同时在里唤醒非满等待队列 
　　} finally { 
　　　　lock.unlock(); 
　　} 
} 
```

　　最后一个方法size。

```java
public int size() { 
　　final ReentrantLock lock = this.lock; 
　　lock.lock(); 
　　try { 
　　　　return count; 
　　} finally { 
　　　　lock.unlock(); 
　　} 
}
```

　　可以看到ArrayBlockingQueue队列的size方法，是直接返回的count变量，它不像ConcurrentLinkedQueue，ConcurrentLinkedQueue的size则是每次会遍历这个队列，故ArrayBlockingQueue的size方法比ConcurrentLinkedQueue的size方法效率高。而且ConcurrentLinkedQueue的size方法并没有加锁！也就是说很有可能其size并不准确，这在它的注释中说明了ConcurrentLinkedQueue的size并没有多大的用处。