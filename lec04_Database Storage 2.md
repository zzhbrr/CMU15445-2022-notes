#! https://zhuanlan.zhihu.com/p/598891564
- [Lec04 Database Storage 2](#lec04-database-storage-2)
  - [Log-Structured Storage](#log-structured-storage)
  - [Data Representation](#data-representation)
  - [System Catalogs](#system-catalogs)

# Lec04 Database Storage 2

这节课程的内容有：

* Log-Structured Storage
* Data Representation
* System Catalogs



## Log-Structured Storage

上节课讨论的tuple-oriented中的slotted page design可能会有什么问题呢：

1. 碎片，删除tuples会在page中留下碎片
2. 无用的磁盘I/O。磁盘读取是一块一块地读。
3. 随机磁盘I/O，比如更新20个在不同page的tuple

如果DBMS不会覆盖页中的数据，只会创建新的页，会解决上面的一些问题。

Log-Structured存储模型就是用上面这种思想。

**Log-Structured Storage：** DBMS不存储tuples，而是存储log记录

* 存储文件如何修改的记录。每个log record都包含tuple的id。添加log就往末尾添加。

  <img src="https://pic4.zhimg.com/80/v2-9fdf4e97906d0bdb3e5a1e7a8502e557.png" style="zoom:67%;" />


* 为了读取一个record，DBMS扫描log file，重建tuple。DBMS可以维护索引，从tuple_id映射到最新的log record

  <img src="https://pic4.zhimg.com/80/v2-b5508f3801f32a809825b3d70508828a.png" style="zoom:60%;" />



* 写很快，但是读很慢。磁盘写是连续的，而且现有页是不可变的，减少随机磁盘I/O

* 对于append-only存储友好，因为DBMS不可以回去更新数据

* DBMS可以周期性的压缩log

  <img src="https://pic4.zhimg.com/80/v2-c1be070d3a5fa3df20ecc26fbb1bf4b7.png" style="zoom:67%;" />


* 压缩之后，时序信息已经没用了，因为对于一个tuple，一个page中最多出现一次。DBMS可以按照id排序，存入一个表中（SSTable），提高性能，更快进行查找

  <img src="https://pic4.zhimg.com/80/v2-0917b1525946c8db26bfd86018b84c70.png" style="zoom:90%;" />


* 压缩策略有universal compaction和level compaction，如RocksDB就是用的level compaction

  ![](https://pic4.zhimg.com/80/v2-2ca36bb3e35ab2d29e5f97196b35cab3.png)


  
Log-Structured存储引擎在如今很常用，可以归功于RocksDB快速发展。而且也会使分布式数据库更容易实现。

* Log-Structured存储的缺点：

  * Write-Amplification 

    如果对于一个tuple永远不改它，那么每次compact都要重新写一遍这个tuple。

  * Compaction is Expensive

## Data Representation

tuple中的数据本质上就是字节数组，DMBS来负责解释这些字节，得出属性的**值**。

Data representation schema 是DBMS是存储一个**值**的字节的方法。

有五种high level的数据类型可以存储到tuple中: integers, variable-precision numbers, fixed-point precision numbers, variable length values, and dates/times.

**Integers**

Examples: INTEGER, BIGINT, SMALLINT, TINYINT

大多数DBMS用C/C++中的类型来存储Integers。长度是固定的。

**Variable Precision Numbers**

Examples: FLOAT, REAL

用C/C++中的浮点类型来存储，但是浮点数会产生误差，这个对于数据库来说很致命。长度是固定的。

**Fixed-Point Precision Numbers**

Examples: NUMERIC, DECIMAL

numeric的数据类型，存储精确，通常用变长数组存储，像是一个string，要存储一些元信息，来说明自己的长度、scale等。

为了保持精度会有很大的性能损失。

如Postgresql的NUMERIC数据类型定义如下：

<img src="https://pic4.zhimg.com/80/v2-0a737abc5786e0ad0b1e0394fd64f0ba.png" style="zoom:80%;" />

**Variable-Length Data**

Examples: VARCHAR, VARBINARY, TEXT, BLOB.

通常跟一个header一块存储，指示这个变长数据有多长，以便跳到下一个值，header中还可能有checksum。

<img src="https://pic4.zhimg.com/80/v2-5998042ef003043bff6c3457fe13b0a9.png" style="zoom:67%;" />

大多数DBMS不允许一个tuple超过page size大小，如果超过了，会把超过的那个值存储到一个单独的page中，然后用一个指针指向这个新page。如果一个page盛不下，也可以连接更多的page。

<img src="https://pic4.zhimg.com/80/v2-51d1ebba5f94b9acd73912185d1315fa.png" style="zoom:67%;" />


**Dates and Times**

Examples: TIME, DATE, TIMESTAMP

通常跟C中的time(0)一样，记录unix时代到现在的单位时间。

## System Catalogs

DBMA将数据库的元信息存储到内部目录中

* Tables, columns, indexes, views
* Users, permissions
* Internal, statistics

几乎所有的DBMA将数据库的目录当做表存储到自身中。

可以通过DBMS的 INFORMATION_SHCEMA 目录获取关于数据库的信息。