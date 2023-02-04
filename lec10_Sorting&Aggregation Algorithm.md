#! https://zhuanlan.zhihu.com/p/603310203

- [Lec10 Sorting \& Aggregation Algorithms](#lec10-sorting--aggregation-algorithms)
  - [Sorting](#sorting)
    - [External Merge Sort](#external-merge-sort)
      - [Two-way Merge Sort](#two-way-merge-sort)
      - [General (K-way) Merge Sort](#general-k-way-merge-sort)
      - [Double Buffering Optimization](#double-buffering-optimization)
      - [Using B+ Trees](#using-b-trees)
  - [Aggregations](#aggregations)
    - [Soring Aggregation](#soring-aggregation)
    - [Hasing Aggregation](#hasing-aggregation)

 
# Lec10 Sorting & Aggregation Algorithms

接下来将学习使用我们现在学习的DBMS组件来执行查询。

我们今天要讨论的算法都是基于Disk的，即查询的中间结果也需要存储到磁盘中。我们需要使用缓存池去实现这些算法，要最大化磁盘连续I/O。

![Image](https://pic4.zhimg.com/80/v2-9526f0956db226c6cc761e0384458ce5.png)

**Query Plan:**

算子组织成树形结构，数据从叶子节点流向根节点，根节点的输出就是查询的结果，我们将会在下节课讨论数据移动的粒度。

<img src="https://pic4.zhimg.com/80/v2-263ae0f74034d273b8a9f20b7fd5cd14.png" style="zoom:80%;" />


## Sorting

关系模型是 unsorted 的，即tuple不会预先排序。

在 `ORDER BY`、`DISTINCT`、`GROUP BY` 、`JOIN`算子中，都有可能需要进行排序。

如果查询包含`ORDER BY`和`LIMIT`语句，这就表明DBMS只需要扫表一次，得到top-N的元组。这样叫做**Top-N Heap Sort**。堆排序的理想场景是top-N元素能存到内存中，这样之用在内存中维护一个优先队列就行了。

当数据量过大不能存到内存中时，最标准的做法是 **external merge sort**。

### External Merge Sort

用分治思想，将数据分割成一些独立的 *runs* ，单独地对它们进行排序，接着将它们组合成一个更长的 *runs* 。可以将 *runs* 存到磁盘中，并在需要的时候读回来。（这里的一个 *run* 是指一系列key/value pairs）。

算法有两个阶段：

1. **Sorting:** 算法将能放进内存中的小块数据进行排序，并将排序好的数据写回磁盘
2. **Merge:** 算法将两个（可能是多个，两个的叫做two-way）排好序的子文件合并成一个大文件

#### Two-way Merge Sort

“2” 表示每次将2个runs合并成一个大run。

该算法在排序阶段读取每个页面，对其进行排序，并将排序后的版本写回磁盘。然后，在合并阶段，它使用三个缓冲页。它从磁盘中读取两个排序的页面，并将它们合并到第三个缓冲区页面中。每当第三页填满时，它就会被写回磁盘并替换为空页。每组排序的页面称为一个run。然后，算法递归地将runs合并在一起。

如下图，一开始一共有8个页，每个页是一个独立的run，然后第一次遍历，也就是pass0，先将每一个run都排好序；第二次遍历中，每次读取进两个相邻的长度为2的run，然后进行合并，输出长度为4的排好序的run（被放置在2个页中）；第三次遍历中，每次读取相邻两个长度为4的run，然后进行合并，输出长度为8的排好序的run（放置在4个页中）；第四次遍历中，将两个长度为8的run合并，最终生成长度为16的run（放置在8个页中），算法结束。

<img src="https://pic4.zhimg.com/80/v2-9c7d4be558f00316dc1ab673e0737c5b.png" style="zoom:80%;" />

如果N是数据页的总数，一共需要$1 + ⌈log_2N⌉$次遍历数据（里面的1，表示第一步先将所有页内的都排好序；$⌈log_2N⌉$是合并过程中迭代的次数）。所有的I/O花费是 2N ∙ (# of passes)，因为每次遍历数据都要读、写。

#### General (K-way) Merge Sort

DBMS可以使用3个以上的缓冲页。

$B$表示可用的缓冲页数量，sorting阶段，缓冲区可以一次性读进来B个页，并且将$\left \lceil \frac{N}{B} \right \rceil $个排好序的runs存进磁盘中；Merge阶段，每次可以将B-1个runs结合在一起，使用另一个缓冲页排序，写回磁盘。

算法一共需要$1+\left \lceil log_{B-1}\frac{N}{B} \right \rceil $次遍历数据（1是sorting阶段，$\left \lceil log_{B-1}\frac{N}{B} \right \rceil$ 是merge阶段），I/O花费是2N ∙ (# of passes)。

#### Double Buffering Optimization

外排序的一种优化方法是，在后台预获取下一个run，并在系统处理当前run时，将其存储在第二个缓冲区中。这样通过连续使用磁盘减少了每一步I/O请求的等待时间。如在处理page1中的run时，同时把page2中的run放进内存。

这种优化需要使用多个线程，因为预获取应该在当前运行的计算过程中进行。

![](https://pic4.zhimg.com/80/v2-bdd62764c9fa093c60e444762d0ee2cc.png)

#### Using B+ Trees

如果我们要进行排序的属性，正好有一个构建好的B+树索引，那么可以直接使用B+树排序，而不是用外排序。

如果B+树是 **聚簇B+树**，那么可以直接找到最左的叶子节点，然后遍历叶子节点，这样总比外排序好，因为没有计算消耗，所有的磁盘访问都是连续的，而且时间复杂度更低。

<img src="https://pic4.zhimg.com/80/v2-4a80b170eed4f6a3f7ebfc8aeaa4f5f1.png" style="zoom:80%;" />


如果B+数是 **非聚簇B+树**，那么遍历树总是更坏的，因为每个record可能在不同的页中，所有几乎每一个record访问都需要磁盘读取。

<img src="https://pic4.zhimg.com/80/v2-d6a6e5665ceecb4e917d06f465266843.png" style="zoom:80%;" />


## Aggregations

聚集算子就是将多个元组的单个属性的值计算成为单个标量值。

实现聚集有两种方法，(1) 排序， (2) 哈希。

### Soring Aggregation

DBMS首先对`GROUP BY`的键上的元组进行排序。如果所有内容都能放进缓冲池，可以使用内排序算法（例如，快速排序），如果数据大小超过内存，则可以使用外部归并排序算法。然后DBMS对排序后的数据执行顺序扫描以计算聚集。聚集算子的输出将按键值排序。

当执行排序聚集的时候，合理安排查询算子执行顺序对提高性能是很重要的。例如，如果查询是一个filter（即选择），那么先执行filter然后再排序，可以减小需要排序的数据量。

如下图，首先先进行filter操作，将grade是B或者C的元组筛选出来，然后进行投影操作，将无用的列去掉，然后进行排序，对排好序的结果进行聚集。

<img src="https://pic4.zhimg.com/80/v2-6f8d51bb7a6325ae9a333faa0510c826.png" style="zoom:80%;" />


### Hasing Aggregation

如果我们不需要数据最终是排好序的，如`GROUP BY`和`DISTINCT`算子，输出结果都不需要进行排序，那么Hashing是一个更好的选择，因为不需要排序，而且哈希计算更快。

DBMS在扫描表时构建临时哈希表。对于每个记录，检查哈希表中是否已经存在enrty，并执行适当的修改。如果哈希表的大小太大，无法容纳在内存中，那么DBMS必须将其存到磁盘。

外哈希聚集有两个步骤：

1. **Partition: ** 使用哈希函数 $h1$，将元组划分成磁盘上的partition，一个partition是指，有同样hash value的键值组成的一个或多个页。如果我们有B个buffer可以用，那么我们使用B-1个buffer用来做partition，剩下1个buffer用来输入数据。
2. **ReHash: ** 对于每一个在磁盘上的partition，将它的页面（可能是多个）读进内存，并且用另一个哈希函数 $h2(where \,\,h1\ne h2)$ 构建一个in-memory哈希表。之后遍历哈希表每个bucket，并将相匹配的元组计算聚集。（这里假设了一个partition可以被放进内存）。

在ReHash阶段，DBMS可能存储形式为(GroupByKey$\to$RunningValue)的pair，以计算聚集，这些pair的具体形式取决于聚集函数。当构建那个in-memory哈希表时，当在哈希表中插入一个新元组的时候，如果发现了一个匹配的GroupByKey，就更新对应的RunningValue；否则新建一个(GroupByKey$\to$RunningValue) pair。

下面展示一下过程，首先是Partition，因为我们要做`DISTINCT`，所以将这些元组按照cid作为key进行partition，我们将这些key分为了B-1个partition，然后将这些partition放入磁盘

<img src="https://pic4.zhimg.com/80/v2-02ff3f952e2dee375a710a389c4c63fa.png" style="zoom: 67%;" />


然后进行ReHash，因为在上一个阶段，一个partition内可能有碰撞，所以读入partition进行二次哈希，注意这张图显示的，第一次哈希的一个partition可能有好几页。最终形成结果。

<img src="https://pic4.zhimg.com/80/v2-c62358edbe4af6d8e65f5ac99445fe50.png" style="zoom: 67%;" />


根据聚集任务不同，RunningValue可能不一样，如果我们要进行`AVG`聚集，那么RunningValue就是(COUNT, SUM)。在ReHash阶段，每次对一个元组进行哈希，都更改哈希表中的对应键值的RunningValue值。

<img src="https://pic4.zhimg.com/80/v2-95fe7b58ed406e52e5dcb878abfc95c9.png" style="zoom:67%;" />
