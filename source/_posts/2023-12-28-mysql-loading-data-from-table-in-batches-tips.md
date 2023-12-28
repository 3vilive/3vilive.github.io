---
title: MySQL 从表中分批读取数据技巧
tags:
  - MySQL
description: ''
index_img: img/mysql.png
categories:
  - Database
date: 2023-12-28 15:51:39
---


# 前言

有时候我们的程序需要读取大量的表数据，最简单的方法就是直接全部读出：

```sql
select * from user where category = 1
```

表数据量不大的时候，这样问题不大。

但当表数据的时候大的时候，会导致 MySQL 短时间压力过大，而且将如此多的数据读到内存中，程序占用的内存也会比较大。

利索当然的，我们会想要分批处理数据。


# 通过分页（order, offset, limit）分批次读取

假设我们每次读取 1000 条数据，可以这样读取：

第一次读取：

```sql
select * from user where category = 1 order by id limit 0, 1000
```

第二次读取：

```sql
select * from user where category = 1 order by id limit 1000, 1000
```

以此类推，直到读取不到数据后，即可完成读取。

这样的方案也很简单，但是存在一个问题就是：`offset` 越大查询的速度越慢。

这是 MySQL 在执行 `limit` 子句的时候，会执行整个查询，然后再跳过指定的数量的行数，再返回结果。跳过的行数越大，查询越慢。

用黑色表示 MySQL 会读取的数据，通过分页分批次查询可以这样表示：

```
第一次查询
|█░░░░░░░░░|

第二次查询
|██░░░░░░░░|

最后一次查询
|██████████|
```

可以看到，实际上越深的分页，会将之前读取的数据再读取一遍。这是导致越到后面，查询越慢的原因，越到后面查询的消耗就越大。

一个对比的例子：

```
mysql> select * from user where bind_status > 0 order by id limit 10, 10;
...
10 rows in set (0.00 sec)

mysql> select * from user where bind_status > 0 order by id limit 2007967, 10;
...
10 rows in set (0.90 sec)
```


# 基于索引分批次读取

我们想要避免上面的情况，尽量让每次查询都能够只扫描一部分数据，减少消耗。

这时候，我们可以利用索引，跳过已经扫描过的数据。

假设每次读取 1000 行，那么现在的读取语句就变成了这样：

第一次读取：

```sql
select * from user where id > 0 category = 1 order by id limit 1000
```

假设第一次读取返回的最大 id 为 1500，那么第二次读取的语句则是：

```sql
select * from user where id > 1500 category = 1 order by id limit 1000
```

以此类推，直到读取不到数据后，即可完成读取。

用黑色表示 MySQL 会读取的数据，通过索引分批次查询可以这样表示：

```
第一次查询
|█░░░░░░░░░|

第二次查询
|░█░░░░░░░░|

最后一次查询
|░░░░░░░░░█|
```

通过索引，我们跳过已经扫描的部分，从上次扫描的末尾开始扫描。

