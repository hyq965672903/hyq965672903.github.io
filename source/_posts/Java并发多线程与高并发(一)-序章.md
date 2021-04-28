---
title: Java并发多线程与高并发(一)-序章
index_img: https://file.hyqup.cn/img/792a4b6934fc491b81342206e3192e59.png
banner_img: /img/default.png
abbrlink: fc3007d7
date: 2021-04-28 22:01:58
tags:
categories:
description: 
---

并发线程体验，概述本系列讲的内容（线程并发安全/高并发解决方案）

<!-- more -->

# 初体验

```java
package cn.hyqup.concurrency.experience;

import lombok.extern.slf4j.Slf4j;

import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.Semaphore;

/**
 * Copyright © 2021灼华. All rights reserved.
 *
 * @author create by hyq
 * @version 1.0
 * @date 2021/4/21
 * @description:
 */
@Slf4j
public class CountExample {

    private static int threadTotal = 100;
    private static int clientTotal = 3000;

    private static long count = 0;

    public static void main(String[] args) {

        ExecutorService executorService = Executors.newCachedThreadPool();
        final Semaphore semaphore = new Semaphore(threadTotal);
        for (int i = 0; i < clientTotal; i++) {
            executorService.execute(() -> {
                try {
                    semaphore.acquire();
                    add();
                    semaphore.release();
                } catch (Exception e) {
                    log.error("exception", e);
                }
            });
        }
        executorService.shutdown();
        log.info("count:{}", count);
    }


    private static void add() {
        count++;
    }
}

```

输出结果:

```java
22:14:39.877 [main] INFO cn.hyqup.concurrency.experience.CountExample - count:2958

Process finished with exit code 0
```

多线程下上述案例没有达到**3000**这个预期的值

> **Semaphore** 信号量，Java 并发库 的Semaphore可以控制某个资源可被同时访问的个数，通过 acquire() 获取一个许可，如果没有就等待，而 release() 释放一个许可

# 概念

**进程**：

{% note success %}

进程是程序一次执行过程。比如程序从启动到销毁是一个进程

{% endnote %}

**线程**：

{% note success %}

线程是比进程更小的单位。一个进程在执行过程中可以产生多个线程

{% endnote %}

**并发与多线程的考虑点：**

{% note success %}

考虑点在于多线程操作资源时候保证线程安全，保证程序不出现错误

{% endnote %}

**高并发考虑点：**

{% note success %}

考虑点在于提高服务在处理多个请求时候，我们程序要尽可能的去提升程序性能，保证程序的抗压能力。可以从水平扩展，垂直扩展去实现

{% endnote %}

# 线程的状态

- **新建状态**（NEW）:新建状态，线程被构建，还未调用start()方法
- **可运行状态**（RUNNABLE）: 线程对象创建后，其他线程(比如 main 线程）调用了该对象 的 start ()方法。该状态的线程位于可运行线程池中，等待被线程调度选中，获 取 cpu 的使用权
- **运行状态**（RUNNING）:可运行状态( runnable )的线程获得了 cpu 时间片（ timeslice ） ，执行程序代码
- **阻塞**（BLOCK）:阻塞状态是指线程因为某种原因放弃了 cpu 使用权，也即让出了 cpu timeslice ，暂时停止运行。直到线程进入可运行( runnable )状态，才有 机会再次获得 cpu timeslice 转到运行( running )状态
- **死亡**（DEAD）:线程 run ()、 main () 方法执行结束，或者因异常退出了 run ()方法，则该线程结束生命周期。死亡的线程不可再次复生。