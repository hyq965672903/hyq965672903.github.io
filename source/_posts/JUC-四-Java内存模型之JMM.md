---
title: JUC(四)-Java内存模型之JMM
index_img: https://tva1.sinaimg.cn/large/9bd9b167gy1g2rle8s7wkj21hc0u04fj.jpg
banner_img: https://tva1.sinaimg.cn/large/9bd9b167ly1g2rl13okgsj21hc0u0qfz.jpg
abbrlink: f189a462
date: 2022-06-18 19:20:46
tags:
categories:
description:
---

Java内存模型下对变量的规则，多线程下变量赃读取脏写的原因，内存屏障及其原理分析，以及Volatile的关键字的作用，原理。使用场景

<!-- more -->

Java并发多线程与高并发(二)-并发基础 已经对概念进行的基础的覆盖，接下来逐步加深

## 多线程先行发生原则之happens-before

### happens-before之8条

- **程序次序规则**（Program Order Rule）：写在前面的操作先行发生于写在后面的操作
- **管程锁定规则**（Monitor Lock Rule）：一个 unlock 操作先行发生于后面对同一个锁的 lock 操作
- **volatile 变量规则**（Volatile Variable Rule）：对一个 volatile 变量的写操作先行发生于后面对这个变量的读操作，**前面的写对后面的读是可见的**。这里的 “后面” 同样是指时间上的先后。
- **传递性**（Transitivity）：如果操作 A 先行发生于操作 B，操作 B 先行发生于操作 C，那就可以得出操作 A 先行发生于操作 C 的结论。
- **线程启动规则**（Thread Start Rule）：Thread 对象的 start() 方法先行发生于此线程的每一个动作。
- **线程中断规则**（Thread Interruption Rule）：对线程 interrupt() 方法的调用先行发生于被中断线程的代码检测到中断事件的发生，可以通过 Thread 对象的 interrupted() 方法检测到是否有中断发生。
- **线程终止规则**（Thread Termination Rule）：**线程中的所有操作都先行发生于对此线程的终止检测。**我们可以通过 Thread 对象的 join() 方法是否结束、Thread 对象的 isAlive() 的返回值等手段检测线程是否已经终止执行。
- **对象终结规则**（Finalizer Rule）：一个对象的初始化完成（构造函数执行结束）先行发生于它的 finalize() 方法的开始。

## Volatile内存屏障（难点）

> 并发编程的三大问题：原子性 可见性  有序性
>
> volatile 保证了可见性和有序性

volatile：

当写一个volatile变量时，JMM会把该线程对应的本地内存中的共享变量值`立即刷新回主存`中

当读一个volatile变量时，JMM会把该线程对应的本地内存设置为无效，直接从主存中读取共享变量 

所以volatile的写内存语义是直接刷新到主内存中，读的内存语义是直接从主内存中读取

### 内存屏障(保证了可见性)

内存屏障指令，Java内存模型的重排规则会要求Java编译器在生成JVM指令时插入特定的内存屏障指令

对一个volatie域的写，happens-before与任意后续对这个volatile域的读，也叫写后读

 底层是C++

loadload  storestore  loadstore  storeload 四个内存屏障策略

| 屏障类型   | 指令示例                 | 说明                                                         |
| ---------- | ------------------------ | ------------------------------------------------------------ |
| loadload   | load1;loadload;load2     | 保证load1读取操作在load2读取操作之前执行                     |
| storestore | store1;storestore;store2 | 在store2及其后的写操作执行前，保证store1的写操作已经刷新到主内存 |
| loadstore  | load1;storestore;store2  | 在store2及其后的写操作执行前，保证load1的读操作已读取结束    |
| storeload  | store1;storestore;load2  | 保证store1的写操作已刷新到主内存之后，load之后的读操作才能执行 |

总结：（重点）

1、当第一个操作Volatile读时候，无论第二个操作是什么，都不能重排序，保证Volatile读之后的操作不会重排序到Volatile读之前

2、当第二个操作是Volatile写时候，无论第一个操作是什么，都不能重排序，保证Volatile写之前的操作不会被重排序到Volatile写之后

3、当第一个操作是Volatile写时，第二个操作是Volatile读时，不能重排



延伸：使用两个Volatile时候，只有第一个是Volatile读，第二个也是Volatile读才可以重排，其他所有情况均不可以重排

### volatile

写总结：

1、在每个volatile写操作的前面插入一个storestore屏障

2、在每个volatile写操作的后面插入一个storeload屏障

读总结

1、在每个volatile读操作后面插入一个loadload屏障

2、在每个volatile读操作后面插入一个loadstore屏障

## Volatile变量的读写过程

> **read(读取)→load(加载)→use(使用)→assign(赋值)→store(存储)→write(写入)→lock(锁定)→unlock(解锁)**

read： 作用于主内存，将变量的值从主内存传输到工作内存，主内存到工作内存。

laod：作用于工作内存，将read从主内存传输的变量值放入工作内存变量副本中，即数据加载。

use：作用于工作内存，将工作内存变量副本的值传递给执行引擎，每当JVM遇到需要该变量的字节码指令时会执行该操作。

assign：作用于工作内存，将从执行引擎接收到的值赋值给工作内存变量，每当JVM遇到一个给变量赋值字节码指令时会执行该操作。

write：作用于主内存，将store传输过来的变量值赋值给主内存中的变量。

由于上述操作，只能保证单条指令的原子性，针对多条指令的组合性原子保证，没有大面积加锁，所以JVM另外提供了两个原子指令：

lock：作用于主内存，将一个变量标记为一个线程独占的状态，只是写时候加锁，就只是锁了写变量的过程。

unlock：作用于主内存，将一个处于锁定状态的变量释放，然后才能被其它线程占用。

## Volatile 为什么没有原子性？

复合性的操作不具有原子性

example:i++

i++ 被拆分处理三个指令：

1、getfield拿到原始i

2、执行iadd进行加一操作

3、执行putfield把累加后的值写回

如果第二个线程在第一个线程读取旧值和写回新值期间读取了i的值，那么第二个线程就会和第一个线程看到同一个值，并执行加一操作，导致线程安全失败。

### 本质原因

内存屏障保证了read(读取)→load(加载)→use(使用)  以及assign(赋值)→store(存储)→write(写入)成为了两个不可分割的原子操作，但是在use和assign之间依然有真空期，这个时候如i++ 这样多个操作就会导致多线程安全问题

## Volatile的应用场景

> 通常用作保存某个boolean 或者 int 值

 





