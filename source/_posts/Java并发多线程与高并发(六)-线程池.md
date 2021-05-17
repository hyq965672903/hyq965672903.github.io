---
title: Java并发多线程与高并发(六)-线程池
index_img: https://file.hyqup.cn/img/2ae41a1d880511ebb6edd017c2d2eca2.png
banner_img: https://file.hyqup.cn/img/2ae41a1d880511ebb6edd017c2d2eca2.png
abbrlink: 2e7e1308
date: 2021-04-28 23:50:21
tags:
- 线程池
- executors
categories:
- Java
- Java进阶-并发
description:
---

针对性的学习线程池框架 ExecutorService ，包括核心概念，核心参数，创建使用等

<!-- more -->

## 创建线程的方式

### 继承Thread

- 定义类继承Thread，重写run方法（线程执行的内容），又叫执行体
- 创建Thread继承类的实例，调用start()启动线程

```java
package cn.hyqup.concurrency.thread;

import lombok.extern.slf4j.Slf4j;

/**
 * Copyright © 2021灼华. All rights reserved.
 *
 * @author create by hyq
 * @version 1.0
 * @date 2021/5/16
 * @description:
 */
@Slf4j
public class ThreadExample1 extends Thread {
    /**
     * 重写run方法
     */
    @Override
    public void run() {
        log.info("继承thread开启线程");
    }

    public static void main(String[] args) {
        for (int i = 0; i < 5; i++) {
            new ThreadExample1().start();
        }
    }
}

```



### 实现Runable

- 定义类实现runable，重写run方法（线程执行的内容），又叫执行体

- 创建Runnable实现类的实例, new Thread作为入参传递。得到一个线程对象

- 调用start()启动线程

  ```java
  package cn.hyqup.concurrency.thread;
  
  import lombok.extern.slf4j.Slf4j;
  
  /**
   * Copyright © 2021灼华. All rights reserved.
   *
   * @author create by hyq
   * @version 1.0
   * @date 2021/5/16
   * @description:
   */
  @Slf4j
  public class ThreadExample2 implements Runnable {
      /**
       * 重写run方法
       */
      @Override
      public void run() {
          log.info("实现Runnable开启线程");
      }
  
      public static void main(String[] args) {
          for (int i = 0; i < 5; i++) {
              ThreadExample2 threadExample2 = new ThreadExample2();
              new Thread(threadExample2).start();
          }
      }
  }
  
  ```

### 实现 Callable ，并结合 Future

- 定义Callable接口的实现类，实现call()方法，又叫执行体，并且定义**返回值**类型
- 创建Callable实现类的实例，使用FutureTask类来包装Callable对象，并且FutureTask返回类型和Callable的返回类型一致
- new Thread，FutureTask做入参得到一个Thread线程对象
- 调用Thread的start()方法，执行开启线程
- 调用FutureTask对象的get()方法，得到子线程返回值

```java
package cn.hyqup.concurrency.thread;

import lombok.extern.slf4j.Slf4j;

import java.util.concurrent.Callable;
import java.util.concurrent.ExecutionException;
import java.util.concurrent.FutureTask;

/**
 * Copyright © 2021灼华. All rights reserved.
 *
 * @author create by hyq
 * @version 1.0
 * @date 2021/5/16
 * @description:
 */
@Slf4j
public class ThreadExample3 implements Callable<Integer> {

    @Override
    public Integer call() throws Exception {
        return 100;
    }
    public static void main(String[] args) throws ExecutionException, InterruptedException {
        ThreadExample3 threadExample3 =new ThreadExample3();
        FutureTask<Integer> futureTask = new FutureTask<>(threadExample3);
        for (int i = 0; i < 5; i++) {
            new Thread(futureTask).start();
            Integer integer = futureTask.get();
            log.info("子线程返回的值"+integer);
        }


    }


}

```



## 使用线程池创建线程（这里介绍JDK自带的Executors ）

除了上述三种创建线程的方式，我们还可以通过线程池来创建线程

### new Thread 的弊端

- 每次都new对象，性能差，复用性差
- 缺乏线程之间统一管理，造成资源浪费甚至可能内存溢出
- 缺乏更多功能，如定时执行、定期执行、线程中断。

### 线程池的好处

- 重用存在的线程，减少对象创建、消亡的开销，性能佳。
- 可有效控制最大并发线程数，提高系统资源的使用率，同时避免过多资源竞争，避免堵塞。
- 提供定时执行、定期执行、单线程、并发数控制等功能。

### 线程池核心概念及参数介绍

```java
  public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory,
                              RejectedExecutionHandler handler) 
```

- corePoolSize(核心线程数)
- maximumPoolSize（最大线程数）
- keepAliveTime（线程活动保持存活时间）
- unit（时间单位）
- workQueue（任务存储队列）
  - SynchronousQueue(直接交接)：没有工作队列缓冲区，每次直接扔到线程去处理，不能处理则抛异常
  - LinkedBlockingQueue(无界队列)：队列永远不会满，也就是永远用不上最大线程数
  - ArrayBlockingQueue(有界队列)：基于数组结构的有界阻塞队列，遵循先进先出
- ThreadFactory（创建线程的工厂）
- handler（拒绝策略）
  - DiscardPolicy：不处理，丢弃掉。
  - AbortPolicy：直接抛出异常。
  - DiscardOldestPolicy：丢弃队列里最近的一个任务，并执行当前任务。
  - CallerRunsPolicy：只用调用者所在线程来运行任务。

### 增减线程的特点

### 自动创建线程池的风险分析

 针对线程池默认的创建线程池的构造方法的分析

- newFixedThreadPool

  创建一个**定长**线程池，可控制线程最大并发数，超出的线程会在**队列中等待**。**队列用的是LinkedBlockingQueue（新任务一直向队列中放）**

  缺点：容易造成大量内存占用，可能会导致OOM

- newSingleThreadExector

  创建一个定长为1的线程池。类似于newFixedThreadPool，**队列用的是LinkedBlockingQueue（新任务一直向队列中放）**

  缺点：容易造成大量内存占用，可能会导致OOM

- newCachedThreadPool

  创建一个可缓存线程池，核心数量为0，**最大线程数为整数最大值**

  缺点：新来任务直接创建线程执行，也可能造成OOM

- newScheduledThreadPool

  建一个**定长线程池**，支持定时及周期性任务执行,可以延迟，可以一定频率

总结：正确的线程池的创建应该是手动创建线程池，依据是调研后，根据业务定制



