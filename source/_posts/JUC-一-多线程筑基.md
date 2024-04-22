---
title: JUC(一)-多线程筑基
index_img: https://tva4.sinaimg.cn/large/9bd9b167gy1g2rkyl0xa2j21hc0u0tk8.jpg
banner_img: https://tva2.sinaimg.cn/large/9bd9b167gy1g2rko7x9qsj21hc0u0dlf.jpg
abbrlink: e2166019
date: 2022-06-14 23:14:08
tags:
categories:
 - Java
 - JUC
description:
---

 回顾复习一下多线程基础知识，为JUC筑基

<!-- more -->

## 实现多线程的方式

- 继承Thread

  run()调用：主线程调用run()的时候，整个流程只有主线程一条执行路径，主线程run执行后再串行执行run方法体的内容

  start()调用：调用start(),会由子线程执行子线程的run()，主线程和子线程交替执行

​		run和start的区别：

​				1、start是启动一个新的线程

​				2、run()方法内的内容是执行的主体，是执行内容的入口方法

​				3、start()方法只能调用一次，run()方法调用没有限制	

​				4、start()方法不会阻塞主线程，run会阻塞调用线程

- 实现Runnable
- 实现Callable

## 线程状态

![线程状态流转图](http://hyqup-blog-upyun.test.upcdn.net/img/1775037-20191112153234240-689002981.png)

一个线程从创建到消亡会经历新建状态（New）、就绪状态（Runnable）、运行状态（Running）、等待（Waiting）、阻塞状态（Blocked）和死亡状态

- 新建状态（New）： 新创建了一个线程对象，还未调用线程的start()方法。
- 就绪状态（Runnable）： 线程对象创建后，其他线程调用了该对象的start()方法，该状态的线程位于可运行线程池中，变得可运行，等待获取CPU的使用权。
- 运行状态（Running）：就绪状态的线程获取了CPU，执行程序代码的状态，还有种可能就是这个线程正在等待其他的系统资源（IO资源等），这种状态也称为Running状态。**需要注意的是，通过Thread::getStatus查看状态时，Runnable状态和Running统一称为Runnable状态。**
- 阻塞状态（Blocked）：一个线程因为等待监视锁而被阻塞的状态，也称之为阻塞状态。**阻塞的线程不会被分配CPU资源**。
- 等待状态（WAITING）：一个正在等待的线程的状态，也称之为等待状态。造成线程等待的原因有三种，分别是调用Object.wait()、join()以及LockSupport.park()方法。处于等待状态的线程，正在等待其他线程去执行一个特定的操作。例如：因为wait()而等待的线程正在等待另一个线程去调用notify()或notifyAll()；一个因为join()而等待的线程正在等待另一个线程结束。**处于等待状态的线程不会被分配CPU资源**。
- 超时等待（TIMED_WAITING）：一个在限定时间内等待的线程的状态。也称之为限时等待状态。造成线程限时等待状态的原因有五种，分别是：Thread.sleep(long)、Object.wait(long)、join(long)、LockSupport.parkNanos(obj,long)和LockSupport.parkUntil(obj,long)。
- 死亡状态（Dead）：线程执行完了或者因异常退出了run()方法，该线程结束生命周期(当时如果线程被持久持有, 可能不会被回收)。

### 三种阻塞状态

**1. 超时等待状态（TIMED_WAITING）**
Java文档官方定义TIMED_WAITING状态为：“一个线程在一个特定的等待时间内等待另一个线程完成一个动作会在这个状态”。调用下面的这些方法会让线程进入TIMED_WAITING状态。

- Thread#sleep()；
- Object#wait() 并加了超时参数；
- Thread#join() 并加了超时参数；
- LockSupport#parkNanos()；
- LockSupport#parkUntil()。

**2. 等待状态（WAITING）**
Java文档官方定义WAITING状态是：“一个线程在等待另一个线程执行一个动作时在这个状态。”

当线程调用以下方法时会进入WAITING状态：

- Object#wait() 而且不加超时参数
- Thread#join() 而且不加超时参数
- LockSupport#park()。

在对象上的线程调用了Object.wait()会进入WAITING状态，直到另一个线程在这个对象上调用了Object.notify()或Object.notifyAll()方法才能恢复。一个调用了Thread.join()的线程会进入WAITING状态直到一个特定的线程来结束。

**3. BLOCKED状态**
Java文档官方定义BLOCKED状态是：“这种状态是指一个阻塞线程在等待monitor锁。”

### sleep、yield、wait、join的区别

#### 简单使用

Thread.sleep(long )，线程休眠

obj.wait() 线程等待

Thread.yield() 让出cpu调度，重新竞争资源

线程对象obj.join(),A线程对象中调用B线程join方法，此时A线程进入等待，B线程开始执行，执行完了A线程继续执行

详解 https://www.cnblogs.com/aspirant/p/8876670.html

### 用戶线程和守护线程

> 用户线程(普通线程)、守护线程(后台线程)

用户线程：

 守护线程 Daemon：程序运行的时候在后台提供一种通用服务的线程，比如垃圾回收线程

当所有的非守护线程结束时，程序也就终止了，同时会杀死进程中的所有守护线程。

通过setDaemon(true) 将普通线程变为守护线程

## 线程同步

并发：一个对象被多个线程同时操作

使用队列+锁机制解决

java的wait/notify的通知机制可以用来实现线程间通信。wait表示线程的等待，调用该方法会导致线程阻塞，直至另一线程调用notify或notifyAll方法才可另其继续执行。经典的生产者、消费者模式即是使用wait/notify机制得以完成

## 线程的启动原理

从一个线程的strat启动开始，

线程的启动，JNI底层调用C的代码，进一步调用OS来操作进程线程相关

```java
public class DaemonDemo {
    public static void main(String[] args) {
        new  Thread(()->{
            System.out.println("哈哈哈");
        }).start();
    }

}
```

```java
public synchronized void start() {
    /**
     * This method is not invoked for the main method thread or "system"
     * group threads created/set up by the VM. Any new functionality added
     * to this method in the future may have to also be added to the VM.
     *
     * A zero status value corresponds to state "NEW".
     */
    if (threadStatus != 0)
        throw new IllegalThreadStateException();

    /* Notify the group that this thread is about to be started
     * so that it can be added to the group's list of threads
     * and the group's unstarted count can be decremented. */
    group.add(this);

    boolean started = false;
    try {
        start0();
        started = true;
    } finally {
        try {
            if (!started) {
                group.threadStartFailed(this);
            }
        } catch (Throwable ignore) {
            /* do nothing. If start0 threw a Throwable then
              it will be passed up the call stack */
        }
    }
}
```



start0 方法是native方法，是jvm底层通过C/C++调用操作系统的方法

