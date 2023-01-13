#! https://zhuanlan.zhihu.com/p/598890271
- [Lec03 Database Storage 1](#lec03-database-storage-1)
  - [Storage](#storage)
    - [Storage Hierarchy](#storage-hierarchy)
  - [Disk-Oriented DBMS Overview](#disk-oriented-dbms-overview)
  - [DBMS vs. OS](#dbms-vs-os)
  - [File Storage](#file-storage)
    - [Database Pages](#database-pages)
    - [Database Heap](#database-heap)
  - [Page Layout](#page-layout)
    - [Header](#header)
    - [Slotted Pages](#slotted-pages)
    - [Record IDS](#record-ids)
  - [Tuple Layout](#tuple-layout)
  - [Conclusion](#conclusion)

# Lec03 Database Storage 1

从这节课开始将会学习如何构建DBMS，我们将会cover这些内容：

* Storage
* Execution
* Concurrency Control
* Recovery
* Distributed Database

## Storage

我们将会学习“disk-oriented” DMBS架构，即主要的存储位置在非易失的硬盘上。（除此之外还有"memory-oriented"的，如Redis）。

### Storage Hierarchy

<img src="https://pic4.zhimg.com/80/v2-0301f75d75cf513eed24955f24e77084.png" style="zoom:67%;" />

在存储体系中越靠上，离CPU越近，存取的速度也越快，但是同时存储空间也越小，越贵。越靠下存储空间越大，但是越慢，而且速度是指数级变慢。register和caches都是在CPU附近，DRAM是内存，SSD、HDD、Network Storage都是属于外存。

<img src="https://pic4.zhimg.com/80/v2-1011d0071452eccfdb4aed59c8bf843a.png" style="zoom: 40%;" />

**易失设备：**

* 断电数据会丢失
* 支持快速随机存取，并且按照**字节**编址，程序可以存取任意字节地址

**非易失设备：**

* 不需要持续的电源来存储数据
* 按照**块**编址。意味着要修改一个内容，要读一个块进来，然后修改一个内容，再把整个块写回去
* 是顺序访问的，可以一次读几个块

还有一些**新兴的存储设备：**

* Fast Network Storage, 云计算新兴起的一种存储器，比本地机械硬盘还要快，是通过万兆光纤接入电脑的，平时说的买的云服务器配载的云硬盘，就是这种存储器。

* Persistent Memory，非易失的内存，内存断电之后东西不丢，但是现在还没有大规模用，非常贵。

![Image](https://pic4.zhimg.com/80/v2-cd3a2a27c5a4b5f0472fa48f1a6736fd.png)

上面这张图展示了存取不同设备的用时，可见根据存储体系结构设计程序是多么的重要。

**System Design Goal：**

* 让DBMS可以处理比内存空间更大的数据库，就是用Disk-Oriented DBMS。

* 数据库不可能时时刻刻把数据写到硬盘上，因为太慢了，也不可能直接操作硬盘。我们的DBMS就要负责解决如何在非易失的磁盘和易失的内存之间移动数据，来减少长时间的等待和性能退化。

* 对非易失存储的连续存取远远快于随机的存取。而用户的请求可能是不连续的，DBMS想最大化连续存取，减少读写不连续的页。

## Disk-Oriented DBMS Overview

数据库都是在磁盘上的，数据库文件的数据被组织成**页**（page），第一个页是目录页（directory page）。DBMS需要将数据移入内存中来操作数据。这是通过利用**buffer pool**来管理内存和磁盘间数据的移动。

DBMS有一个**execution engine**来执行查询，execution engine向buffer pool索要特定的页，buffer pool将会负责将页移入内存中，并给execution engine指向这个页的指针。

接来下的几节课将重点讲disk上的数据存储，之后会讲buffer pool和execution engine。

<img src="https://pic4.zhimg.com/80/v2-33c488feb98aef8fbc469cc1a83760d8.png" style="zoom:67%;" />


## DBMS vs. OS

前面提到了DBMS的设计目标，支持超过可用内存的数据库，要仔细管理磁盘的使用，不希望获取磁盘数据时有长时间的停顿，而是应该处理其他查询。

这个设计目标有点像**虚拟内存**，有非常大的地址空间，并且OS可以从磁盘中读取页。其中一种方式是我们熟知的**mmap**，可以将文件的内容映射到进程的虚拟地址空间中，在这里OS负责在磁盘和内存中移动数据。

**mmap的问题：**

* Transaction Safety

  OS可能在任意时间将脏页写回，打破了事物的原子性

* I/O Stalls

  DBMS不知道哪个页在内存中。而且一旦发生了page fault，就会阻塞进程

* Error Handling

  DBMS可能需要在读数据进内存时检查checksum，如果有错误可以及时处理，但是OS隐藏了数据进入和出去的过程。

* Performance Issues

  OS数据结构会有竞争。TLB击落（当处理器更改虚拟地址段到物理地址的映射时，需要通知其他的处理器）。会成为高性能数据库的瓶颈。

**DBMS可以比OS做得更好：**

* 可以将脏页在正确时间写回磁盘
* 可以prefetching
* 定制的buffer replacement policy
* 进程、线程调度

**OS is not friend.**

---

数据库存储总共有两个问题：

1. DBMS如何表示存储在磁盘上的数据库的文件
2. DBMS如何管理内存，并在内存和磁盘之间移动数据

这节课聚焦于第一个问题，接下来将会包含三个议题：

1. File Storage
2. Page Layout
3. Tuple Layout

逐渐细化，讲解DBMS如何表示数据库的文件。

## File Storage

在最基本的形式中，DBMS将数据库以**文件**的形式存储在磁盘上。

OS只知道这些文件的存在，而不知道这些文件是怎么样编码的，只有DBMS知道如何读取这些文件。

DBMS的 **storage manager** 负责管理数据库的文件，它们可以调度读和写来提高页的时间和空间局部性。它将文件表示成**页**的集合，它维护了哪些数据已经被读入和写入页，也维护了这些页还有多少空闲空间。

### Database Pages

**页**是**定长**的一块数据。页可以包含不同类型的数据（tuples，meta-data，indexes，log records等）。大多数系统会固定每个页的功能，不会一个页存放两种不同类型的数据。一些系统要求页是*self-contained*，即页内有解释自己的meta-data。

每个页都需要一个标识，即page id。如这个数据库只有一个文件，那么page id可以是offset。大多数DBMS有一层indirection layer，将page id映射到文件路径和offset。系统的上层可能要求读取特定的page number，然后storage manager就将page number转换成文件和offset去找这个页。

现在遇到了三个关于页的术语：

* hardware page (usually 4KB)

* OS page (usually 4KB)

* Database page (1-16KB)

  如SQLite, DB2, Oracle都是4KB， Postgresql是8KB，MySQL是16KB。

存储设备保证了大小为hardware page的数据存储的原子的，要么hardware page个数据都写，要么都没写，如果Database page大于hardware page的话，需要额外的努力来保证原子性了。

### Database Heap

有很多方式在磁盘上寻找DBMS想要的页的位置，heap file organization是一种方式。

**heap file** 是一个页的**无序**集合，而且页中的元组也是随机的顺序。可以支持创建、查找、修改、删除页，并且必须支持迭代页（iterating over all pages）。

如果是一个文件的话，寻找页很简单，能很轻松算出来每个页的offset：

![Image](https://pic4.zhimg.com/80/v2-75858b4b93c433ff07fa98b3c0298d2a.png)

如果有很多文件，就需要元数据记录哪些页在多个文件中存在，那些页有空余的空间。

DBMS维护一个特殊的页，记录数据库文件中每个页的位置，同时还要维护一些元信息，每个页中空闲slots的数量，空页的列表等。

<img src="https://pic4.zhimg.com/80/v2-c191f102369c02b5cb03f7fb877c16ae.png" style="zoom:67%;" />

## Page Layout

### Header

每个页都包含一个**header**，记录页内容的元信息：

* Page size
* Checksum
* DBMS version 
* Transaction visibility
* Compression Information
* self-containment（Oracle就需要这个）

有两种方法组织页内数据的方式：

* Tuple-oriented
* Log-structured （下节课讲）

### Slotted Pages

最简单的一种方法是记录page内tuple的数量，添加的时候直接往末尾加，但是如果删除一个中间的元组，则想填补上这个位置就需要从头开始线性扫描；并且如果元组不是定长的，就会造成很多内部碎片。

一种存储元组的方法叫做Slotted pages。

* Header维护已经使用的slots的数量，最后的slot的起始位置的offset，和一个slot array，slot array记录了每个元组的起始位置，将'slots'映射到元素起始位置offset。
* slot array从头开始，元组存储的位置从尾部开始，如果slot array和tuple data相遇了，即认为填满了这个page。

<img src="https://pic4.zhimg.com/80/v2-9439586a4f308e80db598af868e3a3eb.png" style="zoom:67%;" />


如果删除某个元组产生内部碎片，我们可以移动其他的元组，但是元组的slot不会变，如删除了tuple3之后，将tuple4移动

<img src="https://pic4.zhimg.com/80/v2-49fbf7e2e128b6647a97d2af6ba08e79.png" style="zoom:67%;" />


### Record IDS

DBMS需要维护record id，是物理概念的tuple，可以通过record id找到对应的tuple。

通常record id可以是 page_id + offset/slot

## Tuple Layout

Tuple本质上是字节的序列，DBMS负责将这些字节解释成属性和值。

**Tuple header:** contains meta-data about the tuple

* Visibility info (for concurrency control)
* Bit Map for NULL values
* 不需要存储有关schema的元数据

**Tuple Data:** Actual data for attributes

* 通常属性按照创建表时的顺顺序来存储
* 大多数DBMS不允许tuple超过page size
* ![](https://pic4.zhimg.com/80/v2-0b2135676ca4a04f0b93b2eb0092333c.png)


**Denormalized Tuple Data**

* 如果两个表相关，DBMS可以“pre-join”这些表，这些表将会存储在一个页中。
* 可以更快得读取
* 使更新更加耗时，因为DBMS对于每个tuple需要更多的空间



## Conclusion

* Database is organized in pages. 
  * page size

* Different ways to track pages. 
  * Heap file
  * Directory, link list...

* Different ways to store pages. 
  * Header
  * Tuple-oriented
    * slotted pages
  * Log-structured

* Different ways to store tuples.
  * Header
  * Data
  * Denormalized

插入一个tuple的过程：

1. 检查page directory，找到一个有空闲slot的page
2. 如果page不在内存中的话，需要从磁盘中读取
3. 检查slot array寻找page中的合适的空闲位置

用tuple的record id来更新它：

1. 检查page directory寻找page的位置
2. 如果page不在内存中的话，需要从磁盘中读取
3. 用slot array查找在page中的offset
4. 如果新数据能在原地放下的话，覆盖现有的数据