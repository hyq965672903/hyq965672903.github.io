---
title: JUC(八)-AbstractQueuedSynchronizer之AQS
index_img: https://tva1.sinaimg.cn/large/9bd9b167gy1g2rmkj1qn2j21hc0u0b29.jpg
banner_img: https://tva2.sinaimg.cn/large/9bd9b167ly1g2rl02t1gpj21hc0u0am3.jpg
abbrlink: e0cf2292
date: 2022-07-07 20:38:51
tags:
categories:
 - Java
 - JUC
description:
---

`AQS`即是抽象的队列式的同步器，熟知的`ReentrantLock`、`ReentrantReadWriteLock`、`CountDownLatch`、`Semaphore`等都是基于`AQS`来实现的。

<!-- more -->

## 基本原理

> 是用来构建锁或者其它同步器组件的重量级基础框架及整个JUC体系的基石，通过内置的FIFO队列来完成资源获取线程的排队工作，并通过一个int类变量表示持有锁的状态

![image.png](https://file.hyqup.cn/img/171d2d42c77b2765tplv-t2oaga2asx-image.image)

<center>图片来自网络</center>

CLH:Craig、Landin and Hagersten 队列，是一个单向链表，AQS中的队列是CLH变体的虚拟双向队列FIFO

`AQS`中 维护了一个`volatile int state`（代表共享资源）和一个`FIFO`线程等待队列（多线程争用资源被阻塞时会进入此队列）

这里`volatile`能够保证多线程下的可见性，当`state=1`则代表当前对象锁已经被占有，其他线程来加锁时则会失败，加锁失败的线程会被放入一个`FIFO`的等待队列中，比列会被`UNSAFE.park()`操作挂起，等待其他获取锁的线程释放锁才能够被唤醒

另外`state`的操作都是通过`CAS`来保证其并发修改的安全性

AQS的基本API

- getState()：获取锁的标志state值
- setState()：设置锁的标志state值
- tryAcquire(int)：独占方式获取锁。尝试获取资源，成功则返回true，失败则返回false
- tryRelease(int)：独占方式释放锁。尝试释放资源，成功则返回true，失败则返回false

## AQS源码

以**ReentrantLock**为例，加锁过程

1、尝试加锁

2、加锁失败，线程放入队列

3、线程入队列后，进入阻塞状态

### 非公平锁NonfairSync为例，案例都是理想正常流程执行 A-B-C 线程执行，实际上有差异

### Lock方法（挂起线程）

```java
static final class NonfairSync extends Sync {
    private static final long serialVersionUID = 7316153563782823691L;

    /**
     * Performs lock.  Try immediate barge, backing up to normal
     * acquire on failure.
     */
    final void lock() {
        if (compareAndSetState(0, 1))
            setExclusiveOwnerThread(Thread.currentThread());
        else
            acquire(1);
    }

    protected final boolean tryAcquire(int acquires) {
        return nonfairTryAcquire(acquires);
    }
}
```

`compareAndSetState` 比较state，这里A先进入比较并交换 设置`state`0到1，`AQS`实现，

~~~java
protected final boolean compareAndSetState(int expect, int update) {
    return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
}

protected final void setExclusiveOwnerThread(Thread thread) {
    exclusiveOwnerThread = thread;
}

~~~

此时B线程执行`lock`开始抢占锁，此时非常抱歉，此时`state=1`,那么B只能进入`acquire`方法

模板方法进入`AbstractQueuedSynchronizer`的acquire方法，注意：tryAcquire 方法AbstractQueuedSynchronizer本身不实现，会抛出UnsupportedOperationException异常

```java
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```

而非公平NonfairSync里面的实现 

```java
protected final boolean tryAcquire(int acquires) {
    return nonfairTryAcquire(acquires);
}
```

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

由于此时C=1,并且getExclusiveOwnerThread=A ,所以此时返回false

接下来执行父类模板定义的方法

 acquireQueued(addWaiter(Node.EXCLUSIVE), arg)

```java
private Node addWaiter(Node mode) {
    Node node = new Node(Thread.currentThread(), mode);
    // Try the fast path of enq; backup to full enq on failure
    Node pred = tail;
    if (pred != null) {
        node.prev = pred;
        if (compareAndSetTail(pred, node)) {
            pred.next = node;
            return node;
        }
    }
    enq(node);
    return node;
}
```

```java
private Node enq(final Node node) {
    for (;;) {
        Node t = tail;
        if (t == null) { // Must initialize
            if (compareAndSetHead(new Node()))
                tail = head;
        } else {
            node.prev = t;
            if (compareAndSetTail(t, node)) {
                t.next = node;
                return t;
            }
        }
    }
}
```

此时tail=null,enq 自旋先创建`哨兵节点`,用于占位，再创建b节点入队，并将Node 头尾节点设置

```java
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

​    for (;;)这里又是一个自旋，

第一次自旋，tryAcquire去抢锁，这里默认A 休眠五分钟，所以这里肯定抢占失败返回false，执行shouldParkAfterFailedAcquire 将p也就是当前的哨兵节点，此时哨兵节点pred的waitStatus=0，条件进入比较并交换为-1。

第二次自旋，shouldParkAfterFailedAcquire 哨兵节点pred的waitStatus=-1，此方法返回true,便执行parkAndCheckInterrupt中的方法，此时就把线程B挂起

```java
private final boolean parkAndCheckInterrupt() {
    LockSupport.park(this);
    return Thread.interrupted();
}
```

### unLock 方法（唤醒线程）

 unlock--->sync.release(1)--->tryRelease 子类实现

```java
public final boolean release(int arg) {
    if (tryRelease(arg)) {
        Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    return false;
}
```

```java
protected final boolean tryRelease(int releases) {
    int c = getState() - releases; // 1-1=0
    if (Thread.currentThread() != getExclusiveOwnerThread())
        throw new IllegalMonitorStateException();
    boolean free = false;
    if (c == 0) {
        free = true;
        setExclusiveOwnerThread(null);
    }
    setState(c);
    return free;
}
```

tryRelease=0,返回true,执行unparkSuccessor，此时node是头节点是哨兵节点 waitStatus=-1,下一节点B将执行

  LockSupport.unpark(s.thread)。也就是B节点线程唤醒

```
private void unparkSuccessor(Node node) {
    /*
     * If status is negative (i.e., possibly needing signal) try
     * to clear in anticipation of signalling.  It is OK if this
     * fails or if status is changed by waiting thread.
     */
    int ws = node.waitStatus; 
    if (ws < 0)
        compareAndSetWaitStatus(node, ws, 0);

    /*
     * Thread to unpark is held in successor, which is normally
     * just the next node.  But if cancelled or apparently null,
     * traverse backwards from tail to find the actual
     * non-cancelled successor.
     */
    Node s = node.next;
    if (s == null || s.waitStatus > 0) {
        s = null;
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0)
                s = t;
    }
    if (s != null)
        LockSupport.unpark(s.thread);
}
```
