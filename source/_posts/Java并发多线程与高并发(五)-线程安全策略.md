---
title: Java并发多线程与高并发(五)-线程安全策略
index_img: https://file.hyqup.cn/img/16df9bf3400b4e6db45a87f643daf9c2.jpg
banner_img: /img/default.png
abbrlink: d50876c2
date: 2021-04-28 23:40:44
tags:
- 不可变对象
- 集合
categories:
- 并发
- 多线程
description:
---

本文讲述在Java中经常会使用一些对象，集合。单线程下没问题，在多线程下就会出现问题，我们对这些对象在使用的时候如何去做来保证线程安全

<!-- more -->

## 不可变对象

### 不可变对象需要满足的条件：

- 对象创建以后其状态就不能修改
- 对象所有域都是final类型
- 对象是正确创建的（在对象创建期间，this引用没有逸出）

简而言之就是将类声明为final，将所有的成员声明为私有的，对变量不提供set方法。将所有可变的成员声明为final。在get方法中不返回对象本身，而是克隆对象的拷贝。（可参考String类）。

### final关键字：类、方法、变量

- 修饰类：不能被继承，**final类中的方法会被隐式的被指定为final方法**

- 修饰方法：1、锁定方法被继承类修改；2、效率。**一个类的private方法会被隐式指定为final类型**

- 修饰变量：基本数据类型变量、引用类型变量。**基本数据类型数值一旦被初始就不能被修改了，引用类型在对其初始化之后便不能指向另外一个*对象***，*（注意）引用类型的值是可修改的*

  ```java
  package cn.hyqup.concurrency.immutable;
  
  import java.util.HashMap;
  import java.util.Map;
  
  /**
   * Copyright © 2021灼华. All rights reserved.
   *
   * @author create by hyq
   * @version 1.0
   * @date 2021/5/15
   * @description:
   */
  public class ImmutableExample1 {
      private final static Integer a = 1;
      private final static String b = "2";
      private final static Map<Integer, Integer> c = new HashMap<>();
  
      static {
          c.put(1, 1);
          c.put(2, 2);
      }
  
      public static void main(String[] args) {
  //        a=11; 不可以修改，编译时候就不允许
  //        b="22"; 不可以修改，编译时候就不允许
  //        c=new HashMap<>(); 不可以指向新的对象，编译时候就不允许
          c.put(1, 3);  // 引用类型但是可以修改值
      }
  }
  
  ```

###  不可变引用数据类型

- java提供：Collections.unmodifiableXXX:Collection、List、Set、Map...

- guava提供:ImmutableXXX：Collection、List、Set、Map...


## 线程封闭

当访问共享的可变数据时，通常需要同步。一种避免同步的方式就是不共享数据。如果仅在单线程内访问数据，就不需要同步，这种技术称为线程封闭（thread  confinement）,ThreadLocal

## 线程不安全的类与写法

- StringBuilder和StringBuffer两种字符串拼接类
  - StringBuilder 线程不安全，但效率高
  - StringBuffer 线程安全，但效率低

- SimpleDateFormate

  多线程下不安全，可以采用jodaTime来实现

- ArrayList,HashSet，HashMap等集合（使用同步容器解决）

  - ArrayList
  - HashSet
  - HashMap

## 同步容器

使用synchronized来实现

- ArrayList->Vector,Stack

- HashMap->HashTable(key、value不能为null)

- Collections.synchronizedXXX(List、Set、Map)

## 并发容器及安全共享策略

同步容器的安全性得以保证，但是性能不是很好，所以Java中通常使用并发容器来替代同步容器，来完成并发下的工作

### 并发容器J.U.C(java.util.concurrent)

#### ArrayList --> CopyOnWriteArrayList

##### 概念 : 

简单的讲就是写操作时赋值,当有新元素添加到CopyOnWriteArrayList时,它先从原有的数组里边Copy一份出来然后在新的数组上做些操作,操作完成以后在将引用指向新的数组;CopyOnWriteArrayList所有的操作都是在锁的保护下进行的,这样做的目的主要是为了在多线程并发做add操作的时候复制出多个副本出来导致数据混乱

##### 缺点 :

- 由于是copy的操作所以比较消耗内存,如果元素的内容较多的时候可能会触发GC

- 不能用于实时读的场景,它比较适合读多写少的场景(因为写的时候会复制新集合)

##### 思想 :

- 读写分离
- 最终一致性
- 另外开辟空间解决并发冲突

#### HashSet --> CopyOnWriteArraySet和TreeSet --> ConcurrentSkipListSet

- CopyOnWriteArraySet 线程安全，底层实现是使用CopyOnWriteArrayList，特性类似CopyOnWriteArrayList，适合于**数据量小**的时候，只读操作大于可变操作的时候
-  ConcurrentSkipListSet 是jdk6新增的类，和TreeSet 一样支持自然排序，并且在构造的时候自己定义比较器。基于Map集合的，在多线程环境下，（set 、add、remove）等方法都是线程安全的，但是对于批量操作（addAll、removeAll等）并不能保证以原子方式进行操作，原因是底层还是（add、remove 等）批量操作的时候只能保证每一次的操作是原子性的，不能保证每一次操作不能被其他操作打断。所以**使用批量操作的时候可能需要手动处理一下（加锁）**

#### HashMap --> ConcurrentHashMap 和 TreeMap --> ConcurrentSkipListMap

- ConcurrentHashMap  不允许空值，ConcurrentHashMap针对读操作多了特别多的优化,具有特别高的并发性

- ConcurrentSkipListMap 底层是使用SkipList这种跳表的结构实现的

  

  对比：ConcurrentHashMap  存取效率高于ConcurrentSkipListMap 。但是：

  1、ConcurrentSkipListMap 的key是有序的 2、支持更高的并发

总结：这里只涉及到了一小部分J.U.C的知识点





