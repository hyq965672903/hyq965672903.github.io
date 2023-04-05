---
title: MySQL进阶(五)-性能优化篇
index_img: /img/default.png
banner_img: /img/default.png
date: 2023-04-04 23:52:08
tags:
categories:
description:
---

 MySQL数据库调优

<!-- more -->

## 调优范围

- **调SQL语句**：根据需求创建结构良好的SQL语句

- **调索引**：索引创建原则

- **调数据库表结构**

- **调MySQL配置**：最大连接数，连接超时，线程缓存，查询缓存，排序缓存，连接查询缓存...

- **调MySQL宿主机OS**：TCP连接数，打开文件数，线程栈大小

- **调服务器硬件**：更多核CPU、更大内存

- **MySQL客户端**：连接池（MaxActive，MaxWait），连接属性

## 执行计划Explain

![执行计划Explain](http://file.hyqup.cn/img/%E6%89%A7%E8%A1%8C%E8%AE%A1%E5%88%92Explain.png)

```sql
EXPLAIN select * from user
```

### 参数介绍

- id：SELECT识别符，这是SELECT查询序列号。

- **select_type**：表示单位查询的查询类型，比如：普通查询、联合查询(union、union all)、查询等复杂查询。

- table：表示查询的表

- partitions：使用的哪些分区（对于非分区表值为null）。

- **type**（重要）表示表的连接类型。

- possible_keys：此次查询中可能选用的索引

- key：查询真正使用到的索引

- key_len：显示MySQL决定使用的索引size

- ref：哪个字段或常数与 key 一起被使用

- **rows**：显示此查询一共扫描了多少行，这个是一个估计值，不是精确的值。

- filtered: 表示此查询条件所过滤的数据的百分比

- **Extra**：额外信息

#### select_type参数

> 总体分为三大类，简单查询，连接查询，子查询

- **simple** 简单查询
- **primary** 主表，多张表的时候有一张主表

- **union**：连接查询

- **dependent union** 依赖连接查询
- **subquery**：子查询
- **dependent subquery**：依赖子查询
- **derived**：派生表

#### type参数

> 显示的是单位查询的**连接类型**或者理解为**访问类型**

```sql
system
const
eq_ref
ref
fulltext
ref_or_null
unique_subquery
index_subquery
range
index_merge
index
ALL
```

##### system

表中**只有一行数据或者是空表**

##### **`const`**

使用**唯一索引或者主键**

**`eq_ref`**

**唯一性索引扫描**

##### `ref`

**非唯一性索**引扫描

##### **fulltext**

全文索引检索

##### **ref_or_null**

与ref方法类似，只是增加了null值的比较

##### **unique_subquery**

用于where中的in形式子查询，子查询返回不重复值唯一值

##### **index_subquery**

用于in形式子查询使用到了辅助索引或者in常数列表，子查询可能返回重复值，可以使用索引将子查询去重

##### **`range`**

**索引范围扫描**，常见于使用>,<,is null,between ,in ,like等运算符的查询中

#####  **index_merge**

表示查询使用了两个以上的索引，最后取交集或者并集，常见and ，or的条件使用了不同的索引

##### **`index`**

select结果列中使用到了索引，type会显示为index。**全部索引扫描**，把索引从头到尾扫一遍，常见于使用索引列就可以处理不需要读取数据文件的查询、可以使用索引排序或者分组的查询

##### **`all`**

这个就是全表扫描数据文件，然后再**在****server****层进行过滤**返回符合要求的记录

#### **Extra参数**

- **Using filesort**

  使用了文件排序

- **Using index**

  表示相应的SELECT查询中使用到了索引，避免访问表的数据行，这种查询的效率很高！

- **Using where**

  MySQL将对InnoDB提取的结果在SQL Layer层进行过滤

- **Using join buffer**

  使用了连接缓存

## 优化

创建索引的原则，判断组合索引失效场景

### 慢查询

**查看是否开启慢查询功能**

```sql
# 查看是否开启慢查询日志
show variables like '%slow_query%';
show variables like 'long_query_time%';
```

slow_query_log：是否开启慢查询日志，1为开启，0为关闭

log-slow-queries：旧版（5.6以下）MySQL数据库慢查询日志存储路径。

slow-query-log-file：新版（5.6及以上）MySQL数据库慢查询日志存储路径。

​		不设置该参数，系统则会默认给一个文件host_name-slow.log

long_query_time：慢查询阈值，当查询时间多于设定的阈值时，记录日志，单位秒

### 开启慢查询

```sql
# 开启慢查询日志
set global slow_query_log=on;
# 大于1秒钟的数据记录到慢日志中，如果设置为默认0，则会有大量的信息存储在磁盘中，磁盘很容易满
掉
# 如果设置不生效，建议配置在my.cnf配置文件中
set global long_query_time=1;
# 记录没有索引的查询。
set global log_queries_not_using_indexes=on;
```

### **连接数**max_connections

```sql
# 查看 max_connections
show global variables like 'max_connections'
# 设置 max_connections（立即生效重启后失效）
set global max_connections = 800;
# 这台MySQL服务器最大连接数是256，然后查询一下服务器使用过的最大连接数：
show global status like 'Max_used_connections';
# MySQL服务器过去的最大连接数是245，没有达到服务器连接数上限256，应该没有出现1040错误，
比较理想的设置是：Max_used_connections / max_connections * 100% ≈ 85%
最大连接数占上限连接数的85%左右，如果发现比例在10%以下，MySQL服务器连接数上限设置的过高
了
```

