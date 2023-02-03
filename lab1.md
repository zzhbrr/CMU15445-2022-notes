# Lab1 Buffer Pool Manager

project1 是让实现一个 buffer pool manager，分为三个部分

* Extendible Hash Table （可扩展哈希表）
* LRU-K Replacer （LRU-K页替换算法）
* Buffer Pool Manager Instance （缓存池管理器）

**可扩展哈希表**和**LRU-K替换算法**是**缓存池管理器**的内部组件。

下面介绍我对这三部分的实现。

## 1 Extendible Hash Table

我在`ExtendibleHashTable`类中新开了一个`std::vector<std::shared_ptr<Bucket>> buckets_`存放哈希表中bucket的实例，方便管理。

在`ExtendibleHashTable::Bucket`类中新建了一个`size_t identifier_`，表示这个bucket识别的后缀是什么。

## Buffer pool manager instance

* UnpinPgImp中，如果传进来is_dirty是false，则pages不能赋成false，而是不变。