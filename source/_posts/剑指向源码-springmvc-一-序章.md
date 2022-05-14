---
title: 剑指向源码-springmvc(一)-序章
index_img: /img/default.png
banner_img: /img/default.png
date: 2022-05-14 09:42:38
tags:
categories:
description:
---

搭建springmvc源码环境。准备工作与前置工作分析

<!-- more -->

> 前置基础：了解SPI,Service Provider Interface,服务提供者接口 jdk提供给“服务提供厂商”或者“插件开发者”使用的接口,不需要修改原来作为接口的jar的情况下，将原来实现的那个jar替换为另外一种实现的jar即可。

使用规范：

定义服务的通用接口，针对通用的服务接口，提供具体的实现类。

在jar包的META-INF/services/目录中，新建一个文件，文件名为 接口的"全限定名"。 文件内容为该接口的具体实现类的"全限定名"。

将spi所在jar放在主程序的classpath中

服务调用方用java.util.ServiceLoader，用服务接口为参数，去动态加载具体的实现类到JVM中。

## 模块搭建

### 1、新增springmvcsource-test

![image-20220514094502540](https://file.hyqup.cn/img/image-20220514094502540.png)

### 2、移入webmvc 依赖

![image-20220514095259458](https://file.hyqup.cn/img/image-20220514095259458.png)

```groovy
implementation(project(":spring-webmvc"))
```

