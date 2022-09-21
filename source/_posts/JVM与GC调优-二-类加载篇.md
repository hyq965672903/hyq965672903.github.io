---
title: JVM与GC调优(二)-类加载篇
index_img: /img/default.png
banner_img: /img/default.png
date: 2022-09-21 20:53:12
tags:
categories:
description:
---

类的加载过程解析与双亲委派机制与破坏双亲委派机制相关

<!-- more -->

## 类的加载过程

> 类加载主要过程分为 加载、链接、初始化三个阶段，而链接阶段又分为验证 准备 解析

![image-20220921210744405](https://file.hyqup.cn/img/image-20220921210744405.png)

 ClassLoader只负责class文件的加载，至于它是否可以运行，则由Execution Engine决定。

 [JVM内存模型——堆(heap)、栈(stack)和方法区(method)](https://www.cnblogs.com/aaabbbcccddd/p/14539063.html)

### 过程一：Loading（加载）阶段

> 将Java类的字节码文件加载到机器内存中，并在内存中构建出java类的原型实例（类模板对象），类结构信息存储到`方法区`

`类模板对象`：本质就是java类在JVM内存中的一个快照，JVM将从字节码文件中解析出的常量池、类字段、类方法等信息存储到类模板中，在JVM运行的时候，就可以通过类模板获取Java类的所有信息，能够对Java类的成员变量进行遍历，也能进行Java 方法的调用

反射的原理也就JVM在运行期间去拿到类模板信息

### 过程二：Linking（链接）阶段

#### 小节一：链接阶段值Verification(验证) 

> 目的是**保证加载的字节码是合法、合理并符合规范的**

验证的内容则涵盖了类数据信息的**格式验证、语义检查、字节码验证，以及符号引用验证**等

#### 小节二：链接阶段值Preparation(准备)

> **为类的静态变量分配内存，并将其初始化为默认值**

| 类型      | 默认初始值 |
| --------- | ---------- |
| byte      | (byte)0    |
| short     | (short)0   |
| int       | 0          |
| long      | OL         |
| float     | 0.0f       |
| double    | 0.0        |
| char      | \u0000     |
| boolean   | false      |
| reference | null       |

#### 小节三：链接阶段值Resolution(解析)

> **解析阶段（Resolution），简言之，将类、接口、字段和方法的符号引用转为直接引用。**

符号引用就是一些字面量的引用，和虚拟机的内部数据结构和和内存布局无关。比较容易理解的就是**在Class类文件中，通过常量池进行了大量的符号引用**。但是在程序实际运行时，只有符号引用是不够的，比如当println()方法被调用时，系统需要明确知道该方法的位置。

### 过程三：Initialization（初始化）阶段

> ***初始化阶段，简言之，为类的静态变量赋予正确的初始值。***

1、具体描述

类的初始化是类装载的最后一个阶段。如果前面的步骤都没有问题，那么表示类可以顺利装载到系统中。此时，类才会开始执行Java字节码。**（即：到了初始化阶段，才真正开始执行类中定义的Java程序代码。）**

初始化阶段的重要工作是执行类的初始化方法：`<clinit>()方法`

该方法仅能由`Java编译器生成`并由`JVM调用`，程序开发者无法自定义一个同名的方法，更无法直接在Java程序中调用该方法，虽然该方法也是由字节码指令所组成。
它是由类静态成员的赋值语句以及static语句块合并产生的。 

clinit编译生成：

​	**静态的变量赋值编译才会生成clint方法**

案例说明：

```java
public int a=1; //不会，不是静态，链接(Linking)的准备阶段赋值
public static int a; //不会，没有赋值
public static final int a=1;//final 修饰后不是变量是常量所以也不会

public static  int a=1; // 此时会，静态变量初始化赋值
```

####  static final 修饰的一定不会在初始化赋值吗？

eg: public static final Integter INTEGTER_CONSTANT=Integer.valueOf(1000);

因为这里不是常量而是一个方法调用，所以此时也会生成clint

#### 总结：

使用 static+final修饰的成员变量，称为：`全局常量`

什么时候在`链接`的准备阶段赋值：给该全局常量赋的值是`字面量`或者`常量`，不涉及到`方法的调用`，其余场景在初始化阶段赋值
