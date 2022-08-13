---
title: JUC(五)-CAS
index_img: https://tva2.sinaimg.cn/large/9bd9b167gy1g2rkorfsdej21hc0u0n3p.jpg
banner_img: https://tva2.sinaimg.cn/large/9bd9b167gy1g2rlmxizo9j21hc0u0h5m.jpg
abbrlink: 16f16ca5
date: 2022-06-19 10:19:11
tags:
categories:
description:
---

 CAS底层原理和UnSafe的理解,自旋锁，ABA问题解决

<!-- more -->

## 概述

CAS（Compare-and-Swap），即比较并替换

没有使用原子类之前：采用volatile+synchronized重锁机制保证并发安全

使用原子类之后：Atomic时候无需加锁保证安全

## 原理

CAS有三个操作数，位置内存值V，旧的预期值A,要修改的更新值B。

当且仅当旧的预期值A和内存V相同时，将内存值V修改为B,否则就什么都不做或者`重来`

### Unsafe类（开发时候不要使用，线程不安全）

Unsafe类中方法都是natice修饰的本地方法。也就是说Unsafe类的方法是直接调用操作系统底层资源执行相应任务

Unsafe提供的CAS方法，底层实现即为CPU指令cmpxchg,也就是说CAS的原子性实际上是CPU实现的，cmpxchg会在多核系统总线加锁。

## 原子类AtomicXXX

> Atomic原理就是使用cas+volatile+native实现的

结合java内存模型

### AtomicInteger


假设线程A和线程B两个线程同时执行getAndAddInt操作（分别跑在不同CPU上）：

1  AtomicInteger里面的value原始值为3，即主内存中AtomicInteger的value为3，根据JMM模型，线程A和线程B各自持有一份值为3的value的副本分别到各自的工作内存。

2  线程A通过getIntVolatile(var1, var2)拿到value值3，这时线程A被挂起。

3  线程B也通过getIntVolatile(var1, var2)方法获取到value值3，此时刚好线程B没有被挂起并执行compareAndSwapInt方法比较内存值也为3，成功修改内存值为4，线程B打完收工，一切OK。

4  这时线程A恢复，执行compareAndSwapInt方法比较，发现自己手里的值数字3和主内存的值数字4不一致，说明该值已经被其它线程抢先一步修改过了，那A线程本次修改失败，`只能重新读取重新来一遍了`。

5  线程A重新获取value值，因为变量value被volatile修饰，所以其它线程对它的修改，线程A总是能够看到，线程A继续执行compareAndSwapInt进行比较替换，直到成功。

### AtomicReference

AtomicReference<Order> atomicReferenceUser = new AtomicReference<>();

这里Order 例指所有业务对象

atomicReferenceUser .compareAndSet

## 自旋锁(spinlock)

> 尝试获取锁的线程不会立即阻塞，而是采用循环的方式去尝试获取锁,当线程发现锁被占用时，会不断循环判断锁的状态，直到获取.

好处是减少线程上下文切换的消耗，缺点是循环会消耗CPU

 

```java
package cn.hyqup.juc;

import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicReference;

/**
 * Copyright © 2022 灼华. All rights reserved.
 *
 * @author create by hyq
 * @version 0.1
 * @date 2022/6/19
 * @description:
 */
public class SpinLockDemo {
    AtomicReference<Thread> atomicReference = new AtomicReference<>();

    public void myLock() {
        Thread thread = Thread.currentThread();
        System.out.println(Thread.currentThread().getName() + "\t come in");
        while (!atomicReference.compareAndSet(null, thread)) {

        }
    }

    public void myUnLock() {
        Thread thread = Thread.currentThread();
        atomicReference.compareAndSet(thread, null);
        System.out.println(Thread.currentThread().getName() + "\t myUnLock over");
    }

    public static void main(String[] args) {
        SpinLockDemo spinLockDemo = new SpinLockDemo();

        new Thread(() -> {
            spinLockDemo.myLock();
            try {
                TimeUnit.SECONDS.sleep(5);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            spinLockDemo.myUnLock();
        }, "A").start();

        //暂停一会儿线程，保证A线程先于B线程启动并完成
        try {
            TimeUnit.SECONDS.sleep(1);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        new Thread(() -> {
            spinLockDemo.myLock();
            spinLockDemo.myUnLock();
        }, "B").start();

    }

}
```

## CAS缺点及ABA问题

1、如果CAS失败，会一直进行尝试。如果CAS长时间一直不成功，造成锁饥饿，可能会给CPU带来很大的开销。

2、CAS会导致“ABA问题”。



比如说一个线程one从内存位置V中取出A，这时候另一个线程two也从内存中取出A，并且线程two进行了一些操作将值变成了B，
然后线程two又将V位置的数据变成A，这时候线程one进行CAS操作发现内存中仍然是A，然后线程one操作成功。

尽管线程one的CAS操作成功，但是不代表这个过程就是没有问题的。

### ABA问题的解决（AtomicStampedReference ）

 atomicStampedReference.getStamp();





