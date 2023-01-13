#! https://zhuanlan.zhihu.com/p/598888037
- [Lec02 Advanced SQL](#lec02-advanced-sql)
  - [SQL History](#sql-history)
  - [Aggregates](#aggregates)
  - [String Operation](#string-operation)
  - [Data and Time](#data-and-time)
  - [Output Redirection](#output-redirection)
  - [Output Control](#output-control)
  - [Nested Queries](#nested-queries)
  - [Window Functions](#window-functions)
  - [Common Table Expression](#common-table-expression)


# Lec02 Advanced SQL
这部分比较熟悉，粗略地记录一下我没见过的SQL语句
## SQL History

SQL : "SEQUEL" (Structured English Query Language).是一种对于关系数据库使用的声明式查询语言。

分类:

* Data Manipulation Language(DML) ：如SELECT, INSERT, UPDATE, DELETE 
* Data Definition Language(DDL)：如Schema definitions for tables, indexes, views, and other objects.
* Data Control Language(DCL)：如权限控制
* 也包括定义视图、完整性约束、事物等

SQL是基于**bags**的（duplicates），而不是基于sets（no duplicates）的.

最好一句SQL解决查询问题，这样数据库可以尽可能进行优化。

## Aggregates

## String Operation

大小写敏感，用单引号

## Data and Time

## Output Redirection

对查询结果重定向，输出到其他表中。

* 输出到一个新的表里

  ```sql
  SELECT DISTINCT cid INTO CourseIds FROM enrolled;
  ```

* 输出到已存在的表里

  ```sql
  INSERT INTO CourseIds (SELECT DISTINCT cid FROM enrolled);
  ```

## Output Control

输出排序、限制输出个数。

## Nested Queries

## Window Functions

```sql
SELECT ... FUNC-NAME(...) OVER(...)
FROM tablename;	
```

<img src="https://pic4.zhimg.com/80/v2-cc36f59a859f63e554894e0543e083da.png" style="zoom:80%;" />


## Common Table Expression

在sql语句中定义临时视图

<img src="https://pic4.zhimg.com/80/v2-cab6045090121f1ee35e5b7aa4b42caf.png" style="zoom:80%;" />