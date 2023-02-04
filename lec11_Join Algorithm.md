#! https://zhuanlan.zhihu.com/p/603527656

- [Lec11 Join Algorithm](#lec11-join-algorithm)
  - [Joins](#joins)
    - [Operator Output](#operator-output)
    - [Cost Analysis](#cost-analysis)
  - [Nested Loop Join](#nested-loop-join)
    - [Simple Nested Loop Join](#simple-nested-loop-join)
    - [Block Nested Loop Join](#block-nested-loop-join)
    - [Index Nested Loop Join](#index-nested-loop-join)
    - [Summary](#summary)
  - [Sort-Merge Join](#sort-merge-join)
  - [Hash Join](#hash-join)
    - [Basic Hash Join](#basic-hash-join)
    - [Partitioned Hash Join(GRACE hash join)](#partitioned-hash-joingrace-hash-join)
  - [Conclusion](#conclusion)


# Lec11 Join Algorithm

## Joins

在关系数据库设计的时候我们就想要避免冗余，因此需要对表进行规范化，而当我们想重现原始表时，就需要**连接**。
本课关注双表的**内等值**连接。

原则上我们希望，连接时将小表放到左侧（作为外表）。

### Operator Output
对于符合连接条件的元组 $r\in R$ 和 $s \in S$，连接算子将元组$r$和$s$连接成一个新的元组。在现实中，输出元组的内容可以有很多，取决于DBMS的查询处理模型、存储模型和查询本身。
下面列举了几种输出元组的内容：

1. DATA
   即将外元组和内元组的属性值复制到新的输出元组中：

   <img src="https://pic4.zhimg.com/80/v2-c2f24d0c78b1485dff432ce229f70059.png" style="zoom:80%;" />


   这样也叫做**提前物化**。这种方式的优点在于，未来的算子需要回到原表中获取一些其他数据；缺点在需要更多的内存。DBMS可以做一些额外的计算，也可以忽略查询过程中以后也不会需要的属性。

2. RECORD IDs

   即将两个元组的码复制进新的元组里面。

   <img src="https://pic4.zhimg.com/80/v2-9a9fe7a294e0b05ab2f0d41151a227c0.png" style="zoom:80%;" />

   也叫**推迟物化**。适用于列存储，因为这样DBMS不需要复制对于查询没有的属性。

### Cost Analysis

我们使用磁盘I/O数来作为性能评判的依据。（但我们没有计算将计算结果写到磁盘中的时间，因为各种算法的输出次数都一样）。

我们在下面的分析中假设

* 表**R**中有M页，m个元组
* 表**S**中有N页，n个元组

下面介绍的连接算法，可能在某些情况下性能更好，但是没有一个算法可以在所有场景下表现良好。

## Nested Loop Join

也就是暴力循环算法。

总体上来看，这个嵌套循环算法主要由2个for嵌套，每个for分别迭代一个表，如果两个元组符合连接谓词，就输出它们。

在外循环的表叫做**外表**，在内循环的表叫做**内表**。以下的讨论中R表是外表，S表是内表。

DBMS总是希望小表当做外表，这里的“小”是指占用的页数或者元组个数。DBMS希望在内存中缓存尽量多的外表，在遍历内表时也可以使用索引加速。

### Simple Nested Loop Join

傻瓜式暴力两层大循环嵌套。

<img src="https://pic4.zhimg.com/80/v2-52207375dcfe21dd4e2fca542b79f440.png" style="zoom: 80%;" />

对于每一个外表的元组，都遍历一遍S表，没有任何缓存机制，没有利用任何局部性。

这个的磁盘I/O花费是 $M + (m \times N)$ 。

M是读入R表的次数，然后对于R表的m个元组，每个都扫描一遍S表。

这样用时为一个多小时：

<img src="https://pic4.zhimg.com/80/v2-84183956fb6a0938f83f5ed736497205.png" style="zoom: 80%;" />


### Block Nested Loop Join

将外表和内表分块，能更好得利用缓存和局部性原理。对于外表的每一个块，获取内表的每一个块，然后将两个块内部进行连接操作。

<img src="https://pic4.zhimg.com/80/v2-4a97759bbc96d728501d3e82bfaea854.png" style="zoom: 80%;" />


这样每一个外表的块扫描一次内表，而不是每一个外表元组扫描一次内表，这样减少了磁盘I/O。

这里的外表要选择页数少的，而不是元组个数少的。

假设对于外表的划分是一页为一块，这时的花费是$M+(M\times N)$。M是读入外表的花费，然后对于每一个页遍历一遍内表。

时间花费为50s：

<img src="https://pic4.zhimg.com/80/v2-f18a12e34b77c4095c052fe3a1e69a46.png" style="zoom:80%;" />

假设我们可以使用B个buffer？我们将一个buffer用来存储输出结果，一个buffer用来存储内表，剩下的B-2个buffer用来存储外表，这样就将外表的一个block大小设置为B-2个页。

<img src="https://pic4.zhimg.com/80/v2-20cd0d5621c7a7af8a4cff9e741b48fb.png" style="zoom:80%;" />

这样的花费就是$M+(\lceil \frac{M}{B-2} \rceil \times N)$。M为读入外表的花费，然后对于外表每一个块（有$\lceil \frac{M}{B-2} \rceil$个块），都扫描一遍内表。

这样做的用时为0.15s：

<img src="https://pic4.zhimg.com/80/v2-de5f6e37370571346ed2724aab637ce9.png" style="zoom:80%;" />


### Index Nested Loop Join

原先的嵌套循环方法慢的原因是，需要线性扫描一遍内表，如果内表有索引的话，就不用每次线性扫描了。

DBMS可以临时构建一个索引，或者利用已有的索引。

<img src="https://pic4.zhimg.com/80/v2-6ace3b160993379cca6f7f939f3a3333.png" style="zoom:80%;" />



假设每一次索引查询都需要$C$的花费，那么总花费就是$M+(m\times C)$。



### Summary

关键点：

1. 选择小表作为外表
2. 缓存尽可能多的外表，以减少内表扫描次数
3. 扫描内表（可以利用索引）

## Sort-Merge Join

总体上分为两个阶段：

1. Sort
   * 对两个表按照连接的键值分别进行排序。
   * 可以使用上一节介绍的外排序
2. Merge
   * 用双指针遍历两个表，输出匹配的元组
   * 可能需要指针“撤回”

算法描述如下：

<img src="https://pic4.zhimg.com/80/v2-20d7f8494a6b2fc862b1f77d2cd7148f.png" style="zoom:80%;" />

根据上节课，可以知道花费如下：

<img src="https://pic4.zhimg.com/80/v2-1bd63d6dd9355e3e20b3160128f02fd7.png" style="zoom:80%;" />

分成两部分，排序和合并。

这样的方法用时只需要0.75s：

<img src="https://pic4.zhimg.com/80/v2-e4c601f810ebd0fa1069a8979891f7ca.png" style="zoom:80%;" />


但是最坏情况下，也就是所有的键值都相等，Merge花费会退化到$M\times N$。

什么时候使用Sort-Merge Join呢:

1. 当两个表**已经**按照join的键值排好序的时候（有可能在之前显示使用sort，或者使用索引扫描过join的键值）
2. 输出结果**必须**按照join的键值排序的时候

## Hash Join

使用哈希表，基于join的键值将元组分成小块。这减少了每一个元组进行比较的次数。Hash Join只能用在等值连接中。

### Basic Hash Join

分成两个阶段：

1. Build
   * 扫描外表，使用哈希函数$h_1$，作用于join的键值得出一个哈希表。
   * 可以使用我们之前讨论过的哈希表，但是实际上线性探测哈希表表现最好。如果提前已知外表的大小，则可以用静态哈希表；如果不知道，还是应该用动态哈希表，并且支持溢出页。
2. Probe
   * 扫描内表，对每个元组使用哈希函数$h_1$，计算到哈希表的对应位置，然后找匹配的元组。



过程如下，首先对外表利用哈希函数$h_1$建立哈希表：

<img src="https://pic4.zhimg.com/80/v2-204ceaeca70f81070e0331f3965b9b8f.png" style="zoom:80%;" />


然后扫描内表的所有元组，利用哈希函数$h_1$找到对应的桶，然后在桶中找到匹配的元组。



哈希表中的内容需要有：

1. Key
   * 因为我们需要在一个哈希表的桶中找到键值相等的元组。
2. Value
   * 根据实现不同，value不同
   * 和上面讨论过的Join的output类似，可以有提前物化和延迟物化两种实现



一个大小为N页的表，需要$\sqrt{N}$个buffer。使用一个附加系数f，最终的花费为$B\times \sqrt{f\times N}$。

**Optimization：Probe Filter**

可以使用布隆过滤器来优化查询，因为当键值不在哈希表中的时候，可以不用去查。

于是在哈希表中找之前，先使用布隆过滤器（一个概率数据结构，可以放入CPU cache，可以得知一个元素在不在，不会出现假阴性，但是会出现假阳性），如果布隆过滤器结果是不存在，那么就不用去哈希表中找。

布隆过滤器插入阶段，使用k个哈希函数，将过滤器对应位设为1；查询阶段，查看过滤器k个哈希函数值对应的位是否都是1.

<img src="https://pic4.zhimg.com/80/v2-8de5185843fdb12cb4d3d797fc7e91f9.png" style="zoom:80%;" />

<img src="https://pic4.zhimg.com/80/v2-5a030cc04e34c133756d3eb7cd919916.png" style="zoom:80%;" />

<img src="https://pic4.zhimg.com/80/v2-87fb99d936198f7c4a99b84190c5ae75.png" style="zoom:80%;" />

出现了假阳性：

<img src="https://pic4.zhimg.com/80/v2-7da2bbd287ebda523102e55a24061754.png" style="zoom:80%;" />


### Partitioned Hash Join(GRACE hash join)

当内存不足以放下整个哈希表时，DBMS会随机还如换出哈希表页，可能会导致低性能。

Grace hash join是基础哈希连接的一个扩展，也将内表制作成可以写入磁盘的哈希表。

1. Build Phase

   对两个表的join键值使用相同的哈希函数分别进行哈希

2. Probe Phase

   比较两个表对应partition的元组，输出连接结果

先对外表和内表生成哈希表：

<img src="[D:\学习工作\学习资料\其他课程\CMU15445\notes\lec11_Join Algorithm.assets\image-20230204215920161.png](https://pic4.zhimg.com/80/v2-1662c735a755dc4032601b0e8f6492cb.png)" style="zoom:80%;" />

然后对两个对应的buckets进行匹配：

<img src="https://pic4.zhimg.com/80/v2-030f1df9e29bb7712f1bfc03a2eef4be.png" style="zoom:80%;" />


如果一个bucket也不能放到内存中，那么就使用 recursive partition，将这个大bucket使用另一个哈希函数再进行哈希，使其能放入内存中。

例如，下面这张图中bucket1太大了，没办法放进内存中：

<img src="https://pic4.zhimg.com/80/v2-4d5cec78099714652560d68b851ad4b5.png" style="zoom:80%;" />


我们就对bucket1再进行一次hash，使它能放到内存中：

<img src="https://pic4.zhimg.com/80/v2-11d0992fdb912f3132a82ba551bdc1c1.png" style="zoom:80%;" />

然后再对另一个表对应的bucket进行匹配，注意，这里右边的bucket1，要与左边的1'、1''、1'''都进行匹配

<img src="https://pic4.zhimg.com/80/v2-9506d78fe6088e65d57a3405e99d30b0.png" style="zoom:80%;" />


假设我们有足够的buffer，这样的花费为$3(M+N)$，其中partition阶段，需要读、写两个表各一次，花费为$2(M+N)$，probing阶段要读两个表各一次，花费$M+N$。

这样需要0.45s:

<img src="https://pic4.zhimg.com/80/v2-efcff97d6e6b80c937d178472a2b68d7.png" style="zoom:80%;" />

也可以使用融合哈希连接：假设键值是有偏斜的，那么DBMS让多的partition放在内存中，然后理解进行匹配操作，而不是将它们溢出放到磁盘中。（难以实现，现实中少见）

## Conclusion

上面讨论的连接算法花费如下：

<img src=https://pic4.zhimg.com/80/v2-9b685565363e30c8385f9b4a668f9e6d.png" style="zoom:80%;" />

哈希方法几乎总是比排序的方法好，但是有些情况下排序的方法更好：当数据已经是排好序的时候、当数据结果总归要进行排序的时候。

DBMS一般都会实现这两种方法。
