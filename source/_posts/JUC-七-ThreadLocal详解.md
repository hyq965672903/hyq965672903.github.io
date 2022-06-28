---
title: JUC(七)-ThreadLocal详解
index_img: /img/default.png
banner_img: /img/default.png
abbrlink: 6f92e213
date: 2022-06-28 14:46:38
tags:
categories:
description:
---

学习并掌握ThreadLocal的使用，使用场景，原理。以及相关的强、软、弱、虚引用，ThreadLocal内存泄露问题剖析以及解决办法

<!-- more -->

## ThreadLocal简介

ThreadLocal又叫做线程局部变量，ThreadLocal为变量在每个线程中都创建了一个副本，那么每个线程可以访问自己内部的副本变量。

　**内存溢出（memory overflow）**：是指不能申请到足够的内存进行使用，就会发生内存溢出，比如出现的OOM（Out Of Memory）

　**内存泄漏（memory lack）**：内存泄露是指在程序中已经动态分配的堆内存由于某种原因未释放或者无法释放（已经没有用处了，但是没有释放），造成系统内存的浪费，这种现象叫“内存泄露”。

　　当内存泄露到达一定规模后，造成系统能申请的内存较少，甚至无法申请内存，最终导致内存溢出，所以内存泄露是导致内存溢出的一个原因。

## 如何创建

泛型可以是任意，这里以String 为例

第一种(伪代码)：set方法,`当前线程赋值值，当前线程取值`

```
private static final ThreadLocal<String> threadLocal = new ThreadLocal<String>(){};

// 在方法中调用set赋值
threadLocal.set("xxx")
```

第二种（伪代码）：使用withInitial，`统一初始化所有线程的ThreadLocal的值`

```java
private ThreadLocal<Integer> threadLocal = ThreadLocal.withInitial(() -> 6);
```

## Thread、ThreadLocal和ThreadLocalMap三者之间的关系

- Thread

​		线程类，在这个类中存在一个`threadLocals`变量

```
ThreadLocal.ThreadLocalMap threadLocals = null;
```

- ThreadLocal

​		此类提供了一个简单的`set`,`get`,`remove`方法，用于设置，获取或移除 绑定到线程本地变量中的值。里面存一个**匿名静态内部类**ThreadLocalMap

- ThreadLocalMap

​		这是在ThreadLocal中定义的一个类，可以简单的将它理解成一个Map，不过它的key是`WeakReference弱引用`类型，这样当这个值没有在别的地方引用时，在发生垃圾回收时，这个map的`key`会被自动回收，不过它的值不会被自动回收。

## 源码分析 get和set

### get实现

```
public T get() {
    // 获取当前线程
    Thread t = Thread.currentThread();
    // 获取这个线程自身绑定的 ThreadLocalMap 对象
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        // this是ThreadLocal对象，获取Map中的Entry对象
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings("unchecked")
            // 获取具体的值
            T result = (T)e.value;
            return result;
        }
    }
    // 设置初始值
    return setInitialValue();
}
```

### set实现

```java
public void set(T value) {
    // 获取当前线程
    Thread t = Thread.currentThread();
    // 获取绑定到这个线程自身的 ThreadLocalMap，这个ThreadLocalMap是从Thread类的`threadLocals`变量中获取的
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        // 向map中设置值，key为 ThreadLocal 对象的实例。
        map.set(this, value);
    } else {
        // 如果map不存在，则创建出来。
        createMap(t, value);
    }
}
```

1. 获取当前线程`Thread`
2. 获取当前线程的 `ThreadLocalMap` 对象。
3. 向`ThreadLocalMap`中设置值，key为`ThreadLocal`对象，值为具体的值。

## 强引用、软引用、弱引用和虚引用是什么？

> 强度由高到低依次为：强引用 -> 软引用 -> 弱引用 -> 虚引用

- 强引用(StrongReference)

  **强引用**是使用最普遍的引用。如果一个对象具有强引用，那**垃圾回收器**绝不会回收它

  如:Object strongReference = new Object();

​		当**内存空间不足**时，`Java`虚拟机宁愿抛出`OutOfMemoryError`错误，使程序**异常终止**，也不会靠随意**回收**具有**强引用**的**对象**来解决内存不足的问题。显式地设置`strongReference`对象为`null`，或让其**超出**对象的**生命周期**范围，则`gc`认为该对象**不存在引用**，这时就可以回收这个对象。具体什么时候收集这要取决于`GC`算法

- 软引用(SoftReference)

​		如果一个对象只具有**软引用**，则**内存空间充足**时，**垃圾回收器**就**不会**回收它；如果**内存空间不足**了，就会**回收**这些对象的内存。只要垃圾回收器没有回收它，该对象就可以被程序使用。

- 弱引用(WeakReference)

  在垃圾回收器线程扫描它所管辖的内存区域的过程中，一旦发现了只具有**弱引用**的对象，不管当前**内存空间足够与否**，都会**回收**它的内存。

- 虚引用(PhantomReference)

  **虚引用**顾名思义，就是**形同虚设**。与其他几种引用都不同，**虚引用**并**不会**决定对象的**生命周期**。如果一个对象**仅持有虚引用**，那么它就和**没有任何引用**一样，在任何时候都可能被垃圾回收器回收。

ThreadLocal内部类 ThreadLocalMap采用**弱引用**

```java
static class Entry extends WeakReference<ThreadLocal<?>> {
    /** The value associated with this ThreadLocal. */
    Object value;

    Entry(ThreadLocal<?> k, Object v) {
        super(k);
        value = v;
    }
}
```

## ThreadLocal内存泄露原理分析，如何解决？

### 原因分析：

ThreadLocalMap的每个Entry 都是一个对key的弱引用，同时，每个Entry 都包含了一个对value的强引用。正常情况在多线程中，线程终止后，保存在ThreadLocal里的value会被垃圾回收，因为没有任何强引用了

但是在使用线程池的情况下，线程会复用，不会立即终止，这个时候即便是key弱引，被回收，但是key对应的value就不能被回收，因为value和Thread之间还存在这个强引用链路，所以导致value无法回收，就可能会出现OOM

### 如何解决

JDK已经考虑到了这个问题，所以在set，remove，rehash方法中会扫描key为null的Entry，并把对应的value设置为null，这样value对象就可以被回收，但是如果一个**ThreadLocal不被使用**，那么实际上set，remove，rehash方法也不会被调用，如果同时线程又不停止，那么调用链就一直存在，那么就导致了value的内存泄漏

在使用完ThreadLocal之后，**主动调用remove**方法

## ThreadLocal为什么采用弱引用？



## 使用场景

ThreadLocal 适用于每个线程需要自己独立的实例且该实例需要在多个方法中被使用，也即变量在线程间隔离而在方法或类间共享的场景
