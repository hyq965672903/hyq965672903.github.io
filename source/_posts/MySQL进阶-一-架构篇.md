---
title: MySQL进阶(一)-架构篇
index_img: /img/default.png
banner_img: /img/default.png
abbrlink: a7d81bc0
date: 2023-03-27 21:49:59
tags:
categories:
description:
---

 概述整个MySQL的整体架构，日志文件，存储引擎InnoDB，执行流程

<!-- more -->

### MySQL逻辑架构图

![MySQL逻辑架构图](E:\git_repo\hyq965672903.github.io\source\_posts\MySQL进阶-一-架构篇.assets\MySQL逻辑架构图.png)

### MySQL日志文件

MySQL是通过文件系统对数据索引后进行存储的，MySQL从物理结构上可以分为**日志文件**和**数据及索引**

**文件**。MySQL在Linux中的数据索引文件和日志文件通常放在/var/lib/mysql目录下。MySQL通过日志记

录了数据库操作信息和错误信息。

```sql
show variables like 'log_%';
```

#### 常用日志文件

- 错误日志：/var/log/mysql-error.log

- 二进制日志：/var/lib/mysql/mysql-bin

- 查询日志：general_query.log

- 慢查询日志：slow_query_log.log

- 事务重做日志：redo log

- 中继日志：relay log

#### 错误日志

> 默认开启，错误日志记录了运行过程中遇到的所有严重的错误信息，以及 MySQL每次启动和关闭的详细信息。错误日志所记录的信息是可以通过log_error和log_warnings配置来定义的。从5.5.7以后不能关闭错误日志

log_error：指定错误日志存储位置

log-warnings：是否将警告信息输出到错误日志中。

- log_warnings 为0， 表示不记录告警信息。

- log_warnings 为1， 表示告警信息写入错误日志。

- log_warnings 大于1， 表示各类告警信息，例如：有关网络故障的信息和重新连接信息写入

错误日志。

```sql
log_error=/var/log/mysql-error.log
log_warnings=2
```

#### 二进制日志

> 默认关闭，需要通过配置进行开启。binlog记录了数据库所有的ddl语句和dml语句，但不包括select语句内容，语句以事件的形式保存，描述了数据的变更顺序，binlog还包括了每个更新语句的执行时间信息。如果是DDL语句，则直接记录到binlog日志，而DML语句，必须通过事务提交才能记录到binlog日志中

binlog主要用于实现mysql**主从复制、数据备份、数据恢复**

不同版本开启方式有些许`差异`

#### 通用查询日志

> **默认关闭，**由于通用查询日志会记录用户的所有操作，其中还包含增删查改等信息，在并发操作大的环境下会产生大量的信息从而导致不必要的磁盘IO，会影响MySQL的性能的。

```sql
show global variables like '%general_log%';
```

一般不建议开启，调试时候使用

#### 慢查询日志

> **默认关闭**，通过以下设置开启。记录执行时间超过**long_query_time**秒的所有查询，便于收集查询时间比较长的SQL语句。

```sql
show global status like '%Slow_queries%';
show variables like '%slow_query%';
show variables like 'long_query_time%';
```

配置慢查询开启

```sql
# 开启慢查询日志
slow_query_log=ON
# 慢查询的阈值，单位秒
long_query_time=10
# 日志记录文件
# 如果没有给出file_name值， 默认为主机名，后缀为-slow.log。
# 如果给出了文件名，但不是绝对路径名，文件则写入数据目录。
slow_query_log_file=slow_query_log.log
```

### MySQL数据文件

查看MySQL数据文件

```sql
show variables like '%datadir%';
```

- `ibdata`文件：使用系统表空间存储表数据和索引信息，所有表共同使用一个或者多个ibdata文件
- `InnoDB`存储引擎的数据文件
  - .**frm**文件：主要存放与表相关的数据信息，主要包括表结构的定义信息
  - .**ibd**：**使用独享表空间存储表**数据和索引信息，一张表对应一个ibd文件。
- `MyISAM`存储引擎的数据文件
  - **.frm**文件：主要存放与表相关的数据信息，主要包括表结构的定义信息
  - **.myd**文件：主要用来存储表数据信息。
  - **.myi**文件：主要用来存储表数据文件中任何索引的数据树。

### MySQL存储引擎之InnoDB

> 除了特殊场景，一般都选择InnoDB，支持事务，分布式事务

#### 存储引擎种类

| **存储引擎** | 说明                                                         |
| ------------ | ------------------------------------------------------------ |
| `MyISAM`     | 高速引擎，拥有较高的插入，查询速度，**但不支持事务**         |
| `InnoDB`     | **5.5****版本后****MySQL****的默认数据库存储引擎，支持事务和行级锁**，比MyISAM处理速度稍慢 |
| ISAM         | MyISAM的前身，MySQL5.0以后不再默认安装                       |
| MRG_MyISAM   | 将多个表联合成一个表使用，在超大规模数据存储时很有用         |
| `Memory`     | **内存存储引擎，拥有极高的插入，更新和查询效率。**但是会占用和数据量成正比的内存空间。只在内存上保存数据，意味着数据可能会丢失 |
| Archive      | 将数据压缩后进行存储，非常适合存储大量的独立的，作为历史记录的数据， |
| CSV          | CSV 存储引擎是基于 CSV 格式文件存储数据(应用于跨平台的数据交换) |

#### `InnoDB`和`MyISAM`存储引擎区别

| **比较项**   | InnoDB                                 | MyISAM                                   |
| ------------ | -------------------------------------- | ---------------------------------------- |
| **存储文件** | .frm 表定义文件.ibd 数据文件和索引文件 | frm 表定义文件.myd 数据文件.myi 索引文件 |
| **锁**       | 表锁、行锁                             | 表锁                                     |
| **事务**     | 支持                                   | 不支持                                   |
| **CRUD**     | 读、写                                 | 读多                                     |
| **索引结构** | B+ Tree                                | B+ Tree                                  |

