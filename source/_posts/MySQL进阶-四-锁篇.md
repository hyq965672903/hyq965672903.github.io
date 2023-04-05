---
title: MySQL进阶(四)-锁篇
index_img: /img/default.png
banner_img: /img/default.png
date: 2023-04-02 23:09:13
tags:
categories:
description:
---

MySQL中的锁，共享锁、排它锁、全局锁、表锁、行锁、读锁、写锁等

<!-- more -->

## 锁分类

MySQL加锁过程

```sql
update tab_user set name='曹操' where id = 1;
```

![加锁过程](http://file.hyqup.cn/img/%E5%8A%A0%E9%94%81%E8%BF%87%E7%A8%8B.png)

### **功能划分**

**`共享锁（shared lock）`**：也叫S锁、读锁，读锁是共享的，读锁之间互相不阻塞

```sql
select … lock in share mode
```

**`排他锁（exclusive lock）`**：也叫X锁、写锁，写锁是排他的，写锁阻塞其他的读和写锁

```sql
select … for update
```

### **按粒度分**

- 全局锁：锁整Database，由MySQL的SQL layer层实现
- 表级锁：锁某Table，由MySQL的SQL layer层实现
- 行级锁：锁某Row的索引，也可锁定行索引之间的间隙，由存储引擎实现【InnoDB】
  - 记录锁（Record Locks）：锁定索引中一条记录
  - 间隙锁（Gap Locks）：仅仅锁住一个索引区间
  - 临键锁（Next-Key Locks）：记录锁和间隙锁的组合，**解决幻读问题**
  - 插入意向锁(Insert Intention Locks)：做insert时添加的对记录id的锁
  - 意向锁：存储引擎级别的“表级”锁

## 全局锁

全局锁是对整个数据库实例加锁，加锁后整个实例就处于只读状态，将阻塞DML、DDL及已经更新但未提交的语句

应用场景：全库逻辑备份

### **表锁相关命令**

加锁命令：

```sql
flush tables with read lock;
```

释放锁命令：

```sql
unlock tables;
```

注：**`断开Session锁自动释放全局锁`**

## 表级锁

- `表读锁（Table Read Lock）`，阻塞对当前表的写，但不阻塞读

- `表写锁（Table Write Lock）`，阻塞对当前表的读和写

- `元数据锁（Meta Data Lock，MDL)`，不需要显式指定，在访问表时会被自动加上，作用保证读写的正确性
  - 当对表做**增删改查**操作的时**加元数据读锁**
  - 当对表做**结构变更**操作的时**加元数据写锁**
- `自增锁(AUTO-INC Locks)`， AUTO-INC是一种特殊的表级锁，自增列事务性插入操作时产生

### **表锁相关命令**

- 查看表锁定状态：show status like 'table_locks%’;

- 添加表读锁：lock table t read;

- 添加表写锁：lock table t write;

- 查看表锁情况：show open tables;

- 删除表锁：unlock tables;

## **行锁**

> MySQL的行级锁是由存储引擎实现，**InnoDB行锁**是通过给索引上的**索引项加锁来实现**

注：**只有通过索引条件检索的数据InnoDB才使用行级锁，否则InnoDB都将使用表锁**

### **按范围分：**

记录锁（Record Locks）、间隙锁（Gap Locks）、临键锁（Next-Key Locks）、插入意向锁（Insert Intention Locks）

### 按功能分：

读锁：允许事务**去读**目标行，阻止其他事务**更新**。阻止其他事务加写锁，但不阻止加读锁

写锁：允许事务**更新**目标行，阻止其他事务**获取或修改**。同时阻止其他事务加读锁和写锁。

### 如何加锁?

- 对于Update、Delete和Insert语句，InnoDB会自动给涉及数据集加**写锁**

- 对于普通Select语句，**InnoDB不会加任何锁**

- 事务手动给Select记录集加读锁或写锁

- 手动加锁

  - **添加读锁**

    ```sql
    select * from xxx where id = xx lock in share mode;
    ```

  - **添加写锁**

    ```sql
    select * from xxx where id = xx for update;
    ```

### **行锁四兄弟：记录、间隙、临键和插入意向锁**

#### **记录锁**

> 记录锁（Record Locks）仅仅锁住索引记录的一行

- 记录锁锁住的永远是索引，而非记录本身，即使该表上没有任何显示索引 
- 没有索引，InnoDB会创建隐藏列ROWID的聚簇索引

#### 间隙锁

> 间隙锁（Gap Locks）仅仅锁住一个索引区间，开区间，不包括双端端点和索引记录

- 在索引记录间隙中加锁，并不包括该索引记录本身
- 间隙锁可用于防止幻读，保证索引间隙不会被插入数据

#### 临键锁

> 临键锁（Next-Key Locks）相当于记录锁 + 间隙锁，左开右闭区间

- 默认情况下，InnoDB使用临键锁来锁定记录，但会在不同场景中退化
- 场景01-唯一性字段等值（=）且记录存在，退化为**记录锁**
- 场景02-唯一性字段等值（=）且记录不存在，退化为**间隙锁**
- 场景03-唯一性字段范围（< >），还是**临键锁**
- 场景04-非唯一性字段，默认是**临键锁**

#### **插入意向锁**

> 插入意向锁（Insert Intention Locks）是一种在 INSERT 操作之前设置的一种特殊的间隙锁

- 插入意向锁表示了一种插入意图，即当多个不同的事务，同时往**同一个索引**的**同一个间隙**中插入数据的时候，它们互相之间无需等待，即不会阻塞
- 插入意向锁不会阻止插入意向锁，但是插入意向锁会阻止其他**间隙写锁（排他锁）、记录锁**

### 加锁规则

**主键索引：**

- 等值条件，命中加记录锁

- 等值条件，未命中加间隙锁

- 范围条件，命中包含where条件的临键区间加临键锁

- 范围条件，没有命中加间隙锁

**辅助索引：**

- 等值条件，命中，命中记录辅助索引项，回表主键索引项加记录锁，辅助索引项两侧加间隙锁

- 等值条件，未命中加间隙锁

- 范围条件，命中包含where条件的临键区间加临键锁。命中记录回表主键索引项加记录锁

- 范围条件，没有命中加间隙锁
