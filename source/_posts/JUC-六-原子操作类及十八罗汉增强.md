---
title: JUC(六)-原子操作类及十八罗汉增强
index_img: /img/default.png
banner_img: /img/default.png
date: 2022-06-19 14:21:57
tags:
categories:
description:
---

理解atomic原子类的基本类型原子类、数组类型原子类、引用类型原子类、对象的属性修改原子类的概要思想与案例分析

<!-- more -->

## 基本类型原子类

### AtomicInteger

```java
public final int get()：获取当前的值
public final int getAndSet(int newValue)：获取当前的值，并设置新的值
public final int getAndIncrement()：获取当前的值，并自增(类似i++)
public final int incrementAndGet()：自增后再获取当前的值(类似++i)
public final int getAndDecrement()：获取当前的值，并自减
public final int getAndAdd(int delta)：获取当前的值，并加上预期的值
void lazySet(int newValue): 最终会设置成newValue,使用lazySet设置值后，可能导致其他线程在之后的一小段时间内还是可以读到旧的值。
```

```java
package cn.hyqup.juc;

import java.util.concurrent.CountDownLatch;
import java.util.concurrent.atomic.AtomicInteger;

/**
 * Copyright © 2022 灼华. All rights reserved.
 *
 * @author create by hyq
 * @version 0.1
 * @date 2022/6/21
 * @description:
 */
class MyNumber {

    private AtomicInteger atomicInteger = new AtomicInteger();

    public void addPlusPlus() {
        atomicInteger.incrementAndGet();
    }

    public AtomicInteger getAtomicInteger() {
        return atomicInteger;
    }

    public void setAtomicInteger(AtomicInteger atomicInteger) {
        this.atomicInteger = atomicInteger;
    }
}

public class AtomicIntegerDemo {
    public static void main(String[] args) throws InterruptedException {

        MyNumber myNumber = new MyNumber();
        CountDownLatch countDownLatch = new CountDownLatch(100);

        for (int i = 1; i <= 100; i++) {
            new Thread(() -> {
                try {
                    for (int j = 1; j <= 5000; j++) {
                        myNumber.addPlusPlus();
                    }
                } finally {
                    countDownLatch.countDown();
                }
            }, String.valueOf(i)).start();
        }

        countDownLatch.await();

        System.out.println(myNumber.getAtomicInteger().get());

    }

}
```

#### 扩展知识CountDownLatch

它是一个同步工具类，允许一个或多个线程一直等待，直到其他线程运行完成后再执行。

使用场景：

- 场景1：让多个线程等待

    多个线程awit()的时候，等待某个线程调用countDown来触发

  ```java
  CountDownLatch countDownLatch = new CountDownLatch(1);
  for (int i = 0; i < 5; i++) {
      new Thread(() -> {
          try {
              //准备完毕……运动员都阻塞在这，等待号令
              countDownLatch.await();
              String parter = "【" + Thread.currentThread().getName() + "】";
              System.out.println(parter + "开始执行……");
          } catch (InterruptedException e) {
              e.printStackTrace();
          }
      }).start();
  }
  
  Thread.sleep(2000);// 裁判准备发令
  countDownLatch.countDown();// 发令枪：执行发令
  ```

- 场景2：和让单个线程等待

  ​	多线程执行任务的同时，主线程等待多线程执行完成后，再在主线程得到数据，上面的AtomicIntegerDemo就是这种场景

### AtomicBoolean

原子更新布尔类型，用法类似

### AtomicLong

原子更新长整型，用法类似

## 原子更新数组

### AtomicIntegerArray

原子更新整型数组里的元素。

### AtomicLongArray

原子更新长整型数组里的元素。

### AtomicReferenceArray

原子更新引用类型数组里的元素。

## 引用类型原子类

> 上一章节讲到的ABA问题，这里就使用AtomicStampedReference和AtomicMarkableReference解决

### AtomicReference

原子更新引用类型,通过CAS原理调用compareAndSet

### AtomicStampedReference

原子更新引用类型, 内部使用Pair来存储元素值及其版本号，携带`版本号`的引用类型原子类，解决修改过几次

```java
package cn.hyqup.juc;

import java.util.concurrent.atomic.AtomicStampedReference;

/**
 * Copyright © 2022 灼华. All rights reserved.
 *
 * @author create by hyq
 * @version 0.1
 * @date 2022/6/21
 * @description:
 */
public class AtomicStampedReferenceDemo {
    static AtomicStampedReference atomicStampedReference = new AtomicStampedReference(1, 0);

    public static void main(String[] args) {
        new Thread(() -> {
            System.out.println("操作线程" + Thread.currentThread() + ",初始值 a = " + atomicStampedReference.getReference());
            int stamp = atomicStampedReference.getStamp(); //获取当前标识别
            try {
                Thread.sleep(1000); //等待1秒 ，以便让干扰线程执行
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            boolean isCASSuccess = atomicStampedReference.compareAndSet(1, 2, stamp, stamp + 1);  //此时expectedReference未发生改变，但是stamp已经被修改了,所以CAS失败
            System.out.println("操作线程" + Thread.currentThread() + ",CAS操作结果: " + isCASSuccess);
        }, "t1").start();

        new Thread(() -> {
            Thread.yield(); // 确保thread-main 优先执行
            atomicStampedReference.compareAndSet(1, 2, atomicStampedReference.getStamp(), atomicStampedReference.getStamp() + 1);
            System.out.println("操作线程" + Thread.currentThread() + ",【increment】 ,值 = " + atomicStampedReference.getReference());
            atomicStampedReference.compareAndSet(2, 1, atomicStampedReference.getStamp(), atomicStampedReference.getStamp() + 1);
            System.out.println("操作线程" + Thread.currentThread() + ",【decrement】 ,值 = " + atomicStampedReference.getReference());
        }, "t2").start();
    }


}
```

### AtomicMarkableReference

原子更新带有标记位的引用类型，原子更新带有`标记位`的引用类型对象，解决是否修改过

## 对象的属性修改原子类

> 以一种线程安全的方式操作非线程安全对象内的某些字段

### 使用要求(重点)

1、更新的对象属性必须使用 public `volatile` 修饰符

2、因为对象的属性修改类型原子类都是抽象类，所以每次使用都必须
使用静态方法newUpdater()创建一个更新器，并且需要设置想要更新的类和属性。

### AtomicIntegerFieldUpdater

```java
package cn.hyqup.juc;

import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicIntegerFieldUpdater;

/**
 * Copyright © 2022 灼华. All rights reserved.
 *
 * @author create by hyq
 * @version 0.1
 * @date 2022/6/21
 * @description:
 */
class BankAccount
{
    private String bankName = "ZG";
    public volatile int money = 0;
    AtomicIntegerFieldUpdater<BankAccount> accountAtomicIntegerFieldUpdater = AtomicIntegerFieldUpdater.newUpdater(BankAccount.class,"money");

    //不加锁+性能高，局部微创
    public void transferMoney(BankAccount bankAccount)
    {
        accountAtomicIntegerFieldUpdater.incrementAndGet(bankAccount);
    }
}

public class AtomicIntegerFieldUpdaterDemo {

    public static void main(String[] args)
    {
        BankAccount bankAccount = new BankAccount();

        for (int i = 1; i <=1000; i++) {
            int finalI = i;
            new Thread(() -> {
                bankAccount.transferMoney(bankAccount);
            },String.valueOf(i)).start();
        }

        //暂停毫秒
        try { TimeUnit.MILLISECONDS.sleep(500); } catch (InterruptedException e) { e.printStackTrace(); }

        System.out.println(bankAccount.money);

    }

}
```

### AtomicLongFieldUpdater

### AtomicReferenceFieldUpdater

```java
package cn.hyqup.juc;

import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicReferenceFieldUpdater;

/**
 * Copyright © 2022 灼华. All rights reserved.
 *
 * @author create by hyq
 * @version 0.1
 * @date 2022/6/21
 * @description:
 */
class MyVar
{
    public volatile Boolean isInit = Boolean.FALSE;
    AtomicReferenceFieldUpdater<MyVar,Boolean> atomicReferenceFieldUpdater = AtomicReferenceFieldUpdater.newUpdater(MyVar.class,Boolean.class,"isInit");


    public void init(MyVar myVar)
    {
        if(atomicReferenceFieldUpdater.compareAndSet(myVar,Boolean.FALSE,Boolean.TRUE))
        {
            System.out.println(Thread.currentThread().getName()+"\t"+"---init.....");
            //暂停几秒钟线程
            try { TimeUnit.SECONDS.sleep(2); } catch (InterruptedException e) { e.printStackTrace(); }
            System.out.println(Thread.currentThread().getName()+"\t"+"---init.....over");
        }else{
            System.out.println(Thread.currentThread().getName()+"\t"+"------其它线程正在初始化");
        }
    }


}
public class AtomicReferenceFieldUpdaterDemo {
    public static void main(String[] args) {
        MyVar myVar = new MyVar();

        for (int i = 1; i <=5; i++) {
            new Thread(() -> {
                myVar.init(myVar);
            },String.valueOf(i)).start();
        }
    }

}
```

