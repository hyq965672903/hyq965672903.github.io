---
title: Java并发多线程与高并发(二)-并发基础
index_img: https://file.hyqup.cn/img/559ad287880511ebb6edd017c2d2eca2.png
banner_img: https://file.hyqup.cn/img/d017bd50880411ebb6edd017c2d2eca2.jpg
abbrlink: 81ef5aad
date: 2021-04-28 23:33:36
tags:
categories:
description:
---

在多线程的情况下，从java内存模型和CPU乱序优化浅谈多线程不安全产生的原因

<!-- more -->

> 序章中我们看到多线程中看到基数Demo计数出现误差，本章节从计算机原理和Java虚拟机相关探讨为什么会产生这些异常

## CPU多级缓存

cpu执行频率太快，快到主存跟不上，这样在处理器处理过程中，CPU常常需要等待主存，浪费资源。所以多级缓存（cache）的出现，就是**为了解决CPU和内存之间速度不匹配的问题**（cpu->cache->memory）

### CPU 缓存的意义

- **时间局部性（Temporal Locality）**：如果一个信息项正在被访问，那么在近期它很可能还会被再次访问。
- **空间局部性（Spatial Locality）**：如果一个存储器的位置被引用，那么将来他附近的位置也会被引用。

### CPU缓存一致性协议(MESI)

M 修改 (Modified)  E 独享、互斥 (Exclusive) S 共享 (Shared) I 无效 (Invalid)

该协议目的是**为了保证CPU cache之间的共享数据的一致性**

更多详细信息请查阅资料

### CPU多级缓存-乱序执行优化

**处理器为提高运算速度而做出违背代码原有顺序的优化**

```java
int a=1;	//①
int b=2;	//②
int c=a+b;	//③
```

cpu乱序执行的时候 可能顺序变为②->①->③

在单线程单核的情况下不会出现问题，复杂的顺序的时候**多线程下可能会存在问题**

## Java内存模型

了解Java内存模型，了解Java内存模型如何对上述进行优化的

Java内存模型-JMM(Java Memory Model) Java内存模式是一种虚拟机规范

![java-memory-model-2](https://file.hyqup.cn/img/java-memory-model-2-1620570065967.png)

cpu结构图

![6](https://file.hyqup.cn/img/6.jpg)

Java内存模型抽象结构图

![8](https://file.hyqup.cn/img/8.jpg)











```markdown
## 参考
[^1]: http://ifeve.com/java-memory-model-6/
```

