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

### 类的初始化情况：主动使用 vs被动使用

> Java程序对类的使用分为两种：**主动使用和被动使用**。

#### 主动使用

Class只有在必须要首次使用的时候才会被装载，Java虚拟机不会无条件地装载Class类型。Java虚拟机规定，**一个类或接口在初次使用前，必须要进行初始化**。这里指的“使用”，是指`主动使用`，主动使用只有下列几种情况：（即：如果出现如下的情况，则会对类进行初始化操作。而初始化操作之前的加载、验证、准备已经完成）

1.当创建一个类的实例时，比如使用new关键字，或者通过反射、克隆、反序列化；

2.当调用类的静态方法时，即当使用了字节码invokestatic指令；

3.当使用类、接口的静态字段时（final修饰特殊考虑），比如，使用getstatic或者putstatic指令。（对应访问变量、赋值变量操作）；

4.当使用java.lang.reflect包中的方法反射类的方法时。比如:Class.forName；

5.当初始化子类时，如果发现其父类还没有进行过初始化，则需要先触发其父类的初始化；

6.如果一个接口定义了default方法，那么直接实现或者间接实现该接口的类的初始化，该接口要在其之前被初始化；

7.当虚拟机启动时，用户需要指定一个要执行的主类（包含main()方法的那个类），虚拟机会先初始化这个主类；

8.当初次调用 MethodHandle实例时，初始化该MethodHandle指向的方法所在的类。（涉及解析REF_getStatic、REF_putStatic、REF_invokeStatic方法句柄对应的类）；

#### 被动使用

除了以上的情况属于主动使用，其他的情况均属于被动使用。**被动使用不会引起类的初始化。**

1. 当访问一个静态字段时，只有真正声明这个字段的类才会被初始化；
当通过子类引用父类的静态变量，不会导致子类初始化；
2. 通过数组定义类引用，不会触发此类的初始化；
3. 引用常量不会触发此类或接口的初始化。因为常量在链接阶段就已经被显式赋值了；
4. 调用ClassLoader类的loadClass()方法加载一个类，并不是对类的主动使用，不会导致类的初始化；

## 类的加载器

### 作用

> 类的加载器是JVM执行类加载机制的前提

**ClassLoader**

`ClassLoader`是Java的核心组件，所有的Class都是由ClassLoader进行加载的

`ClassLoader`负责通过各种方式将Class信息的二进制数据流读入JVM内部，转换为一个与目标类对应的java.lang.Class对象实例，然后交给Java虚拟机进行链接、初始化等操作

`ClassLoader`在整个装载阶段，只能影响到类的`加载`（`Loading阶段`），而无法通过ClassLoader去改变类的`链接和初始化`行为。至于它是否可以运行，则由Execution Engine决定。

### 显示加载与隐士加载

- **显示加载**：指的是在代码中通过调用ClassLoader加载class对象，如直接使用Class.forName（name）或this.getClass().getClassLoader().loadClass()加载class对象
- **隐式加载**：不直接在代码中调用ClassLoader的方法加载class对象，而是通过虚拟机自动加载到内存中，如在加载某个类的class文件时，该类的class文件中引用了另外一个类的对象，此时额外引用的类将通过JVM自动加载到内存中。

### 类加载机制的基本特征

- **双亲委派模型**：
- **可见性**：子类加载器可以访问父加载器加载的类型，但是反过来是不允许的。
- **单一性**：父加载器中加载过的类型，就不会在子加载器中重复加载。但是注意，类加载器“邻居”间，同一类型仍然可以被加载多次，因为互相并不可见。

### 类加载器分类（2类）

> **引导类加载器(Bootstrap ClassLoader)**和**自定义类加载器(User-Defined ClassLoader)**

![img](https://file.hyqup.cn/img/20210723180028.png)

> 父子加载类实际上不存在继承关系，而是一种组合关系
>

```java
class ClassLoader {

	ClassLoader parent;       父类加载器

	public ClassLoader(ClassLoader parent) {

	this.parent = parent;

	}

}

class ParentClassLoader extends ClassLoader {

	public ParentClassLoader(ClassLoader parent) {

	super(parent);

	}

}

class ChildClassLoader extends ClassLoader {

	public ChildClassLoader(ClassLoader parent) {

	//parent = new ParentClassLoader();

	super(parent);

	}

}
```

### ClassLoader加载逻辑

```java
   protected Class<?> loadClass(String name, boolean resolve)
        throws ClassNotFoundException
    {
        synchronized (getClassLoadingLock(name)) {
            // First, check if the class has already been loaded
            // 查找一下这个类是不是已经加载过了
            Class<?> c = findLoadedClass(name);
            //如果没有加载过
            if (c == null) {
                long t0 = System.nanoTime();
                try {
                    //parent  每个类加载器都有个父加载器,判断是否有父类加载器，存在则调用父类加载器去加载。双亲委派模型在这
                    if (parent != null) {
                        c = parent.loadClass(name, false);
                    } else {
                        //如果父类加载器为空，就说明到达顶层也就是BootstrapClassLoader,BootstrapClassLoader属于C/C++编写这里拿不到对象的
                        c = findBootstrapClassOrNull(name);
                    }
                } catch (ClassNotFoundException e) {
                    // ClassNotFoundException thrown if class not found
                    // from the non-null parent class loader
                }
				//如果父类加载都没有加载，则使用当前类加载
                if (c == null) {
                    // If still not found, then invoke findClass in order
                    // to find the class.
                    long t1 = System.nanoTime();
                    // findClass用于加载
                    c = findClass(name);

                    // this is the defining class loader; record the stats
                    sun.misc.PerfCounter.getParentDelegationTime().addTime(t1 - t0);
                    sun.misc.PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
                    sun.misc.PerfCounter.getFindClasses().increment();
                }
            }
            // 是否需要解析，默认都是false
            if (resolve) {
                resolveClass(c);
            }
            return c;
        }
    }
```

### 双亲委派模型

> 如果一个类加载器在接到加载类的请求时，它**首先不会自己尝试去加载这个类**，而是把**这个请求任务委托给父类加载器去完成，依次递归，**如果父类加载器可以完成类加载任务，就成功返回。**只有父类加载器无法完成此加载任务时，才自己去加载**

![双亲委派模型](https://file.hyqup.cn/img/20210601230727770.png)

#### 双亲委派模型优势

- 双亲委派保证类加载器，自下而上的委派，又自上而下的加载，避免类的重复加载，确保一个类的全局唯一性。保证每一个类在各个类加载器中都是同一个类。

  简言之：当父亲已经加载了该类时，就没有必要子类的ClassLoader 再加载一次

- 保护程序安全，防止核心API被接口重写

### 自定义类加载器

> 自定义Class类继承ClassLoader重写findClass方法

### 破坏双亲委派

> 继承ClassLoader，并重写findClass，如果想不遵循双亲委派的类加载顺序，还需要重写loadClass。

双亲委派的机制核心是loadClass的逻辑加载自下而上委派，自上而下加载来实现的。我们重写loadClass不遵循这种规则，即打破了双亲委派
