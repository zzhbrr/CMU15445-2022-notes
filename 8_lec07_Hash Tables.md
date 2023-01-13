#! https://zhuanlan.zhihu.com/p/598896263
- [Lec07 Hash Tables](#lec07-hash-tables)
  - [Data Strcutures](#data-strcutures)
  - [Hash Table](#hash-table)
  - [Hash Functions](#hash-functions)
  - [Static Hasing Schemes](#static-hasing-schemes)
    - [Linear Probe Hashing](#linear-probe-hashing)
    - [Robin Hood Hashing](#robin-hood-hashing)
    - [Cuckoo Hashing](#cuckoo-hashing)
  - [Dynamic Hashing Schemes](#dynamic-hashing-schemes)
    - [Chained Hashing](#chained-hashing)
    - [Extendible Hashing](#extendible-hashing)
    - [Linear Hashing](#linear-hashing)
  - [Conclusion](#conclusion)


# Lec07 Hash Tables

![Image](https://pic4.zhimg.com/80/v2-132c615a5b8ff27c9d4652774062edc6.png)

现在学完了Disk Manager和Buffer Pool Manager，接下来要学DBMS执行器如何从页中读写数据了， 也就是Access Methods。

我们要学习两种数据结构：Hash Tables和Trees，本节课将学习Hash Tables。

## Data Strcutures

DBMS需要使用数据结构来维护各种系统的内部数据。比如：

* Internal Meta-Data：这些数据记录了数据库的一些关键信息和系统的状态。比如页表，页目录等。（复习：page table是in-memory的，将page id映射为frame position，page directories要存在磁盘，将page id映射为数据库文件位置）
* Core Data Storage：用来保存数据库中的元组
* Temporary Data Structures：DBMS在执行一些查询时，会构建一些临时的数据结构来加快执行速度。例如在执行join的时候创建哈希表
* Table Indices：表的辅助数据结构，帮助我们更快地找到表中的某些特定元组。

在为DBMS实现数据结构时，需要考虑两个主要的问题：

1. Data organization

   我们如何在内存/页面中布局数据结构，以及存储哪些信息以支持高效访问

2. Concurrency

   如何使多线程同时访问数据结构，而不造成问题

## Hash Table

哈希表是一种抽象数据类型，它使用哈希函数将数据从键(key)映射到值(value)。提供平均$O(1)$的时间复杂度，但是最坏情况下是$O(n)$的。但是有大常数，这个常数是现实实现需要考虑的。

哈希表实现有两部分：

1. Hash Function
   * 如何将大的键空间映射到较小的域。
   * 需要考虑执行效率和冲突率之间的权衡。一个极端是，我们有一个哈希函数，它总是返回一个常数（非常快，但一切都是冲突）。在另一个极端，我们有一个“完美”的哈希函数，其中没有冲突，但计算起来需要非常长的时间。
2. Hashing Scheme
   * 如何解决hash后的键冲突
   * 需要考虑分配大哈希表以减小碰撞率，与提供其他机制处理碰撞之间的权衡

接下来本讲的内容是：

* Hash Functions 
* Static Hashing Schemas
* Dynamic Hashing Schemas

## Hash Functions

不太想讲。。。

哈希函数接受一个键输入，输出一个整数。

DBMS中不使用加密哈希函数，如SHA-256，因为计算量还是太大。我们想要的是计算快的、碰撞率低的哈希函数。

有几种常用的哈希函数：CRC-64, MurmurHash, Google CityHash, Facebook XXHash, Google FarmHash。

## Static Hasing Schemes

静态哈希方案是哈希表大小固定的方案。这意味着如果DBMS耗尽了哈希表中的存储空间，那么它必须从头开始重建更大的哈希表，这非常耗时。通常，新哈希表的大小是原始哈希表的两倍。

我们一般用期望元素个数的两倍作为哈希表大小。

### Linear Probe Hashing

中文也叫开放地址哈希，这是最基本的哈希方案，通常也是最快的。

插入时，哈希函数将键映射到slot。当发生碰撞时，我们线性搜索相邻的slot，直到找到一个未被占用的slot。

对于查找，我们可以检查key哈希到的slot，并线性搜索，直到找到所需的条目。

对于删除，我们必须小心地从槽中删除条目，因为这可能会阻止将来的查找找到已放在现在空槽下面的条目。此问题有两种解决方案：

1. 最常见的叫Tombstones，即标志这个位置的项逻辑上已经删除了。
2. 另一种方法是movement，即将删除元素之后的，有相同哈希值的项移动。

**Non-unique Keys:** 对于用一个key可能有不同values对应的时候，有两种方案:

1. Seperate Link List

   哈希表中存储一个指针，指向单独的存储空间，里面存有那些value

   <img src="https://pic4.zhimg.com/80/v2-070971f58f13eff5090f425697c17f1b.png" style="zoom:80%;" />


2. Redundant Keys

   这是最常见的方法，将key多次存在表中。

   <img src="https://pic4.zhimg.com/80/v2-fc50e1df4fa2178bcf89f8741a935a23.png" style="zoom:80%;" />


### Robin Hood Hashing

对线性探测哈希的改进，思路是尽可能减少每个key离它们应该在的位置距离的最大值（最小化最大值）。跟劫富济贫有点像。

这个方法中，每一项都顺便存储了离自己本来位置的距离。每次插入，如果插入的key离自己本来的位置的距离大于现在的slot中离它本来位置的距离，就插入这个slot，再重新尝试插入被替换的项。

比如，先插入A，A就在自己想在的位置上，于是距离值为0：

<img src="https://pic4.zhimg.com/80/v2-f6212f9625a8da37f05ec06b96f552e6.png" style="zoom:67%;" />

插入B，B也在自己想在的位置上：

<img src="https://pic4.zhimg.com/80/v2-82761ff6f5b820265882684971fae2e1.png" style="zoom:67%;" />


插入C，这时候C与A冲突了，记在A后面，距离为1

<img src="https://pic4.zhimg.com/80/v2-5e9c430125ce185bba5a044a7dd5d304.png" style="zoom:67%;" />

插入D，D和C冲突了，如果D放在C那里，D的距离为0，C的距离为1，D距离比C小，于是不替换，D放在C后面

<img src="https://pic4.zhimg.com/80/v2-fa7ea9f89fa78bdcc102ffc53a900561.png" style="zoom:67%;" />


然后插入E，E与A冲突，如果E放在A位置，距离与A一样都是0；如果放在C位置，距离与C一样都是1；如果放在D位置，E此时距离为2，大于D的距离1，所以用E替换D

<img src="https://pic4.zhimg.com/80/v2-6da633f7e3f9082c4a67bff3bb5cae7b.png" style="zoom:67%;" />

但是还没有完成E的插入，因为被替换的D还没有被放置，D本来应该放在C那里，这时D距离为0，小于C距离；如果放在E那里，跟放E时候讨论一样；所以只能放在E后面，这样完成了E的插入

<img src="https://pic4.zhimg.com/80/v2-e3a6b3b38f4d54674e56ce4276f0b078.png" style="zoom:67%;" />

### Cuckoo Hashing

cuckoo是杜鹃的意思，杜鹃把蛋下在别的鸟的巢里，这个方法可以把一项放在其他的哈希表中。。。

这种方法不使用单个哈希表，而是使用不同的哈希函数维护多个哈希表。哈希函数是相同的算法（例如，XXHash、CityHash）；它们通过使用不同的种子值为同一密钥生成不同的哈希结果。

当插入时，检查每个表，并选择一个有空闲槽的表插入（如果多个表都有槽，我们可以比较负载因子，或者更常见的是选择随机一个表）。

如果没有表有空闲的槽，我们选择（通常是随机的）并删除某个表中的旧项。然后，我们将旧项重新散列到另一个表中。（杜鹃，很形象）

在极少数情况下，我们可能会陷入循环。如果发生这种情况，我们可以使用新的哈希函数种子（不太常见）重建所有哈希表，或者使用更大的表（更常见）重建哈希表。

## Dynamic Hashing Schemes

静态哈希方法要求DBMS知道它想要存储的元素的数量。否则，如果需要增大/缩小表的大小，则必须重建表，时间开销很大。

动态哈希方法能够根据需要调整哈希表的大小，而无需重建整个表。这些方法以不同的方式执行这种调整，可以最大化读取或写入。

### Chained Hashing

最常见的方法！

链式哈希中，哈希表中数组的成员是buckets的链表，因此，当发生冲突时，将元素添加到对应Bucket的末尾即可，如果Bucket已满，则创建一个新的Bucket即可。

<img src="https://pic4.zhimg.com/80/v2-dc2ad6e9f1c6c81d07a9b32ade9b71d2.png" style="zoom:80%;" />


在JAVA的实现中，Bucket大小为1，即是一个链表，如果链表过长，直接把这个链表弄成一棵红黑树。

### Extendible Hashing

链式哈希的变种，将bucket拆分，而不是让链表永远增长，拆分的过程只会移动被拆分的bucket中的元素，而不会影响其他的元素。

这种方法允许哈希表的多个slots指向同一个bucket。

如下图所示，哈希数组中存放对应bucket的指针。

<img src="https://pic4.zhimg.com/80/v2-62cbbd9713e3273285596a3d47014599.png" style="zoom:80%;" />

对于哈希数组，有一个**全局bit位**，表示要检查前多少位作为哈希值，上图中全局bit为2，表明只看前2位就能知道自己的哈希值是多少，也能推断出此时哈希表中slot总数为$2^2=4$个。同时每一个bucket有一个**本地bit位**，表示找到本地的bucket需要多少位。

查找过程如下图，要查找A，全局bit为2，A的前两位为01，可以推断A在第2个slot内，于是在对应的bucket中查找A

<img src="https://pic4.zhimg.com/80/v2-f7007e7fb720a4c70573f86454636c31.png" style="zoom:80%;" />

插入如下图，比如要插入B，B的前两位为10，知道它在第3个slot，于是将它放在第3个slot对应的bucket中

<img src="https://pic4.zhimg.com/80/v2-5e2363c3bac9a9107da732ac19e45bfa.png" style="zoom:80%;" />

而如果插入的时候发现bucket已经满了，如下图，插入C，前两位为10，插到第3个slot对应bucket内，但是已经满了，于是将这个bucket进行拆分，拆分之后，本地bit加了1，变为了3，全局bit也改变为3。

<img src="https://pic4.zhimg.com/80/v2-beffdaf9aaaa7374f0d9845e6d3d829c.png" style="zoom:80%;" />


拆分之后再将C插入

<img src="https://pic4.zhimg.com/80/v2-cc3086da728bf728b19eafd7c7d1dd5d.png" style="zoom:67%;" />

之后按照local bit从小到大，依次遍历bucket，将哈希数据连接到bucket中

<img src="https://pic4.zhimg.com/80/v2-65ad00798b75abfbed433b22696185b9.png" style="zoom:67%;" />

<img src="https://pic4.zhimg.com/80/v2-0219cc2029a41bfe96374128b851dd1f.png" style="zoom: 67%;" />


<img src="https://pic4.zhimg.com/80/v2-bec312856117cba972f97579132d621f.png" style="zoom:80%;" />

拆分过程中，只改变了哈希数组并移动了一个bucket中的元素，修改哈希数组的代价比较小，移动元素的代价较高。

思想很巧妙，利用前缀的性质，进行bucket拆分就变得很简单了。

### Linear Hashing

线性哈希是可扩展哈希的改进，可扩展哈希有一个小的性能瓶颈，在bucket拆分且需要扩展哈希数组时，需要对整个哈希数组加锁直到bucket拆分完成。

为了解决这个问题，提出了线性哈希方式。该方法不是在桶溢出时立即拆分，而是维护一个拆分指针，指向下一个要拆分的桶。无论此指针是否指向发生溢出的存储桶，DBMS始终会拆分这个桶。线性哈希采用多个哈希函数来寻找正确的bucket。

如下图，拆分指针指向0的位置，现在哈希函数为hash1，我们想将17放入哈希表中，应该放到1的位置，但是此时bucket溢出了

<img src="https://pic4.zhimg.com/80/v2-e620e6218d7c34031a443d8d05f933ef.png" style="zoom:80%;" />


于是我们在原bucket中连接一个新的bucket，并对拆分指针指向的bucket（即0对应的bucket）进行拆分，我们添加一个4号位置，创建一个新的哈希函数hash2，为key % 2n，然后将0号位置对应的bucket按照hash2进行拆分

<img src="https://pic4.zhimg.com/80/v2-67b7481e1241a1b2e7bde8bae723cdd7.png" style="zoom:80%;" />

之后我们再进行查询的时候，先用hash1算，如果结果是0，则再用hash2算，看到底是0号还是4号

<img src="https://pic4.zhimg.com/80/v2-a7d8eb6c5dc497c2fad2da697a02d9a6.png" style="zoom:80%;" />


以此类推，当我们对3号进行拆分之后，拆分指针重新指回0，然后抛弃hash1，这样我们就完成了一轮分裂。

## Conclusion

哈希表，作为支持$O(1)$查询的数据结构，在DBMS内部被广泛使用。

哈希表通常**不能**用于存放表索引，因为哈希表不支持区间查询。

下一节课将学习B+树 (aka "The Greatest Data Structure of All Time").

