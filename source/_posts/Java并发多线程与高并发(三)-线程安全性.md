---
title: Java并发多线程与高并发(三)-线程安全性
index_img: https://file.hyqup.cn/img/d822e44f880411ebb6edd017c2d2eca2.jpg
banner_img: https://file.hyqup.cn/img/a9982f7441bd4e07a3b1a26af43457d9.jpg
abbrlink: 6cdf97ff
date: 2021-04-28 23:40:23
tags:
categories:
description:
---

针对性的学习volitale、synconized、ThreadLocal 对象，了解特性，给予案例，及应用场景，简述Java为了线程安全做了哪些工作

<!-- more -->

## 三大性质简介

在 Java 多线程中，Java 提供了一系列与并发处理相关的关键字，比如`volatile`、`synchronized`、`final`、`concurren`包等。

 前面有说过多线程中访问共享变量可能会导致一系列由于资源竞争所产生的的线程不安全问题，Java提供了一些关键字来解决，但本质上是java 内存模型围绕`原子性`、`可见性`和`顺序性` 来设计实现

- 原子性

  > **一个操作是不可中断的，要么全部执行成功要么全部执行失败**

  可类比事务

- 可见性

  > **可见性是指当一个线程修改了共享变量后，其他线程能够立即得知这个修改**

- 有序性

  > **CPU为了优化执行，编译器和处理器会进行指令重排序，但是，在Java中认为一个本线程内操作都是有序的，一个线程观察另一个线程的时候，所有操作都是无序的。这个时候我们就需要做一些操作来保证有序，从而保证程序正确**

## synconized 的使用

>  synconized 加锁，所有同步在一个对象上的同步块在同时只能被一个线程进入并执行操作，所有其他等待进入该同步块的线程将被阻塞，直到执行该同步块中的线程退出。

- 同步普通方法，锁的是当前对象。
- 同步静态方法，锁的是当前 `Class` 对象。
- 同步块，锁的是 `()` 中的对象。

```
 synchronized (this) {
    this.a = a + 1;
}
```

synconized 是可以满足原子性的

缺点： synconized 是比较耗费系统资源的，所以要尽可能小范围的使用



## volitale的使用

> volitale 的作用是保证内存可见性，如果一个成员变量被 valitale 修饰，在 A 线程中修改了 valitale 变量的值，对其他线程来说该修改是可见的。

可见性只能保证每次读取的是最新的值，但是volatile没办法保证对变量的操作的原子性。

举个例子：volitale  int i=10  开启10个线程 每次+1

两个线程A、B，A读取i的值为10，然后被阻塞，线程B读取i的值为10 执行加一，然后写入主存，线程A得到执行权，由于在A的工作空间已经获取值为10 ，执行加1，然后将11写回主存。这也就是执行完后i 的值永远小于等于20的原因

**因此 synchronized 和 volitale 的区别是：加锁机制既可以确定可见性又可以保证原子性，而 volatile 变量只能确保可见性。**

一种典型的使用场景：检查某个判断标记判断是否退出循环

```
valitale boolean isExit;
while(!isExit) {
    doSomething();
}
```

这样当 isExit 标志被另外一个线程修改为 true 的时候，执行判断的线程就能够准确的读取到 isExit 的值，从而退出循环。

## ThreadLocal 的使用

>  ThreadLocal 是为每个线程都提供一份变量的副本，从而实现同时访问而不受影响。

主要的应用场景：

- 多线程多层级传递对象的时候，使用 ThreadLocal 可以代替一些参数的显式传递；

- 全局存储用户信息

- 解决线程安全问题，如`SimpleDateFormat`，`ThreadLocal` 可以确保每个线程都可以得到单独的一个 `SimpleDateFormat` 的对象

