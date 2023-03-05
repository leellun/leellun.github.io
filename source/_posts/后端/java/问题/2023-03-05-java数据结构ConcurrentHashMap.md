---
title: java数据结构ConcurrentHashMap 
date: 2023-03-05 10:18:54
categories:
  - 后端
  - java
tags:
  - ConcurrentHashMap 
---

### 1.HashMap 和 ConcurrentHashMap 的区别

JDK1.7 ConcurrentHashMap对整个桶数组进行了分割分段(Segment)，然后在每一个分段上都用lock锁进行保护，相对于HashTable的synchronized锁的粒度更精细了一些，并发性能更好，而HashMap没有锁机制，不是线程安全的。（JDK1.8之后ConcurrentHashMap启用了一种全新的方式实现,利用CAS算法。）

HashMap的键值对允许有null，但是ConCurrentHashMap都不允许。

- 1、JDK1.8：synchronized+CAS+HashEntry+红黑树；
- 2、JDK1.7：ReentrantLock+Segment+HashEntry。

### 2.ConcurrentHashMap 和 Hashtable 的区别？

ConcurrentHashMap 和 Hashtable 的区别主要体现在实现线程安全的方式上不同。

底层数据结构：JDK1.7的 ConcurrentHashMap 底层采用 分段的数组+链表 实现，JDK1.8 采用的数据结构跟HashMap1.8的结构一样，数组+链表/红黑二叉树。Hashtable 和 JDK1.8 之前的 HashMap 的底层数据结构类似都是采用 数组+链表 的形式，数组是 HashMap 的主体，链表则是主要为了解决哈希冲突而存在的；

实现线程安全的方式（重要）：① 在JDK1.7的时候，ConcurrentHashMap（分段锁） 对整个桶数组进行了分割分段(Segment)，每一把锁只锁容器其中一部分数据，多线程访问容器里不同数据段的数据，就不会存在锁竞争，提高并发访问率。（默认分配16个Segment，比Hashtable效率提高16倍。） 到了 JDK1.8 的时候已经摒弃了Segment的概念，而是直接用 Node 数组+链表+红黑树的数据结构来实现，并发控制使用 synchronized 和 CAS 来操作。（JDK1.6以后 对 synchronized锁做了很多优化） 整个看起来就像是优化过且线程安全的 HashMap，虽然在JDK1.8中还能看到 Segment 的数据结构，但是已经简化了属性，只是为了兼容旧版本；② Hashtable(同一把锁) :使用 synchronized 来保证线程安全，效率非常低下。当一个线程访问同步方法时，其他线程也访问同步方法，可能会进入阻塞或轮询状态，如使用 put 添加元素，另一个线程不能使用 put 添加元素，也不能使用 get，竞争会越来越激烈效率越低。

### 3.ConcurrentHashMap在JDK1.8中,为什么要使用内置锁synchronized来代替重入锁ReentrantLock?

①、粒度降低了;

②、JVM开发团队没有放弃synchronized,而且基于JVM的synchronized优化空间更大,更加自然。

③、在大量的数据操作下,对于JVM的内存压力,基于API的ReentrantLock会开销更多的内存。

### 4 ConcurrentHashMap如何get？为何无需加锁

ConcurrentHashMap的get操作并没有像put操作一样有CAS和synchronized锁。get操作不需要加锁，因为 Node 的元素 value 和指针 next 是用 volatile 修饰的，所以在多线程的环境下，即便value的值被修改了，在线程之间也是可见的。

### 5 ConcurrentHashMap的迭代器是强一致性还是弱一致性？

HashMap的迭代器是强一致性的，而ConcurrentHashMap的迭代器是弱一致性的 

### 6. JDK1.8中为什么使用synchronized替换ReentrantLock？

性能：因为是synchronized在jdk1.6引入了大量的优化，如锁升级，锁粗化，锁消除，自旋锁，自适应自旋锁等的优化，本身的性能得到了提升。

并发度和内存开销：CAS+synchronized方式时加锁的对象是每个链条的头节点，相对Segment再次提高了并发度，如果使用可重入锁达到同样的效果的话，则需要大量继承自ReentrantLock的对象，造成巨大的内存浪费。



 

 

 

 

 