#! https://zhuanlan.zhihu.com/p/598894750
- [Lec06 Memory Management](#lec06-memory-management)
  - [Introduction](#introduction)
  - [Buffer Pool](#buffer-pool)
    - [Buffer Pool Meta-data](#buffer-pool-meta-data)
    - [Memory Allocation Polices](#memory-allocation-polices)
  - [Buffer Pool Optimization](#buffer-pool-optimization)
    - [Multiple Buffer Pools](#multiple-buffer-pools)
    - [Pre-fetching](#pre-fetching)
    - [Scan Sharing (Synchronized Scans)](#scan-sharing-synchronized-scans)
    - [Buffer Pool Bypass](#buffer-pool-bypass)
  - [OS Page Cache](#os-page-cache)
  - [Buffer Replacement Policies](#buffer-replacement-policies)
    - [Least Recently Used (LRU)](#least-recently-used-lru)
    - [CLOCK](#clock)
    - [Alternatives](#alternatives)
      - [LRU-K](#lru-k)
      - [Localization](#localization)
      - [Priority Hints](#priority-hints)
    - [Dirty Pages](#dirty-pages)
  - [Other Memory Pools](#other-memory-pools)
  - [Conclusion](#conclusion)

# Lec06 Memory Management

## Introduction

DBMS负责管理其内存，并在内存和磁盘间移动数据。由于在大多数情况下，数据不能直接在磁盘上操作，因此任何数据库都必须能够高效地将磁盘上的文件数据移动到内存中，以便使用。

DBMS面临的一个困难问题是如何将移动数据的性能影响降到最低。理想情况下，它应该看起来像是所有数据都在内存里。执行器不用担心如何将数据提取到内存中。如下图所示：

<img src="https://pic4.zhimg.com/80/v2-3e89c3458f28d06ed3bf389ac41e2262.png" style="zoom:67%;" />


执行器想使用page2，就告诉内存管理器需要page2，内存管理器发现buffer poo中没有page2，于是从磁盘中加载了page directory到缓存池，它指示了page2的物理位置，然后再将page2加载到缓存池，之后告诉执行器page2的指针。

内存管理需要考虑以下的问题：

* Spatial Control
  * 即将page存储到磁盘的哪里
  * 目标是：尽可能将一块使用的pages放到磁盘中物理位置相邻的地方上
* Temporal Control
  * 即什么时候将page读入内存，什么时候将page写回磁盘
  * 目标是：最小化磁盘读取造成的停顿

本节课要讨论的问题有：

* Buffer Pool Manager
* Replacement Polices
* Other Memory Pools



## Buffer Pool

Buffer pool是一种in-memory cache，用来缓存从磁盘上读取的pages，本质上是从数据库内存中分配的一大块区域。

Buffer pool的组织形式是fixes-size page数组，一个数组项叫做一**帧**（frame）。当DBMS需要一个page时，这个page的副本就放在一个frame中。所以数据库可以先在buffer pool中查看是否有这一个page，如果没有找到，系统就会从磁盘中读入这个page。

如下面这张图，frame1存放着page1， frame2存放着page3：


![Image](https://pic4.zhimg.com/80/v2-aaa46b3a5e0ff88039d507f0cdcfc151.png)

脏页（dirty pages）被缓存，并且不会被立即写回到磁盘中。

### Buffer Pool Meta-data

Buffer pool必须维护一些元数据，以便能高效、正确地运行。

**Page table**

* Page table是一个in-memory hash table，记录目前在内存中的页。此页表非OS中的页表！
* 它将page id映射到frame location（因为在buffer pool中的页不是按顺序存的）
* page directory是将page id映射到数据库文件位置，更改后需要写回磁盘。不同于page table，不需要存储在磁盘上。

**Other meta-data**

* Dirty Flag: 标志这个页是否已经被修改

* Pin/Reference Counter

  pin/reference counter记录有多少线程正在访问这个page（读或写），如果count大于0，则不能被置换出内存。

  同时也可以声明锁，避免多线程产生的问题。
    ![Image](https://pic4.zhimg.com/80/v2-d3d7a6a17845935b3a258550af7ee57a.png)



小插曲，**Locks和Latches有什么区别：**

* Locks

  Locks不同于OS中的锁，数据库中的lock是一个higher-level的，概念上的，避免不同transactions对数据库的竞争。如对tuples、tables、databases的lock。Transactions会在它整个生命周期持有lock

* Latches

  Latches是一个low-level的保护原语，DBMS用于其内部数据结构中的关键部分，如hash table等。Latch只在操作执行的时候被持有。

### Memory Allocation Polices

数据库中的内存根据两个策略分配给缓冲池:

1. Global polices

   做出使正在执行的**整个工作负载受益**的决策。它考虑所有active的事务，以找到分配内存的最佳决策。

2. Local polices

   做出的决策将使**单个查询或事务**运行得更快，即使这对整个工作负载都不好。本地策略将帧分配给特定事务，而不考虑并发事务

大多数系统都综合以上两种策略。

## Buffer Pool Optimization

有许多方法优化buffer pool。

### Multiple Buffer Pools

一般DBMS要维护多个buffer pool，例如每个database一个buffer pool、每个page type用不同的buffer pool。

不同的buffer pool可以用适合它们存储的数据的策略。可以减少latch竞争，提高局部性。

有两种方法将知道page在哪个buffer pool中：

1. Object IDS

   将每个page的record ID加一个标识符，标识这个page在哪个buffer pool中

   <img src="https://pic4.zhimg.com/80/v2-7e6b70220772484239be0d0bff383fad.png" style="zoom:80%;" />


2. Hashing

   将page的record ID 哈希后存储在那个buffer pool中

   <img src="https://pic4.zhimg.com/80/v2-56dbe02a1dc2e1698a1eb1003d281ef6.png" style="zoom:80%;" />


### Pre-fetching

DBMS可以根据query plan提前将page取到buffer pool中。

比如已经扫描了page0和page1，系统认为我们将做全表扫描，那么就会提前将page2和page3读入buffer pool

<img src="https://pic4.zhimg.com/80/v2-21c4b7ab199f6e453316f2c83a21fddc.png" style="zoom: 67%;" />

<img src="https://pic4.zhimg.com/80/v2-6c871c4ef1f8dd962de4333e242f9cfe.png" style="zoom:67%;" />

### Scan Sharing (Synchronized Scans)

查询可以重用从存储或计算中取回的数据。这允许扫描表的单个游标上进行多个查询。如果一个查询想要扫描一个表，而另一个查询已经在执行此操作，那么DBMS会将第二个查询的游标附加到现有游标上。DBMS记录第二个查询开始的位置，以便在扫描到表末尾时再从头开始完成扫描。

如此时有Q1扫描全表

<img src="https://pic4.zhimg.com/80/v2-ceed4a50f09bcc1003ef61221055413e.png" style="zoom:80%;" />


扫描到page3时，Q2来了，Q2也想扫描全表

<img src="https://pic4.zhimg.com/80/v2-17000f63cd45661b5a2d58aa0f8f9685.png" style="zoom:80%;" />


此时可以将Q2和Q1合并扫描

<img src="https://pic4.zhimg.com/80/v2-b985c8deb8c2cb930f561a6782c0b0c2.png" style="zoom:80%;" />


等到扫描完的时候，Q2由于还有page0-page2没有扫描，需要从头开始扫描，最终扫描到page2结束

<img src="https://pic4.zhimg.com/80/v2-37ef3ecba4f03e60f5679e68ad4f3084.png" style="zoom:80%;" />


### Buffer Pool Bypass

顺序扫描运算**不会**将提取的页面存储在缓冲池中，以避免开销。

在操作需要读取磁盘上连续的大量页面时表现很好。

Buffer pool bypass 也可用于临时数据（排序、连接）。

这也被叫做 "light scans"

## OS Page Cache

我们知道OS的文件系统有一层buffer layer，大部分磁盘操作都会经过OS维护的buffer cache。

但是大多数DBMS使用direct I/O，跳过使用OS cache。OS看这些就是一堆没有意义的数据，而DBMS更清楚这些数据的语义，能更好地指定策略。

DBMS跳过OS的cache可以避免pages的冗余拷贝、可以设计特定的替换策略。

不过Postgres使用了OS的buffer cache。


![Image](https://pic4.zhimg.com/80/v2-996574c03947f229e7a0bc9a2bbb2199.png)

## Buffer Replacement Policies

当DBMS需要清理一些frame，为新的page腾出空间，就需要将buffer pool中的page置换出去。

替换策略需要有高正确性，速度快，并且减少更新meta-data的开销。

### Least Recently Used (LRU)

大名鼎鼎的LRU。将最后一次使用时间最旧的page置换出去。

但是精确LRU很难有高效的实现。

### CLOCK

是LRU的近似方法，不需要对每个page维护时间戳。

在CLOCK策略中，每个页都有一个reference bit，当这个page被使用的时候，ref设置为1.

为了更好理解，可以想象这些page围成一个圈，中间有个指针在转，指向一个page的时候，如果ref是1，则置为0；如果ref是0，就直接置换出去。

如此时page1被访问了，ref置为1：

![Image](https://pic4.zhimg.com/80/v2-b6c10526844f411cea79342baa07d10e.png)

然后指针指向page1，ref重新被置为0：


![Image](https://pic4.zhimg.com/80/v2-6cd7fdad79dda2a6042f2be0fc648652.png)

之后指针指向page2，因为page2的ref是0，被置换出去

![Image](https://pic4.zhimg.com/80/v2-06e01f862fa61c9189c27c32351c8c42.png)

### Alternatives

LRU和CLOCK策略易受**sequential flooding**的影响。

因为顺序扫描会扫描表的每一个页面，因此，我们最近使用的页面可能是最不重要的页面（因为顺序扫描一般扫描每个页面只取一次），而之前的页面可能对于另外的query更加重要，这可能会导致LRU和CLOCK替换策略的效率降低。

#### LRU-K

跟踪最后K次引用的历史作为时间戳，并计算后续访问之间的间隔。此历史记录用于预测下一次访问页面的时间。有点像智能方法。

#### Localization

DBMS**根据每个事务/查询**选择要逐出的页面。这将最大限度地减少每次查询对缓冲池的污染。

例如postgres对每个单独的查询维护了一个小的ring buffer

#### Priority Hints

DBMS了解哪些page在query执行的过程中是重要的，比如在索引查询的过程中，根节点是重要的，那么我们可以一直保持根节点在buffer pool中。

### Dirty Pages

当我们置换page时，对于未修改的页，直接覆盖即可。而对于脏页，我们则必须将其修改部分写到磁盘（数据库持久化的要求）。

当缓冲池内存不足时。如果选择删除“干净”的页，速度是很快的。但是如果选择删除脏页，那么需要等待其写入磁盘，这个操作是比较慢的。

DBMS可以周期性的遍历page table，将脏页写回磁盘。

需要小心，系统不能在log records写入磁盘之前，将脏页写回。（要考虑事物的原子性，跟OS中log要求相同）

## Other Memory Pools

DBMS除了元组和索引之外，还需要存储其他东西。根据实现，其他内存池可能不disk-backed的。如

* Sorting + Join Buffers 
* Query Caches 
* Maintenance Buffers 
* Log Buffers 
* Dictionary Caches

## Conclusion

DBMS总是比OS管理内存更好。

使用query plan的语义可以做出更好的决策：

* Evictions
* Allocations
* Pre-fetching

