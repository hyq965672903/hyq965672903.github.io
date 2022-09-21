---
title: JVM与GC调优(一)-字节码篇
index_img: /img/default.png
banner_img: /img/default.png
date: 2022-09-13 14:39:40
tags:
categories:
description:
---

了解JVM整体架构图，javac编译器编译步骤，class文件的解读，字节码指令是什么以及常见的字节码有哪些

<!-- more -->

## 知识体系

> 概述各

- **字节码**
- **类的加载**
- **运行时内存**
- **对象内存布局**
- **执行引擎**
- **垃圾回收**
- **JVM性能监控**
- **性能调优案例**

 ## 字节码篇

> 什么是字节码指令？

Java虚拟机的指令是由`一个字节长度`的、代表着某种特定操作含义的`操作码`（opcode），以及其后跟随`零至多个`代表此操作所需参数的`操作数`（operand）

### 字节码案例

#### 案例1

```java
public class ByteCondeInterview {

    /**
     * 包装类对象的缓存问题
     * Byte         -128-127
     * Short        -128-127
     * Integer      -128-127
     * Long         -128-127
     * Float        没有
     * Double       没有
     * Character    0-127
     * Boolean      true/false
     */
    @Test
    public void test1(){
        Integer i1=128;
        Integer i2=128;

        System.out.println(i1==i2);
    }
}
```

通过idea插件jclasslib观察字节码文件

```text
 0 sipush 128
 3 invokestatic #2 <java/lang/Integer.valueOf : (I)Ljava/lang/Integer;>
 6 astore_1
 7 sipush 128
10 invokestatic #2 <java/lang/Integer.valueOf : (I)Ljava/lang/Integer;>
13 astore_2
14 getstatic #3 <java/lang/System.out : Ljava/io/PrintStream;>
17 aload_1
18 aload_2
19 if_acmpne 26 (+7)
22 iconst_1
23 goto 27 (+4)
26 iconst_0
27 invokevirtual #4 <java/io/PrintStream.println : (Z)V>
30 return

```

Integer.valueOf里面回去调用 IntegerCache缓存对象

#### 案例2

```
@Test
public void test2() {
    String str1 = new String("hello") + new String("world");
    String str2 = "helloworld";

    System.out.println(str1 == str2); //false
}
```

```text
 0 new #5 <java/lang/StringBuilder>
 3 dup
 4 invokespecial #6 <java/lang/StringBuilder.<init> : ()V>
 7 new #7 <java/lang/String>
10 dup
11 ldc #8 <hello>
13 invokespecial #9 <java/lang/String.<init> : (Ljava/lang/String;)V>
16 invokevirtual #10 <java/lang/StringBuilder.append : (Ljava/lang/String;)Ljava/lang/StringBuilder;>
19 new #7 <java/lang/String>
22 dup
23 ldc #11 <world>
25 invokespecial #9 <java/lang/String.<init> : (Ljava/lang/String;)V>
28 invokevirtual #10 <java/lang/StringBuilder.append : (Ljava/lang/String;)Ljava/lang/StringBuilder;>
31 invokevirtual #12 <java/lang/StringBuilder.toString : ()Ljava/lang/String;>
34 astore_1
35 ldc #13 <helloworld>
37 astore_2
38 getstatic #3 <java/lang/System.out : Ljava/io/PrintStream;>
41 aload_1
42 aload_2
43 if_acmpne 50 (+7)
46 iconst_1
47 goto 51 (+4)
50 iconst_0
51 invokevirtual #4 <java/io/PrintStream.println : (Z)V>
54 return

```

str1不和str2相等的原因在于，str1的对象是由两个StringBuilder对象转化使用append拼接，最后调用StringBuilder的toString,而toString里面使用的值又是StringBuilder对象的值新new一个对象的，而str2是直接在堆中常量对象，二者地址自然不对等

```java
@Override
public String toString() {
    // Create a copy, don't share the array
    return new String(value, 0, count);
}
```

#### 案例3

```java

    @Test
    public void test3() {
        String str1 = new String("hello") + new String("world");
        str1.intern();
        String str2 = "helloworld";
        System.out.println(str1 == str2); //true
    }
```

加入  str1.intern(); 就是true  jdk6还事false，jdk8后是true,因为jdk8后的常量池也在堆中，此时通过  str1.intern();会将常量池改对象的地址指向堆中的地址

### 如何解读class文件？

.class文件以16进制存储的

```text
0000000 ca fe ba be 00 00 00 34 00 4b 0a 00 10 00 2a 0a | .......4.K....*.
0000010 00 2b 00 2c 09 00 2d 00 2e 0a 00 2f 00 30 07 00 | .+.,..-..../.0..
0000020 31 0a 00 05 00 2a 07 00 32 08 00 33 0a 00 07 00 | 1....*..2..3....
0000030 34 0a 00 05 00 35 08 00 36 0a 00 05 00 37 08 00 | 4....5..6....7..
0000040 38 0a 00 07 00 39 07 00 3a 07 00 3b 01 00 06 3c | 8....9..:..;...<
0000050 69 6e 69 74 3e 01 00 03 28 29 56 01 00 04 43 6f | init>...()V...Co
0000060 64 65 01 00 0f 4c 69 6e 65 4e 75 6d 62 65 72 54 | de...LineNumberT
0000070 61 62 6c 65 01 00 12 4c 6f 63 61 6c 56 61 72 69 | able...LocalVari
0000080 61 62 6c 65 54 61 62 6c 65 01 00 04 74 68 69 73 | ableTable...this
0000090 01 00 21 4c 63 6e 2f 68 79 71 75 70 2f 6a 76 6d | ..!Lcn/hyqup/jvm
00000a0 2f 42 79 74 65 43 6f 6e 64 65 49 6e 74 65 72 76 | /ByteCondeInterv
00000b0 69 65 77 3b 01 00 05 74 65 73 74 31 01 00 02 69 | iew;...test1...i
00000c0 31 01 00 13 4c 6a 61 76 61 2f 6c 61 6e 67 2f 49 | 1...Ljava/lang/I
00000d0 6e 74 65 67 65 72 3b 01 00 02 69 32 01 00 0d 53 | nteger;...i2...S
00000e0 74 61 63 6b 4d 61 70 54 61 62 6c 65 07 00 3a 07 | tackMapTable..:.
00000f0 00 3c 07 00 3d 01 00 19 52 75 6e 74 69 6d 65 56 | .<..=...RuntimeV
0000100 69 73 69 62 6c 65 41 6e 6e 6f 74 61 74 69 6f 6e | isibleAnnotation
0000110 73 01 00 10 4c 6f 72 67 2f 6a 75 6e 69 74 2f 54 | s...Lorg/junit/T
0000120 65 73 74 3b 01 00 05 74 65 73 74 32 01 00 04 73 | est;...test2...s
0000130 74 72 31 01 00 12 4c 6a 61 76 61 2f 6c 61 6e 67 | tr1...Ljava/lang
0000140 2f 53 74 72 69 6e 67 3b 01 00 04 73 74 72 32 07 | /String;...str2.
0000150 00 32 01 00 05 74 65 73 74 33 01 00 0a 53 6f 75 | .2...test3...Sou
0000160 72 63 65 46 69 6c 65 01 00 17 42 79 74 65 43 6f | rceFile...ByteCo
0000170 6e 64 65 49 6e 74 65 72 76 69 65 77 2e 6a 61 76 | ndeInterview.jav
0000180 61 0c 00 11 00 12 07 00 3c 0c 00 3e 00 3f 07 00 | a.......<..>.?..
0000190 40 0c 00 41 00 42 07 00 3d 0c 00 43 00 44 01 00 | @..A.B..=..C.D..
00001a0 17 6a 61 76 61 2f 6c 61 6e 67 2f 53 74 72 69 6e | .java/lang/Strin
00001b0 67 42 75 69 6c 64 65 72 01 00 10 6a 61 76 61 2f | gBuilder...java/
00001c0 6c 61 6e 67 2f 53 74 72 69 6e 67 01 00 05 68 65 | lang/String...he
00001d0 6c 6c 6f 0c 00 11 00 45 0c 00 46 00 47 01 00 05 | llo....E..F.G...
00001e0 77 6f 72 6c 64 0c 00 48 00 49 01 00 0a 68 65 6c | world..H.I...hel
00001f0 6c 6f 77 6f 72 6c 64 0c 00 4a 00 49 01 00 1f 63 | loworld..J.I...c
0000200 6e 2f 68 79 71 75 70 2f 6a 76 6d 2f 42 79 74 65 | n/hyqup/jvm/Byte
0000210 43 6f 6e 64 65 49 6e 74 65 72 76 69 65 77 01 00 | CondeInterview..
0000220 10 6a 61 76 61 2f 6c 61 6e 67 2f 4f 62 6a 65 63 | .java/lang/Objec
0000230 74 01 00 11 6a 61 76 61 2f 6c 61 6e 67 2f 49 6e | t...java/lang/In
0000240 74 65 67 65 72 01 00 13 6a 61 76 61 2f 69 6f 2f | teger...java/io/
0000250 50 72 69 6e 74 53 74 72 65 61 6d 01 00 07 76 61 | PrintStream...va
0000260 6c 75 65 4f 66 01 00 16 28 49 29 4c 6a 61 76 61 | lueOf...(I)Ljava
0000270 2f 6c 61 6e 67 2f 49 6e 74 65 67 65 72 3b 01 00 | /lang/Integer;..
0000280 10 6a 61 76 61 2f 6c 61 6e 67 2f 53 79 73 74 65 | .java/lang/Syste
0000290 6d 01 00 03 6f 75 74 01 00 15 4c 6a 61 76 61 2f | m...out...Ljava/
00002a0 69 6f 2f 50 72 69 6e 74 53 74 72 65 61 6d 3b 01 | io/PrintStream;.
00002b0 00 07 70 72 69 6e 74 6c 6e 01 00 04 28 5a 29 56 | ..println...(Z)V
00002c0 01 00 15 28 4c 6a 61 76 61 2f 6c 61 6e 67 2f 53 | ...(Ljava/lang/S
00002d0 74 72 69 6e 67 3b 29 56 01 00 06 61 70 70 65 6e | tring;)V...appen
00002e0 64 01 00 2d 28 4c 6a 61 76 61 2f 6c 61 6e 67 2f | d..-(Ljava/lang/
00002f0 53 74 72 69 6e 67 3b 29 4c 6a 61 76 61 2f 6c 61 | String;)Ljava/la
0000300 6e 67 2f 53 74 72 69 6e 67 42 75 69 6c 64 65 72 | ng/StringBuilder
0000310 3b 01 00 08 74 6f 53 74 72 69 6e 67 01 00 14 28 | ;...toString...(
0000320 29 4c 6a 61 76 61 2f 6c 61 6e 67 2f 53 74 72 69 | )Ljava/lang/Stri
0000330 6e 67 3b 01 00 06 69 6e 74 65 72 6e 00 21 00 0f | ng;...intern.!..
0000340 00 10 00 00 00 00 00 04 00 01 00 11 00 12 00 01 | ................
0000350 00 13 00 00 00 2f 00 01 00 01 00 00 00 05 2a b7 | ...../........*.
0000360 00 01 b1 00 00 00 02 00 14 00 00 00 06 00 01 00 | ................
0000370 00 00 0d 00 15 00 00 00 0c 00 01 00 00 00 05 00 | ................
0000380 16 00 17 00 00 00 01 00 18 00 12 00 02 00 13 00 | ................
0000390 00 00 98 00 03 00 03 00 00 00 1f 11 00 80 b8 00 | ................
00003a0 02 4c 11 00 80 b8 00 02 4d b2 00 03 2b 2c a6 00 | .L......M...+,..
00003b0 07 04 a7 00 04 03 b6 00 04 b1 00 00 00 03 00 14 | ................
00003c0 00 00 00 12 00 04 00 00 00 1c 00 07 00 1d 00 0e | ................
00003d0 00 1f 00 1e 00 20 00 15 00 00 00 20 00 03 00 00 | ..... ..... ....
00003e0 00 1f 00 16 00 17 00 00 00 07 00 18 00 19 00 1a | ................
00003f0 00 01 00 0e 00 11 00 1b 00 1a 00 02 00 1c 00 00 | ................
0000400 00 29 00 02 ff 00 1a 00 03 07 00 1d 07 00 1e 07 | .)..............
0000410 00 1e 00 01 07 00 1f ff 00 00 00 03 07 00 1d 07 | ................
0000420 00 1e 07 00 1e 00 02 07 00 1f 01 00 20 00 00 00 | ............ ...
0000430 06 00 01 00 21 00 00 00 01 00 22 00 12 00 02 00 | ....!.....".....
0000440 13 00 00 00 b0 00 04 00 03 00 00 00 37 bb 00 05 | ............7...
0000450 59 b7 00 06 bb 00 07 59 12 08 b7 00 09 b6 00 0a | Y......Y........
0000460 bb 00 07 59 12 0b b7 00 09 b6 00 0a b6 00 0c 4c | ...Y...........L
0000470 12 0d 4d b2 00 03 2b 2c a6 00 07 04 a7 00 04 03 | ..M...+,........
0000480 b6 00 04 b1 00 00 00 03 00 14 00 00 00 12 00 04 | ................
0000490 00 00 00 25 00 23 00 26 00 26 00 28 00 36 00 29 | ...%.#.&.&.(.6.)
00004a0 00 15 00 00 00 20 00 03 00 00 00 37 00 16 00 17 | ..... .....7....
00004b0 00 00 00 23 00 14 00 23 00 24 00 01 00 26 00 11 | ...#...#.$...&..
00004c0 00 25 00 24 00 02 00 1c 00 00 00 29 00 02 ff 00 | .%.$.......)....
00004d0 32 00 03 07 00 1d 07 00 26 07 00 26 00 01 07 00 | 2.......&..&....
00004e0 1f ff 00 00 00 03 07 00 1d 07 00 26 07 00 26 00 | ...........&..&.
00004f0 02 07 00 1f 01 00 20 00 00 00 06 00 01 00 21 00 | ...... .......!.
0000500 00 00 01 00 27 00 12 00 02 00 13 00 00 00 b9 00 | ....'...........
0000510 04 00 03 00 00 00 3c bb 00 05 59 b7 00 06 bb 00 | ......<...Y.....
0000520 07 59 12 08 b7 00 09 b6 00 0a bb 00 07 59 12 0b | .Y...........Y..
0000530 b7 00 09 b6 00 0a b6 00 0c 4c 2b b6 00 0e 57 12 | .........L+...W.
0000540 0d 4d b2 00 03 2b 2c a6 00 07 04 a7 00 04 03 b6 | .M...+,.........
0000550 00 04 b1 00 00 00 03 00 14 00 00 00 16 00 05 00 | ................
0000560 00 00 2e 00 23 00 2f 00 28 00 30 00 2b 00 31 00 | ....#./.(.0.+.1.
0000570 3b 00 32 00 15 00 00 00 20 00 03 00 00 00 3c 00 | ;.2..... .....<.
0000580 16 00 17 00 00 00 23 00 19 00 23 00 24 00 01 00 | ......#...#.$...
0000590 2b 00 11 00 25 00 24 00 02 00 1c 00 00 00 29 00 | +...%.$.......).
00005a0 02 ff 00 37 00 03 07 00 1d 07 00 26 07 00 26 00 | ...7.......&..&.
00005b0 01 07 00 1f ff 00 00 00 03 07 00 1d 07 00 26 07 | ..............&.
00005c0 00 26 00 02 07 00 1f 01 00 20 00 00 00 06 00 01 | .&....... ......
00005d0 00 21 00 00 00 01 00 28 00 00 00 02 00 29       | .!.....(.....)
00005de 
```

如上，.class

文件开头的4个字节`cafe babe` 称之为 **魔数**,0000是编译器jdk版本的次版本号0，0034转化为十进制是52,是主版本号，java的版本号从45开始  52对应1.8，接下来就是一些`常量池`、`方法表集合`等信息 这些字16进制的信息对应为jvm 所认识的字节码指令。

### 字节码指令集与解析

> JVM中的字节码指令集按用途大致分成9类

- 加载与存储指令

​	    **局部变量压入操作数栈**

​			xload：（其中x 为 i、l、f、d、a）

​		**常量入栈**

​			const、push、ldc

- 算术指令
- 类型转换指令
- 对象的创建与访问指令
- 方法调用与返回指令

1.`invokevirtual`指令用于调用对象的实例方法，根据对象的实际类型进行分派(虚方法分派)，支持多态，这也是java语言中最常见的方法分派方式。

2.`invokerinterface`指令用于调用接口方法，它会在运行时搜索由特定对象所实现的这个接口方法，并找出适合的方法进行调用

3.`invokespecial`指令用于调用一些需要特殊处理的实例方法，包括实例`初始化方法(构造器)`，私有方法和父类方法，这些方法都是静态类型绑定的，不会在调用时进行动态派发

4.`invokestatic`指令用于调用命名类中的类方法(static方法)，这是静态绑定的

5.`invokedynamic`指令用于调用动态绑定的方法，这是jdk1.7之后新加入的指令，用于在运行时动态解析出调用点限定符所引用的方法，并执行该方法。前面4条调用指令的分派逻辑都固化在java虚拟机内部，而invokedynamic指令的分配逻辑是由用户所设定的引导方法决定的

- 操作数栈管理指令
- 控制转移指令
- 异常处理指令
- 同步控制指令

 关于字节码指令集详细：

> https://www.cnblogs.com/sjqstart/p/15044517.html

> https://blog.csdn.net/mynameisgt/article/details/125052344

  
