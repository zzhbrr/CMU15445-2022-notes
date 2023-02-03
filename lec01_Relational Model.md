#! https://zhuanlan.zhihu.com/p/598883504
- [Lec01 Relational Model](#lec01-relational-model)
  - [Database](#database)
  - [Flat File](#flat-file)
  - [Database Management System](#database-management-system)
    - [Data Model](#data-model)
    - [Schema](#schema)
  - [Relational Model](#relational-model)
  - [Data Manipulation Languages（DMLS）](#data-manipulation-languagesdmls)
  - [Relational Algebra](#relational-algebra)


# Lec01 Relational Model

## Database

A  *database*  is an **organized collection of inter-related data** that models some aspect of the real-world.

A database management system (DBMS) is the **software** that manages a database.

## Flat File

假设我们要维护一个乐手和专辑的数据库，单独用类似CSV结构也可以作为一个数据库，我们用两张表，一张是乐手的表、一张是专辑的表，表中每个实体的属性都用逗号分开，每次查找都遍历一遍，但是会有一些问题

* Data Integrity
  * 如果有人将专辑的时间改成了一个非法的字符串怎么办
  * 如何处理一个专辑有很多乐手
  * 如果我们将一个乐手删掉，他对应的专辑会怎么样
* Implementation
  * 如何找到特定的记录
  * 两个线程同时写一个表怎么办
* Durability
  * 如果在一个程序修改记录的时候机器断电了怎么办
  * 我们如何在多机器上复制这个数据库应对更高的访问

这就需要DBMS了

## Database Management System

A *DBMS* is a software that allows applications to store and analyze information in a database.

A general-purpose DBMS is designed to allow the **definition, creation, querying, update, and administration** of databases in accordance with some **data model**.

### Data Model

**Data Model**: a collection of concepts for describing the data in database.

例如：Relational(mysql,oracle)，Key/value(redis)，Graph(Neo4J)，Document/Object(MongoDB)等

### Schema

**Schema**: a description of a particular collection of data based on a data model.



## Relational Model

早期的DBMS逻辑层和物理层耦合严重，数据库应用很难构建。Ted Codd在1970年提出了关系模型避免这个问题。

关系模型基于关系定义了数据库抽象，有三个关键点：

* 用简单的数据结构存储数据库（关系）
* 用high-level的语言获取数据，DBMS寻找最优的执行策略
* 物理存储留给DBMS实现

关系模型定义了三个概念：

* Structures: 关系和关系的内容的定义。即关系拥有的属性，以及属性可能的取值。
* Integrity: 保证数据库内容符合约束。
* Manipulation：如何获取和修改数据库内容。

**关系**是一个**无序**的，包含表示实体的属性的关系。因为关系是无序的，所以DBMS可以以任何方式储存他们。

每个属性都可以有特殊的值NULL，表示未定义。



## Data Manipulation Languages（DMLS）

是从数据库中存储和检索信息的方法。有两种这样的语言：

* Procedural：对DBMS的query指定了high-level策略，DBMS需要遵守这个策略。（关系代数）
* Non-Procedural(Declarative)：对DBMS的query只说明了想要什么样的数据，而DBMS自己寻找合适的策略。（关系演算，如SQL）。



## Relational Algebra

**Select**

从关系中选择符合条件的元组。

<img src="https://pic4.zhimg.com/80/v2-92191e7c9b66fe127e6c80243bdb8c91.png" style="zoom:80%;" />

**Projection**

输入关系，输出的关系每个元组只包含指定的属性

<img src="https://pic4.zhimg.com/80/v2-42800e03d62f8d876fadc67838ac59cd.png" style="zoom:80%;" />

**Union**

输入两个关系，输出两个关系的并集（每个元组在两个关系中至少出现一次）

<img src="https://pic4.zhimg.com/80/v2-acd8efc332107e746e6146355fdb15de.png" style="zoom:80%;" />


**Intersection**

输入两个关系，输出两个关系的交集（每个元组在两个关系中都要出现）

<img src="https://pic4.zhimg.com/80/v2-14c4de80111c7e2187b558d107078aca.png" style="zoom:80%;" />

**Difference**

输入两个关系A、B，输出两个关系的差集（元组在A中出现而不在B中出现）

<img src="https://pic4.zhimg.com/80/v2-aeef7acaef717ad7da74fbdbc4e8ed98.png" style="zoom:80%;" />

**Product**

两个元组的笛卡尔积

<img src="https://pic4.zhimg.com/80/v2-3fd7a7160609108e1c2795995731aa37.png" style="zoom:80%;" />
**Join**

两个元组的连接。

<img src="https://pic4.zhimg.com/80/v2-53effcc67871ec0510f8acbad94c54d5.png" style="zoom:80%;" />