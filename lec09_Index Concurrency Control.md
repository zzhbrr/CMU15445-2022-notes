#! https://zhuanlan.zhihu.com/p/603309035

- [Lec09 Index Concurrency Control](#lec09-index-concurrency-control)
  - [Locks vs. Latches](#locks-vs-latches)
  - [Hash Table Latching](#hash-table-latching)
  - [B+Tress Latching](#btress-latching)
    - [Latch Crabbing](#latch-crabbing)
    - [Improved Latch Crabbing Protocol](#improved-latch-crabbing-protocol)
    - [Leaf Node Scans](#leaf-node-scans)


# Lec09 Index Concurrency Control

之前我们讨论的数据结构都是单线程的，然而大部分DBMS在实际场景中需要允许多线程安全地访问数据结构，以充分利用CPU多核以及隐藏磁盘I/O延时。（有DBMS只支持单线程，如Redis）。



并发控制协议，是DBMS使用以保证并发操作正确性的方法。

并发控制协议正确性标准有：

* **Logical Correctness**，线程可以读它期望读到的值，如在一个transaction中，一个线程在写完一个值之后，再读它，应该是它之前写的值。
* **Physical Correctness**，共享对象的内部表示是安全的，如共享对象内部指针不能指向非法物理位置。

本讲着重于Physical Correctness，Logical Correctness后面会讲。

## Locks vs. Latches

之前我们也分析过Locks和Latches的区别。

* Locks

  Locks不同于OS中的锁，数据库中的lock是一个higher-level的，概念上的，避免不同transactions对数据库的竞争。如对tuples、tables、databases的lock。Transactions会在它整个生命周期持有lock

* Latches

  Latches是一个low-level的保护原语，DBMS用于其内部数据结构中的临界区，如hash table等。Latch只在操作执行的时候被持有。

<img src="https://pic4.zhimg.com/80/v2-52f15d3b576ba95e33cd08e1453afe2d.png" style="zoom: 67%;" />

本文讨论的是Latch。

Latch有读模式与写模式两种模式，分别称作“读锁”和“写锁”。当线程要读数据时，需要加上读锁不让别的线程写这个数据；当线程要修改某个数据时，需要加上写锁确保别的线程既不能读也不能写这个数据。

读锁可以相互兼容的，同一个对象可以上多个读锁，但是写锁不兼容其他锁，如果一个线程上了写锁，那么不能上任何一个其他锁。但如果这个对象有一个线程在等待获取写锁，如果这时来了一个新线程想申请读锁，则不能当时就读，必须等到想要写锁的那个线程释放锁之后，再读，这样是为了防止饥饿。

![](https://pic4.zhimg.com/80/v2-eed9b204b19fa26a19f7acdaffa65b72.png)

## Hash Table Latching

接下来介绍哈希表的并发控制。

静态哈希表比较好加锁，如开放地址哈希，进行查找的线程都是从上到下查找，从上到下地获取slot的锁，不会出现死锁的情况，如果要调整哈希表的大小，则对整个哈希表上把大锁就行。

动态哈希方法，有更多的共享状态，更难进行并发控制，但是大体上的方法与静态哈希表是类似的。

不同的局部锁的粒度有：

* **Page Latches**

  对每个页上读写锁，线程访问页之前上锁，但会降低一些并行性。

* **Slot Latches**

  对每个slot上读写锁，增加了并行性，但是也增加了内存和计算开销，可以使用单模式锁（即自旋锁）来减少内存和计算开销，代价是降低了并行性。

## B+Tress Latching

B+树锁要防止两方面问题：

* 多个线程同时尝试修改一个节点的内容
* 一个线程遍历树的时候，另一个线程在分裂或者合并节点

### Latch Crabbing

基本思想如下：

1. 获得父节点的锁
2. 获得子节点的锁
3. 如果“安全”的话，释放父节点的锁。“安全”是指节点一定不会分裂或合并或重分配

“安全”的概念取决于操作是插入还是删除，如已经满载的节点对于删除是安全的，但是对于插入是不安全的。而且读锁不需要考虑“安全”的问题。

**Find:**

1. 根节点开始遍历树
2. 对子节点上读锁
3. 释放父节点的锁
4. 重复直到达到叶子节点

**Insert/Delete:**

1. 从根节点开始遍历树
2. 对子节点上写锁
3. 如果子节点是”安全“的，则释放所有祖宗节点的锁；如果不是“安全“的，则仍然持有父节点的锁
4. 重复直到完成插入/删除

### Improved Latch Crabbing Protocol

考虑到添加和删除的时候，直接上写锁，这大大影响了并发性，但是实际情况是，我们的B+树大得很，只有很少的操作可以影响到树结构。

改进版本的**Find**和之前一样。

改进版本的**Insert/Delete**变为，对中间节点都上读锁，在叶子节点上写锁，如果叶子节点是不安全的，就重新做这个操作，只不过改成和之前一样的，对所有节点都上写锁。

### Leaf Node Scans

对于上面讨论的操作，都是在树上从上到下获取锁，永远不会出现死锁的情况。

但是如果在叶子节点进行扫描的时候，如果一个线程从左向右获取锁，一个线程从右向左获取锁，那么就会出现死锁的情况。而Latch又不支持死锁检测和死锁避免，因此我们只能通过代码规范来避免死锁。

leaf node scan遵守“no-wait”模式，即如果获取锁失败，就立马释放掉自己拥有的所有锁，并重新开始。

