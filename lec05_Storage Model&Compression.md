#! https://zhuanlan.zhihu.com/p/598893434
- [Lec05 Storage Model \& Compression](#lec05-storage-model--compression)
  - [Database Workloads](#database-workloads)
  - [Storage  Model](#storage--model)
    - [N-Ary Storage Model (NSM)](#n-ary-storage-model-nsm)
    - [Decomposition Storage Model (DSM)](#decomposition-storage-model-dsm)
  - [Data Compression](#data-compression)
    - [Naive Compression (Block-level)](#naive-compression-block-level)
  - [Columnar Compression (Column-level)](#columnar-compression-column-level)
    - [Run-Length Encoding (RLE)](#run-length-encoding-rle)
    - [Bit-Packing Encoding](#bit-packing-encoding)
    - [Mostly Encoding](#mostly-encoding)
    - [Bitmap Encoding](#bitmap-encoding)
    - [Delta Encoding](#delta-encoding)
    - [Incremental Encoding](#incremental-encoding)
    - [Dictionary Compression](#dictionary-compression)
  - [Conclusion](#conclusion)


# Lec05 Storage Model & Compression

## Database Workloads

**OLTP: Online Transaction Processing**

指每次只读或写**少量数据**的操作。

通常OLTP workload处理写比读多。

一个例子是Amazon商城，用户可以向购物车添加东西，可以购买，这些动作只会影响他们自己的账号。 

**OLAP: Online Analytical Processing**

指每次读**很多数据**执行很复杂的查询或聚集。

一个例子是Amazon计算下雨的时候卖的最多的物品。

**HTAP: Hybrid Transaction + Analytical Processing**

上面两者的结合。

<img src="https://pic4.zhimg.com/80/v2-32375cf9b9c2b75cb1d3568bd96b051d.png" style="zoom:80%;" />


## Storage  Model

有不同的方式将tuple存入页中，之前我们假设的都是**n-ary storage**(aka row storage)，即是一个元组一个元组存的，每个元组的属性的值都存在相邻的位置。

### N-Ary Storage Model (NSM)

在n-ary storage model中，DBMS在一个页中连续的存储一个元组的所有属性。这个方法对于OLTP workloads来说很理想，因为OLTP workloads需要对于一个实体大量的insert和transactions。它只需要fetch一次就可以得到一个tuple的所有属性。如下图

<img src="https://pic4.zhimg.com/80/v2-18d066c7d3176d8dd37de5057d478607.png" style="zoom:67%;" />


但是对于OLAP workloads不友好，因为OLAP查询可能只需要一个或几个属性，不需要所有的属性，浪费了磁盘I/O，如下图

<img src="https://pic4.zhimg.com/80/v2-27dd9270c1ad2cdb04f69a1715fa72a9.png" style="zoom: 80%;" />

**Advantages:**

* Fast inserts, updates, and delete
* Good for queries that need the entire tuple

**Disadvantages:**

* Not good for scanning large portions of the table and/or a subset of the attributes.

### Decomposition Storage Model (DSM)

DBMS按照列存储，将所有元组的单独一个属性连续存储。也叫做列存储。它适用于OLAP

workloads，有很多对表属性的子集执行大规模扫表的read-only queries 。

<img src="https://pic4.zhimg.com/80/v2-47bc98be401d1c9e81f6ef3170089e6c.png" style="zoom:80%;" />

<img src="https://pic4.zhimg.com/80/v2-b91cc7e0fb9c5b614de6ede840f02bea.png" style="zoom:80%;" />


如何知道哪个值对应哪个tuple呢？有两种方法

1. Fixed-length Offsets

   即属性的每个值长度都一样，这样不同属性间的值就能一一对应。更快。

2. Embedded Tuple Ids

   不太常见。每个值存储的时候也要存tuple_id

    <img src="https://pic4.zhimg.com/80/v2-6e334f512aa19795e00a8afb91bff4af.png" style="zoom:80%;" />


**Advantages:**

* Reduces the amount wasted I/O because the DBMS only reads the data that it needs.
* Better query processing and data compression (more on this later).

**Disadvantages:**

* Slow for point queries, inserts, updates, and deletes because of tuple splitting/stitching.

目前几乎所有的系统都有实现DSM存储模型。



## Data Compression

Compression被广泛使用在disk-based DBMS中。因为磁盘I/O通常是性能的主要瓶颈。而DBMS可以压缩数据以提高每次I/O操作的数据移动效率

压缩不仅可以提高I/O效率，减少内存占用，也可以在查询执行阶段减小CPU cost。



现实中数据的特点：

* 数据集的属性值往往具有高度偏斜的分布

* 数据集往往在同一元组的属性之间具有高度相关性



Database Compression 的目标：

1. Must produce fixed-length values.

   DBMS可以通过offset访问数据

2. Postpone decompression for as long as possible during query execution.

   (late materialization)

3. Must be a **lossless scheme**

   如果用户想要有损压缩，需要在应用层级使用



**Compression Granularity:**

* Block-level

  对一个block进行压缩

* Tuple-level

  对整个tuple进行压缩，只能用于NSM

* Attribute-level

  对一个tuple中单独的一个属性值进行压缩

* Column-level

  对多个tuple的一个或几个属性的多个值进行压缩，只能用于DSM



### Naive Compression (Block-level)

用general purpose algorithm进行压缩，如gzip, LZO, LZ4等。工程师通常选择有最小压缩率的方法，以换取更快的压缩和解压缩。

一个用naive compression的例子是MySQL InnoDB。DBMS压缩磁盘页，并填充到2的幂次KB（2,4,6,8KB），然后放到buffer pool，如果DBMS想要读取数据，存在buffer pool中的压缩数据就要被解压缩。

 由于访问数据需要对压缩数据进行解压缩，这限制了压缩方案的范围。如果目标是将整个表压缩成一个巨大的块，那么使用简单的压缩方案是不可能的，因为每次访问都需要对整个表进行压缩/解压缩。

另一个问题是，这些naive的方案也没有考虑数据的高级语义。该算法既不关心数据的结构，也不关系查询计划如何访问数据。因此，这就失去了late materialization的机会，因为DBMS无法判断何时能够延迟数据解压缩。

而我们期望是，DBMS可以不解压缩，直接在压缩数据上操作。

<img src="https://pic4.zhimg.com/80/v2-046c7400153ddf5e5b747b4f13b90735.png" style="zoom:80%;" />


## Columnar Compression (Column-level)

### Run-Length Encoding (RLE)

将一类中相邻的相同的值变成一个triplets:

* 属性值
* 相同值的这一段开始的位置
* 相邻的相同值的个数

DBMS可以智能地提前进行排序，最大化压缩率。

假设我们有一张表有id和sex两列，可以对sex进行压缩，如下图，

<img src="https://pic4.zhimg.com/80/v2-aa1423d4b6ebc873ac1128f697235b32.png" style="zoom: 67%;" />

可以进行排序最大化压缩率

<img src="https://pic4.zhimg.com/80/v2-0793d009cbd53218a32633afb71e34d3.png" alt="压缩前" style="zoom:67%;" />


<img src="https://pic4.zhimg.com/80/v2-e9df4fce0130a532b13a138d3b256f90.png" alt="压缩后" style="zoom:67%;" />


### Bit-Packing Encoding

当属性的值总是小于声明的最大值时，将这些值存储在更小的数据类型中。

比如我们之前声明用int64存储了这些数：


![Image](https://pic4.zhimg.com/80/v2-f81d3d5d7f9fa5c276aa5b108c69f1e5.png)

发现这些数都达到不了int64最大大小，于是进行压缩：


![Image](https://pic4.zhimg.com/80/v2-87e799c9b05b2b2146621468ddb89a5e.png)

### Mostly Encoding

Bit-packing Encoding的变体，如果有一个值超出了改变数据类型后能表示的范围，那么维护一个查找表去存储这些值


![Image](https://pic4.zhimg.com/80/v2-039768c1c2468e4899fc15fffb0586be.png)

### Bitmap Encoding

对于某个特定的属性，对每个值维护一个bitmap，来表示某个元组对应的属性是否是这个值。

比如性别，可以为男和女分别维护一个bitmap，如下图：


![Image](https://pic4.zhimg.com/80/v2-3ab4147592b52988497a38701fe4791a.png)

由此可见，这个方法只适用于可选的值比较少的时候。

### Delta Encoding

不记录确切值，而是记录一列中与上一个值的差异：

![Image](https://pic4.zhimg.com/80/v2-fd71bbca9960b05357c1fbfca942157b.png)

与RLE方法一起使用压缩效果更好。

### Incremental Encoding

Delta Encoding的一种，可以记录与上一个值共同的前缀或者后缀的长度，sort之后压缩效果更好


![Image](https://pic4.zhimg.com/80/v2-0ec89c9ec69a4b92d087a2822dfd8c00.png)

### Dictionary Compression

这是最常用的压缩方法。

构造一个数据结构，将变长的值映射到小的整数id，需要支持快速编解码，并且支持range queries。


![Image](https://pic4.zhimg.com/80/v2-93f9c0e513fa7c790bc0c672814a0497.png)

## Conclusion

选择正确的 storage model 是很重要的：

* OLTP，选择Row Store
* OLAP，选择Column Store



DBMS可以结合不同方法以更好地压缩。Dictionary encoding是最常用的压缩方法，因为它不需要提前排序。



我们之前讲了数据库存储引擎有两个主要问题，第一个是DBMS如何表示磁盘中的数据库文件，第二个是DBMS如何管理内存和磁盘之间的数据移动。我们到目前已经讲完了第一个问题，之后将会讲第二个问题。