---
title: Java并发多线程与高并发(四)-安全发布对象
index_img: https://file.hyqup.cn/img/eb8bb9bc5bfd4b54ad376cd8c3fedc3f.jpg
banner_img: https://file.hyqup.cn/img/eb8bb9bc5bfd4b54ad376cd8c3fedc3f.jpg
abbrlink: 5839849d
date: 2021-04-28 23:40:36
tags:
- 单例模式
- 对象发布
- 对象溢出
categories:
- 并发
- 多线程
description:
---

安全发布对象，讲述在编码中我们对公共资源正确发布来保证线程安全，且简述使用几种单例模式来发布对象

<!-- more -->

##  概念

**发布对象：**是一个对象能够被当前范围之外的代码嗦使用

**对象溢出：**一种错误的发布。当一个对象还没有构造完成时，就使它被其它线程所见

## 四种安全的发布对象

- 在静态初始化函数中初始化一个对象引用 (JVM 在静态初始化的时候会保证线程安全)
- 将对象的引用保存到 volatile 类型的域或者 AtomicReferance 中
- 将对象的引用保存到某个正确构造对象的的 final 类型域中
- 将对象的引用保存到一个由锁保护的域中

## 线程安全的单例模式的写法

### 饿汉式（类加载的时候就创建好）

```java
package cn.hyqup.concurrency.singleton;

/**
 * Copyright © 2021灼华. All rights reserved.
 *
 * @author create by hyq
 * @version 1.0
 * @date 2021/5/13
 * @description: 饿汉式（类加载的时候就创建好）
 */
public class SingletonExample1 {
    //   私有构造函数
    private SingletonExample1() {
    }

    private static SingletonExample1 instance = new SingletonExample1();

    public static SingletonExample1 getInstance() {
        return instance;
    }
}

```

备注：饿汉模式还可以通过静态块来初始化，**需要注意静态域和静态代码块的顺序**（决定先加载对象还是先加载静态代码块初始对象，如果静态代码块在前就会出现空指针），这里就不演示了

 ### 懒汉式（需要的时候才去创建）【不推荐的写法】

```java
package cn.hyqup.concurrency.singleton;

/**
 * Copyright © 2021灼华. All rights reserved.
 *
 * @author create by hyq
 * @version 1.0
 * @date 2021/5/13
 * @description: 懒汉式（需要的时候才去创建）【不推荐的写法】
 */
public class SingletonExample2 {
    //  私有构造函数
    private SingletonExample2() {
    }

    private static SingletonExample2 instance = null;

    /**
     * 统一时间只有一个线程访问,虽然是线程安全的,但不推荐
     * 原因：加锁在方法上容易造成资源浪费
     * @return
     */
    public static synchronized SingletonExample2 getInstance() {
        if (instance == null) {
            instance = new SingletonExample2();
        }
        return instance;
    }
}

```



### 懒汉式（需要的时候才去创建）-【推荐的写法】

```java
package cn.hyqup.concurrency.singleton;

/**
 * Copyright © 2021灼华. All rights reserved.
 *
 * @author create by hyq
 * @version 1.0
 * @date 2021/5/13
 * @description: 懒汉式（需要的时候才去创建）-【推荐的写法】
 */
public class SingletonExample3 {
    //  私有构造函数
    private SingletonExample3() {
    }
    //    volatile+双重检测机制=》禁止指令重排序
    private volatile static SingletonExample3 instance = null;

    /**
     * 1、双重检测机制
     * 2、同步锁
     * 3、禁止指令重排
     *
     * @return
     */
    public static SingletonExample3 getInstance() {
        if (instance == null) {
            synchronized (SingletonExample3.class) {
                if (instance == null) {
                    instance = new SingletonExample3();
                }
            }
        }
        return instance;
    }
}

```

#### 为什么需要使用volatile

// instance = new SingletonExample3();

cpu的指令，执行new对象的时候分为三步

1. memory=allocate() 分配对象内存空间
2. ctorInstance() 初始化对象
3. instance=memory 设置instance指向刚分配的内存

多线程情况下会出问题

JVM和CPU 发生指令重排 变成1  3 2，因为3和2的顺序不重要

解决办法：使用volatile 禁止指令重排

### 枚举模式：最安全

```java
package cn.hyqup.concurrency.singleton;

/**
 * Copyright © 2021灼华. All rights reserved.
 *
 * @author create by hyq
 * @version 1.0
 * @date 2021/5/13
 * @description: 枚举单例（JVM 保证）-【推荐的写法】
 */
public class SingletonExample4 {
    //  私有构造函数
    private SingletonExample4() {
    }

    /**
     * 1、既能保证整个对象初始化一次
     * 2、又能保证在使用的使用的时候才去初始化
     *
     * @return
     */
    public static SingletonExample4 getInstance() {
        return Singleton.INSTANCE.getInstance();
    }

    private enum Singleton {
        INSTANCE;
        private SingletonExample4 singleton;

        Singleton() {
            singleton = new SingletonExample4();
        }

        public SingletonExample4 getInstance() {
            return singleton;
        }

    }
}

```



