---
title: 通过 straight_join 指定 MySQL 连表查询的执行顺序改善查询性能
tags:
  - MySQL
description: 当执行连表查询的时候，优化器会选择他认为最优的执行顺序，这可能导致一些问题。
index_img: img/mysql.png
categories:
  - Database
date: 2023-12-27 15:14:25
---


# 前言

当执行连表查询的时候，优化器会选择他认为最优的执行顺序，这可能导致一些问题。

假设存在三个表，分别有不同的行数：

| 表名 | 行数 |
| :-: | :-: |
| `tbl_a` | 60000（6W） |
| `tbl_b` | 200000（20W） |
| `tbl_c` | 5000000（500W）|


```sql
select
    count(*)
from tbl_a a
join tbl_b b on a.user_id = b.user_id
join tbl_c c on a.user_id = c.user_id
where
    a.package_id = 10
    and c.create_time between 1604419200 and 1703692800
```

比较符合直觉的情况就是 `MySQL` 使用 `tbl_a` 作为驱动表，与 `tbl_b` 连接后，再与 `tbl_c` 连接。

但是实际情况可能不是这个顺序，比如优化器发现 `tbl_c.create_time` 是存在索引的，判断后按照如下顺序链接：

- `tbl_c`
- `tbl_a`
- `tbl_b`


```sql
+----+-------------+-------+------------+--------+--------------------------------------------------------------------+------------------------------------+---------+----------------------------------+---------+----------+--------------------------+
| id | select_type | table | partitions | type   | possible_keys                                                      | key                                | key_len | ref                              | rows    | filtered | Extra                    |
+----+-------------+-------+------------+--------+--------------------------------------------------------------------+------------------------------------+---------+----------------------------------+---------+----------+--------------------------+
|  1 | SIMPLE      | c     | NULL       | range  | PRIMARY,user_id_idx,create_time_index | create_time_index | 8       | NULL                             | 2075427 |   100.00 | Using where; Using index |
|  1 | SIMPLE      | a     | NULL       | eq_ref | uni_owner_user_id,user_id                                                | uni_owner_user_id                     | 1026    | const,user_id |       1 |   100.00 | Using where; Using index |
|  1 | SIMPLE      | b     | NULL       | eq_ref | user_id                                                               | user_id                               | 1023    | user_id       |       1 |   100.00 | Using where; Using index |
+----+-------------+-------+------------+--------+--------------------------------------------------------------------+------------------------------------+---------+----------------------------------+---------+----------+--------------------------+****
```

虽然使用了 `tbl_c.create_time` 索引，但是导致了大表驱动小表，查询速度反而不如预期：

```
+----------+
| count(*) |
+----------+
|        7 |
+----------+
1 row in set (11.54 sec)
```


通常来说，优化器的选择是最好的，但是遇到这种情况，我们就需要指定连接的顺序了。


# 使用 straight_join 指定顺序

为了达到指定连接顺序的目的，我们可以使用 `straight_join` 。

在 MySQL 文档中这样描述 `straight_join`：

> **STRAIGHT_JOIN** is similar to JOIN, except that the left table is always read before the right table. This can be used for those (few) cases for which the join optimizer processes the tables in a suboptimal order.

翻译过来就是：

**STRAIGHT_JOIN** 与 JOIN 类似，除了左边的表总是在读取右边的表前读取。`straight_join` 用于 join 优化器未使用最优顺序来处理多个表的情况。

使用 `straight_join` 来修改上面的查询：

```sql
select
    count(*)
from tbl_a a
straight_join tbl_b b on a.user_id = b.user_id
straight_join tbl_c c on a.user_id = c.user_id
where
    a.package_id = 10
    and c.create_time between 1604419200 and 1703692800
```

再执行一次，发现效果好多了：

```sql
+----------+
| count(*) |
+----------+
|        7 |
+----------+
1 row in set (1.57 sec)
```

# 注意事项

- 通常不需要使用 straight_join 来指定顺序，除非你知道自己在干什么
- 随着时间推移，你的表数据分布可能发生改变，你可能需要重新判断使用 straight_join 的查询是否还是最优的

# 参考文档

- [MySQL 8.0 Reference Manual - JOIN Clause](https://dev.mysql.com/doc/refman/8.0/en/join.html)