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

#### InnoDB架构图

![InnoDB架构图](http://file.hyqup.cn/img/InnoDB%E6%9E%B6%E6%9E%84%E5%9B%BE.png)

##### 内存结构

1. Buffer Pool 缓冲池

2. Change Buffer 修改缓冲

3. Adaptive Hash Index 自适应索引

4. Log Buffer 日志缓冲

###### Buffer Pool 缓冲池

> 缓冲池Buffer Pool 用于加速数据的访问和修改，通过将热点数据缓存在内存的方法，最大限度地减少磁盘 IO，加速热点数据读写

###### Change Buffer 修改缓冲

> Change Buffer（在 MySQL 5.6 之前叫 insert buffer，简称 ibuf ）是 InnoDB 5.5 引入的一种优化策略。**Change Buffer** **用于加速非热点数据中二级索引的写入操作。**由于二级索引数据的不连续性，导致修改二级索引时需要进行频繁的磁盘 IO 消耗大量性能，Change Buffer 缓冲对二级索引的修改操作，同时将写操作录入 redo log 中，在缓冲到一定量或系统较空闲时进行 merge 操作将修改写入磁盘中。

###### Adaptive Hash Index 自适应索引

> **自适应哈希索引**（Adaptive Hash Index，**AHI**）**用于实现对于热数据页的一次查询。**是建立在索引之上的索引！使用聚簇索引进行**数据页**定位的时候需要根据索引树的高度从根节点走到叶子节点，**通常需要3 到 4 **次查询才能定位到数据**。InnoDB 根据对索引使用情况的分析和索引字段的分析，通过自调优Self-tuning的方式为索引页建立或者删除哈希索引。

###### Log Buffer 日志缓冲

> **InnoDB** **使用** **Log Buffer** **来缓冲日志文件的写入操作。**内存写入加上日志文件顺序写的特点，使得InnoDB 日志写入性能极高。

##### 磁盘结构

**表空间**

在磁盘中，InnoDB 将所有数据都逻辑地存放在一个空间中，称为表空间（Tablespace）。表空间由段（Segment）、区（extent）、页（Page）组成

- 开启独立表空间`innodb_file_per_table=1`每张表的数据都会存储到一个独立表空间，即 表名.ibd 文件

- 关闭独占表空间`innodb_file_per_table=0`，则所有基于InnoDB存储引擎的表数据都会记录到系统表空间，即 ibdata1 文件

表空间是 InnoDB 物理存储中的最高层，目前的表空间类别包括：

- 系统表空间（System Tablespace）

- 独立表空间（File-per-table Tablespace）

- 通用表空间（General Tablespace）

- 回滚表空间（Undo Tablespace）

- 临时表空间（The Temporary Tablespace）

###### 系统表空间

> **系统表空间是** **InnoDB** **数据字典、双写缓冲、修改缓冲和回滚日志的存储位置**，如果关闭独立表空间，它将存储所有表数据和索引。

###### 独立表空间

> **独立表空间用于存放每个表的数据和索引。**其他类型的信息，如：回滚日志、双写缓冲区、系统事务信息、修改缓冲等仍存放于系统表空间内。因此即使用了独立表空间，系统表空间也会不断增长。在5.7版本中默认开启

###### 通用表空间

> 通用表空间（General Tablespace）是一个由 CREATE TABLESPACE 命令创建的共享表空间，创建时必须指定该表空间名称和 ibd 文件位置，ibd 文件可以放置于任何 MySQL 有权限的地方。该表空间内可以容纳多张数据表，同时在创建时可以指定该表空间所使用的默认引擎。

**通用表空间存在的目的是为了在系统表空间与独立表空间之间作出平衡**

###### 回滚表空间

> Undo TableSpace 用于存放一个或多个 undo log 文件。**默认** **undo log** **存储在系统表空间中，**MySql5.7中支持自定义Undo log表空间并存储所有 **undo log**。一旦用户定义了 Undo Tablespace，则系统表空间中的 Undo log 区域将失效。对于 Undo Tablespace 的启用必须在 MySQL 初始化前设置，Undo Tablespace 默认大小为 10MB。Undo Tablespace 中的 Undo log 表可以进行 truncate 操作。

##### 存储架构

###### 段【Segment】

- 表空间由各个段组成，段类型分`数据段`（叶子节点）、`索引段`（非叶子节点）、`回滚段`
- MySQL的索引数据结构是B+树，这个树有叶子节点和非叶子节点
- 一个段包含多个区，至少有一个区，段扩展的最小单位是区

###### 区【Extent】

- 区是由连续的页组成的空间，大小固定为 `1MB`
- 默认情况下，一个区里有64个页
- 为了保证区的连续性，InnoDB一次会从磁盘申请4-5个区

###### 页【Page】

- **页是 InnoDB 的基本存储单位**，页默认大小是16K（可配置innodb_page_size），InnoDB 首次加载后便无法更改
- 操作系统读写磁盘最小单位是页，4K
- 磁盘存储数据量最小单位512 byte

###### 行（Row）

- InnoDB的数据是以行为单位存储，一个页中包含多个行
- InnoDB提供4种行格式：Compact、Redundant、Dynamic和Compressed
- 默认行格式Dynamic

##### 内存中的数据如何进入磁盘？

###### 脏页落盘

**什么是脏页？**

> 对于数据库中页的**修改操作**，则首先修改在缓冲区中的页，缓冲区中的页与磁盘中的页数据不一致，所以称缓冲区中的页为**`脏页`**

**脏页如何进入到磁盘？**

- 脏页从缓冲区刷新到磁盘，不是每次页更新之后触发，而是通过**`CheckPoint机制`**刷新磁盘

###### 数据安全性保证(重点)

- **Write Ahead Log（WAL）**：**要求数据的变更写入到磁盘前，首先必须将内存中的日志**

  **写入到磁盘**，InnoDB就是`redo log`

- **Force Log at Commit**：**当事务提交时，所有事务产生的日志都必须刷到磁盘**

######  **怎么确保日志就能安全的写入系统**

为了确保每次日志都写入到redo日志文件，在每次将redo日志缓冲写入redo日志后，调用一次

**`fsync`**操作，将缓冲文件从文件系统缓存中真正写入磁盘

与数据直接写入磁盘不同的是

- redo日志不会记录完整的一页数据
- 日志是顺序写入，而数据是随机写入。顺序写入效率更高
- 可以通过 innodb_flush_log_at_trx_commit 来控制redo日志刷新到磁盘的策略
  - **为0时：**每秒写入，与事务无关
    - 最多丢失1秒的事务操作
    - 写入效率最高，安全性最低
  - **为1时：**事务提交，写入磁盘
    - 不会丢失数据
    - 写入效率最低，安全性最高
  - **为2时：**事务提交，写入OS Buffer
    - 数据安全性依赖于系统，最多丢1秒事务操作
    - 写入效率居中，安全性居中

###### **CheckPoint机制**

> 它是将缓冲池中的脏页数据刷到磁盘上的机制，决定脏页落盘的时机、条件和脏页的选择等

###### 分类

**sharp checkpoint：**关闭数据库时将脏页全部刷新到磁盘中

**fuzzy checkpoint：默认方式，**在运行时选择不同时机将脏页刷盘，只刷新部分脏页

- **Master Thread Checkpoint：**固定频率刷新部分脏页到磁盘，异步操作不会阻塞用户线程
- **FLUSH_LRU_LIST Checkpoint：**缓冲池淘汰非热点Page，如果该Page是脏页会执行CheckPoint
- **Async/Sync Flush Checkpoint：**redo日志不可用时，强制脏页落盘，有了前两个这种一般不会发生
- **Dirty Page too much Checkpoint：**脏页占比太多强制进行刷盘，阈值75%

###### **Double Write机制**

> 数据库准备刷新脏页时，将16KB的刷入磁盘，但当写入了8KB时，就宕机了这种只写了部分没完成的情况被称为**写失效Partial Page Write**

**Double Write**其实就是写两次，在修改记录redo日志前，先做个副本留个“备胎”

