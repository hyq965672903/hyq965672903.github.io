---
title: JUC(三)-Java的“锁”事
index_img: https://tva1.sinaimg.cn/large/9bd9b167gy1g2rkqolw28j21hc0u0wmp.jpg
banner_img: https://tva2.sinaimg.cn/large/9bd9b167gy1g2rkqrxy1xj21hc0u0qb1.jpg
abbrlink: f60080de
date: 2022-06-18 11:51:04
tags:
categories:
description:
---

 Java的锁通过悲观乐观，由浅入深，循序渐进，

<!-- more -->

## 悲观锁

总是假设最坏的情况，每次取数据时都认为其他线程会修改，所以都会加（悲观）锁。一旦加锁，不同线程同时执行时,只能有一个线程执行，其他的线程在入口处等待，直到锁被释放。

应用场景：	

- `MySQL`的读锁、写锁、行锁等
- `Java`的`synchronized`关键字
- **Synchronized**
- **ReentrantLock**

## 乐观锁

它总认为资源和数据不会被别人所修改，所以读取不会上锁，但是乐观锁在进行写入操作的时候会判断当前数据是否被修改过，乐观锁的实现方案一般来说有两种：`版本号机制` 和 `CAS实现`

应用场景：

AtomicIntegter

## Synchronized详解

### 锁在普通方法上（Synchronized修饰普通方法）

> 对象锁

- 一个对象里面如果有多个**Synchronized方法**，同一时刻内，只要一个线程去调用其中的一个Synchronized方法，其他线程只能等待。**锁的是当前对象this,被锁定后其他线程都不能进入当前对象的其他Synchronized方法**
- 普通方法和锁无关，不影响访问
- 换成不同的对象后，就不是同一把锁了，不同对象之间不受影响

### 锁在static方法上（Synchronized修饰static方法）

> 类锁

- 类锁后，也就是锁定了当前的Class对象。即使创建了不同对象，也是谁先获得类锁谁先执行

### 总结

 对于普通同步方法，锁的是当前实例对象，通常指this

对于静态同步方法，锁的是当前类的class对象

对于同步方法快，锁的是**Synchronized**括号内的对象

## 公平锁和非公平锁

example: 

ReentrantLock lock = new ReentrantLock();  //默认非公平锁

ReentrantLock lock = new ReentrantLock(true);  //公平锁

### 公平锁：

​				多个线程按照申请锁的顺序去获得锁，线程会直接进入队列去排队，永远都是队列的第一位才能得到锁



### 非公平锁：

​					多个线程去获取锁的时候，会直接去尝试获取，获取不到，再去进入等待队列，如果能获取到，就直接获取到锁。

以ReentrantLock 源码为例

非公平锁的 tryAcquire

```java
final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        if (compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0) // overflow
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```



公平锁的tryAcquire

```java
protected final boolean tryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        if (!hasQueuedPredecessors() &&
            compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0)
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```

**hasQueuedPredecessors** 是唯一的区别，会判断前面还有前驱节点

### 为什么有默认设计非公平锁？

1、线程的恢复挂起还是需要时间的，非公平锁就能充分利用cpu时间片，减少cpu空闲时间

2、由于一个线程在获取同步状态，再释放同步状态，非同步锁不需要考虑前驱节点，此时刚刚释放锁的线程获取同步状态的几率相对于其他线程就会非常高，减少线程的开销

### 公平锁有什么问题？

公平锁充分保证了公平性，非公平锁可能导致排队的一直在排队，导致“锁饥饿”问题

### 公平锁和非公平锁的场景？

 需要更高的效率则用非公平锁，节省了线程切换的时间，否则就使用公平锁，大家公平使用

## 可重入锁（递归锁）

> Java中 synchronized和ReentrantLock都是可重入锁

 是指同一个线程在外层方法获取锁的时候，再进入该线程的内层方法会自动获取锁(前提是锁对象得是同一个对象)，不会因为之前已经获取过锁还没有释放而阻塞

简单来说就是外层调用里面的不会因为外面的锁没有释放而造成死锁

synchronized 可重复锁演示：

```java
package cn.hyqup.juc;

/**
 * Copyright © 2022 灼华. All rights reserved.
 *
 * @author create by hyq
 * @version 0.1
 * @date 2022/6/18
 * @description: 可重复锁Demo
 */
class WashClothes {

    /**
     * 放水
     */
    public static synchronized void water() {
        System.out.println("放水");
        wash();
    }

    /**
     * 洗衣服
     */
    public static synchronized void wash() {
        System.out.println("洗衣服");
    }
}

public class ReEntryLockDemo {
    public static void main(String[] args) {
        WashClothes washClothes = new WashClothes();
        new Thread(() -> {
            washClothes.water();
        }).start();

    }
}
```

ReentrantLock也是支持可重入锁的

> Lock.lock()几次 ，对应就需要Lock.unlock()几次，否则 当前线程虽然看起来程序运行没问题，但是其他线程对当前对象锁由于没有释放永远拿不到

```java
package cn.hyqup.juc;

import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

/**
 * Copyright © 2022 灼华. All rights reserved.
 *
 * @author create by hyq
 * @version 0.1
 * @date 2022/6/18
 * @description: 可重复锁Demo
 */


public class ReEntryLockDemo2 {
    public static void main(String[] args) {
        Lock lock = new ReentrantLock();
        new Thread(() -> {
            lock.lock();
            try {
                System.out.println("外部");
                lock.lock();
                try {
                    System.out.println("内部");
                } finally {
                    lock.unlock();
                }
            } finally {
                lock.unlock();
            }
        }).start();

    }
}
```

## 死锁

> 是指两个或两个以上的进程在执行过程中，因争夺资源而造成的一种互相等待的现象，若无外力作用，它们都将无法推进下去。

```java
package cn.hyqup.juc;

import java.util.concurrent.TimeUnit;

/**
 * Copyright © 2022 灼华. All rights reserved.
 *
 * @author create by hyq
 * @version 0.1
 * @date 2022/6/18
 * @description:
 */
public class DeadLockDemo {

    static Object lockA = new Object();
    static Object lockB = new Object();

    public static void main(String[] args) {
        new Thread(() -> {
            synchronized (lockA) {
                System.out.println("持有A锁");
                try {
                    TimeUnit.SECONDS.sleep(1);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                synchronized (lockB) {
                    System.out.println("获得B锁");
                }
            }
        }, "a").start();
        new Thread(() -> {
            synchronized (lockB) {
                System.out.println("持有B锁");
                try {
                    TimeUnit.SECONDS.sleep(1);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                synchronized (lockA) {
                    System.out.println("获得A锁");
                }
            }
        }, "B").start();
    }

}
```

A,B锁资源相互竞争，造成死锁

### 死锁排查

jps -l   ：查找java相关的后台进程

jstack 进程号 ：如果有死锁 会打印 FInd Dead Lock

或者采用 jconsole 检查死锁

## 多线程的中断机制

首先，一个线程的停止应当自己决定自然死亡，而不应该由其他线程去stop,这也是jdk resume,stop方法废弃的原因

但实际生产中可能会由于一些原因去停止一些耗时操作，java提供了停止线程的机制------**中断协商机制**

中断协商机制并不是立刻停止一个线程，interrupt将线程对象的**中断标识设置为true**

 ### 如何优雅的中断一个线程

 - 通过一个volatile变量实现。可见性机制设立标志。

 - 通过AtomicBoolean变量实现， 类似，也是标志位方式来退出

 - 使用Thread.interrupt来设置中断标志位为true,结合isInterrupted使用

   注意：1、interrupt只是设置中断标识位，不会立即停止线程

   ​			2、线程死亡isInterrupted，也是返回false

   ​			3、**如果线程处理被阻塞状态（sleep，wait , join等状态），在别的线程中调用当前线程的interrupt，会立即退出被阻塞状态，将中断标志位清除，设置为false,并抛出interruptException**  

   解决办法：捕获interruptException，再执行一次interrupt，将中断标志位设置为true

Thread.interrupted()返回线程的线程中断状态，并清除。类似于i++

## 线程的等待与唤醒

### 概述

- 方式一：使用Object的wait()方法让线程等待，使用Object中的notify() 方法唤醒线程
- 方式二：使用JUC包中的Condition的await()方法让线程等待，使用signal()方法唤醒线程
- 方式三：LockSupport类可以park等待和unpark唤醒

### wait和notify

```java
package cn.hyqup.juc;

import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

/**
 * Copyright © 2022 灼华. All rights reserved.
 *
 * @author create by hyq
 * @version 0.1
 * @date 2022/6/18
 * @description:
 */
public class LockSupportDemo {
    static Object objectLock = new Object();
    static Lock lock = new ReentrantLock();
    static Condition condition = lock.newCondition();

    public static void main(String[] args) {
        new Thread(() -> {
            synchronized (objectLock) {
                System.out.println(Thread.currentThread().getName() + "进入");
                try {
                    objectLock.wait();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println(Thread.currentThread().getName() + "被唤醒");
            }
        }, "t1").start();
        try { 
            TimeUnit.SECONDS.sleep(2);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        new Thread(() -> {
            synchronized (objectLock) {
                objectLock.notify();
                System.out.println(Thread.currentThread().getName() + "发出通知");

            }
        }, "t2").start();
    }

}
```

#### 总结：

1、wait notify  synchronized 要一起使用，如果wait没在synchronized中包着使用，会抛出异常

2、wait和notify顺序不能变，否则会不起作用

### Condition

```java
package cn.hyqup.juc;

import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

/**
 * Copyright © 2022 灼华. All rights reserved.
 *
 * @author create by hyq
 * @version 0.1
 * @date 2022/6/18
 * @description:
 */
public class LockSupportDemo2 {
    static Object objectLock = new Object();
    static Lock lock = new ReentrantLock();
    static Condition condition = lock.newCondition();

    public static void main(String[] args) {
        new Thread(() -> {
            lock.lock();
            try {
                System.out.println(Thread.currentThread().getName() + "进入");
                condition.await();
                System.out.println(Thread.currentThread().getName() + "被唤醒");
            } catch (InterruptedException e) {
                e.printStackTrace();
            } finally {
                lock.unlock();
            }

        }, "t1").start();
        try {
            TimeUnit.SECONDS.sleep(2);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        new Thread(() -> {
            lock.lock();
            try {
                condition.signal();
                System.out.println(Thread.currentThread().getName() + "发出通知");
            } finally {
                lock.unlock();
            }

        }, "t2").start();
    }

}
```

#### 总结：

1、condition必须包裹在lock和unlock之前

2、顺序性一定保证await后signal，否则无效

### LockSupport

```java
package cn.hyqup.juc;

import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.LockSupport;

/**
 * Copyright © 2022 灼华. All rights reserved.
 *
 * @author create by hyq
 * @version 0.1
 * @date 2022/6/18
 * @description:
 */
public class LockSupportDemo3 {
    public static void main(String[] args) {
        Thread t1 = new Thread(() -> {
            System.out.println(Thread.currentThread().getName() + "进入");
            LockSupport.park();
            System.out.println(Thread.currentThread().getName() + "被唤醒");
        }, "t1");
        t1.start();

        try {
            TimeUnit.SECONDS.sleep(2);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        new Thread(() -> {
            LockSupport.unpark(t1);
            System.out.println(Thread.currentThread().getName() + "发出通知");
        }, "t2").start();
    }

}
```

总结：

不用包在锁块，减低死锁发生的可能性

不用保证顺序

**注意：unpark只有一次上限为1 如果多次unpark，最多只为1，因为上限为1，此时如果park多次则无法执行后续代码(at most 1,at least 0)**
