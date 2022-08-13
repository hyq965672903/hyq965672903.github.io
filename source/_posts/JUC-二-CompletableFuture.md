---
title: JUC(二)-CompletableFuture
index_img: https://tva3.sinaimg.cn/large/9bd9b167gy1g2rlsmpf8tj21hc0u0aut.jpg
banner_img: https://tva2.sinaimg.cn/large/9bd9b167gy1g2rl7g0pfej21hc0u0tns.jpg
abbrlink: 918a696d
date: 2022-06-18 09:11:57
tags:
categories:
 - Java
 - JUC
description:
---

 了解JUC的概念，学习并找我CompletableFuture的使用及其应用场景

<!-- more -->

## 什么是JUC

J.U.C并发包，即`java.util.concurrent`包，是JDK的核心工具包，是JDK1.5之后，由 Doug Lea实现并引入。

整个`java.util.concurrent`包，按照功能可以大致划分如下：

- juc-locks 锁框架
- juc-atomic 原子类框架
- juc-sync 同步器框架
- juc-collections 集合框架
- juc-executors 执行器框架

## CompletableFuture

### Callable接口与Runnable接口

> `Callable`接口类似于[`Runnable`](https://blog.csdn.net/java/lang/Runnable.html) ，因为它们都是为其实例可能由另一个线程执行的类设计的。 然而，`Runnable`不返回结果，也不能抛出被检查的异常。具体区别如下

1. Callable功能更强大些。
2. 相比run()方法，可以有返回值。
3. 方法可以抛出异常。
4. 支持泛型的返回值。
5. 需要借助FutureTask类，比如获取返回结果。

### Future 接口

```
public interface Future<V> {

	//尝试取消执行此任务。
    boolean cancel(boolean mayInterruptIfRunning);
	//如果此任务在正常完成之前被取消，则返回 true 。
    boolean isCancelled();
	//返回 true如果任务已完成。
    boolean isDone();
	//等待计算完成，然后检索其结果
    V get() throws InterruptedException, ExecutionException;
	//如果需要等待最多在给定的时间计算完成，然后检索其结果（如果可用）
    V get(long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;
}
```

Future接口用于对异步任务的执行结果进行取消、查询是 否完成、获取结果等

### FutureTask

> FutureTask是Future的实现，可用于包装[`Callable`](https://blog.csdn.net/java/util/concurrent/Callable.html)或[`Runnable`](https://blog.csdn.net/java/lang/Runnable.html)对象,提交给Thread执行，这样就可以使用FutureTask执行一些Feture的特性方法
>

```java
package cn.hyqup.juc;

import java.util.concurrent.Callable;
import java.util.concurrent.ExecutionException;
import java.util.concurrent.FutureTask;

/**
 * Copyright © 2022 灼华. All rights reserved.
 *
 * @author create by hyq
 * @version 0.1
 * @date 2022/6/18
 * @description:
 */
class CookDinner implements Callable<Integer> {

    @Override
    public Integer call() throws Exception {
        for (int i = 0; i < 50; i++) {
            System.out.println("正在做第" + i + "份饭");
        }
        return 50;
    }
}

public class FutureTaskDemo {


    public static void main(String[] args) {
        CookDinner cookDinner = new CookDinner();
        FutureTask<Integer> futureTask = new FutureTask<Integer>(cookDinner);
        Thread thread = new Thread(futureTask);
        thread.setName("thread1");
        thread.start();

        try {
            Integer integer = futureTask.get();
            System.out.println("总共做了"+integer+"份饭");
        } catch (InterruptedException e) {
            e.printStackTrace();
        } catch (ExecutionException e) {
            e.printStackTrace();
        }


    }
}

```

#### 总结FutureTask

优点：可以包装异步任务，拿到执行结果

缺点：get方法，会阻塞，就失去了并发的特性，变成同步方法

使用总结：实际使用是尽量**不要阻塞**，采用get(long timeout, TimeUnit unit)，带**超时时间**，并且**放在最后**，但这种本质上只是算一种减轻，并不是算替代。

更好的方式是采用轮询方式，CompletableFuture

### CompletableFuture

CompletableFuture是对Future的改进

```java
public class CompletableFuture<T> implements Future<T>, CompletionStage<T> {
}
```

#### Demo1

CompletionStage介绍：是Java8新增得一个接口，用于异步执行中的阶段处理，代表异步计算中的一个阶段或步骤

```java
package cn.hyqup.juc;

import java.util.concurrent.*;

/**
 * Copyright © 2022 灼华. All rights reserved.
 *
 * @author create by hyq
 * @version 0.1
 * @date 2022/6/18
 * @description:
 */
public class CompletableFutureDemo {

    public static void main(String[] args) {
        ThreadPoolExecutor threadPoolExecutor = new ThreadPoolExecutor(1, 10, 1L, TimeUnit.SECONDS, new LinkedBlockingQueue<>(20), Executors.defaultThreadFactory(), new ThreadPoolExecutor.AbortPolicy());
        CompletableFuture<Void> future1 = CompletableFuture.runAsync(() -> {
            System.out.println(Thread.currentThread().getName()+"异步执行");
        });
        CompletableFuture<Void> future2 = CompletableFuture.runAsync(() -> {
            System.out.println(Thread.currentThread().getName()+"异步执行2");
        },threadPoolExecutor);
        threadPoolExecutor.shutdown();
    }
}
```

基本写法，是继承了Future 的特性，所以此时调用Future.get 同样会阻塞主线程

#### Demo2

 应对Future的完成回调功能

当多个一步计算互相独立，同时第二个任务又依赖第一个的结果时

```java
package cn.hyqup.juc;

import java.util.concurrent.*;

/**
 * Copyright © 2022 灼华. All rights reserved.
 *
 * @author create by hyq
 * @version 0.1
 * @date 2022/6/18
 * @description:
 */
public class CompletableFutureDemo2 {

    public static void main(String[] args) {
        ThreadPoolExecutor threadPoolExecutor = new ThreadPoolExecutor(1, 10, 1L, TimeUnit.SECONDS, new LinkedBlockingQueue<>(20), Executors.defaultThreadFactory(), new ThreadPoolExecutor.AbortPolicy());
        CompletableFuture.supplyAsync(() -> {
            try {
                TimeUnit.SECONDS.sleep(1);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            return 1;
        }).thenApply(fn -> {
            return fn + 2;
        }).whenComplete((v, e) -> {
            System.out.println("结果" + v);
        }).exceptionally(e -> {
            e.printStackTrace();
            return null;
        });
        threadPoolExecutor.shutdown();

        System.out.println("main 结束");
        // 前面是守护线程,保证前面线程结果执行完
        try {
            TimeUnit.SECONDS.sleep(5);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

这种异步的特性就是CompletableFuture对CompletionStage的继承实现。

#### CompletableFuture常用API

##### 获得结果和触发计算

- get 获取结果，捕获异常

- get(long timeout, TimeUnit unit) 超时过时不候

- join  不用抛异常

-  T getNow(T valueIfAbsent) 没计算完返回默认值



-  boolean complete(T value)  打断成功，返回默认值，打断失败，返回真实值

##### 对计算结果进行处理

- thenApply 线程串行化，有异常终止
- handle 有异常会继续执行

##### 对计算结果进行消费

- thenAccept 有一个输入参数，没有返回，Consumer接口

##### 对计算速度进行选用

-  applyToEither 谁速度快就用谁

##### 对计算结果进行合并

- thenCombine 合并前一个和当前数据



 



 
