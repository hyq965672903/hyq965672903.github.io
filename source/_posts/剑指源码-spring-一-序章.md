---
title: 剑指源码-spring(一)-序章
index_img: 'https://file.hyqup.cn/img/wallhaven-0pgw19.jpg'
banner_img: 'https://file.hyqup.cn/img/wallhaven-g7gvo3.png'
categories:
  - Java
  - Spring源码
abbrlink: 2db10d0a
date: 2022-04-22 19:53:11
tags:
description:
---

 快速了解spring IOC容器的启动流程

<!-- more -->

这里以我们开发常用的注解形式

1. 创建AnnotationConfigApplicationContext 对象，这个对象继承于GenericApplicationContext，而GenericApplicationContext对象中有DefaultListableBeanFactory；
2. 创建 DefaultListableBeanFactory之后会解析和扫描注解上的所有bean信息封装成BeanDefinition信息，BeanDefinition中有Spring的所有bean的相关定义描述信息；
3. 先将bean进行实例化，然后根据BeanDefinition的信息，底层采用反射的信息进行创建具体的bean对象；
4. 将实例化后的bean对象放到一个Map中，后续应用就从Map中获取Bean对象，该Map也就是我们的IOC容器



ps：其中有许多细节都没说，比如这里的Map 存在多个，解决了什么问题，创建的过程细节等等，接下来就开是进入源码解析的具体学习



#  准备工作

## 源码版本

  github地址  

https://github.com/spring-projects/spring-framework/releases/tag/v5.3.10

源码分析地址

https://github.com/hyq965672903/sourcecode-learn-spring.git

相关环境：

```text
1、操作系统：Windows10
2、JDK 版本：Jdk11
3、IDE 工具：IntelliJ IDEA 2021.2
	idea对应kotlin 编译版本 1.5.10
4、项目构建工具：gradle-7.2
5、spring 版本 5.3.10 
	当前spring kotlin 对应版本是1.5.30（根目录build.gradle中 搜 kotlin.jvm）
```

ps ：要注意当前的idea kotlin 插件版本和当前源码对应的kotlin版本，因为编译需要kotlin ，所以可能报错



## 遇到的问题及解决方法

1、配置环境变量  GRADLE_USER_HOME

类似于maven 的本地仓库地址 

![image-20220423085321489](https://file.hyqup.cn/img/image-20220423085321489.png)

2、IDEA gradle 配置说明

spring源码下gradle 文件夹->wrapper文件夹的gradle-wrapper.properties如下

```groovy
distributionBase=GRADLE_USER_HOME
distributionPath=wrapper/dists
distributionUrl=https\://services.gradle.org/distributions/gradle-7.2-bin.zip
zipStoreBase=GRADLE_USER_HOME
zipStorePath=wrapper/dists

```

IDEA需要配置一些东西来保证使用该wrapper 构建 如果不行 **~~请删除 gradle user  home~~**

![image-20220423123042199](https://file.hyqup.cn/img/image-20220423123042199.png)

3、根目录build.gradle 加入阿里云镜像仓库

![image-20220423123220705](https://file.hyqup.cn/img/image-20220423123220705.png)

```groovy
    maven {
                url 'https://maven.aliyun.com/repository/central'
            }
```

# Spring框架的整体流程

![Spring架构原理图](https://file.hyqup.cn/img/Spring%E6%9E%B6%E6%9E%84%E5%8E%9F%E7%90%86%E5%9B%BE.jpg)

> 图源自于 雷丰阳-设计模式

整体流程分三大块

1、首先从各个环境读取bean定义信息，可以从本地xml,注解，或者网络、磁盘读取到文档信息Document

2、通过BeanDefinitionRegistry将信息读取为BeanDefinition 放入DefaultListableBeanFactory的beanDefinitionMap对象中去

3、通过BeanDefinition的Bean定义信息来创建我们所需要的对象，创建过程十分复杂

## 核心接口

### 基础接口

- Resource ResourceLoader 
- BeanFactory
- BeanDefinition
- BeanDefinitionReader
- BeanDefinitionRegistry
- ApplicationContext
- Aware

### 生命周期-后置处理器

- BeanFactoryPostProcessor
- InitializingBean
- BeanPostProcessor

![image-20220423161153634](%E5%89%91%E6%8C%87%E6%BA%90%E7%A0%81-spring-%E4%B8%80-%E5%BA%8F%E7%AB%A0.assets/image-20220423161153634.png)



日常开发使用注解开发比较多，其核心就是 **AnnotationConfigApplicationContext** 作为入口进行对象解析开始直接流程的





![image-20220423162300061](https://file.hyqup.cn/img/image-20220423162300061.png)

GenericApplicationContext ：
private final DefaultListableBeanFactory beanFactory; 

GenericApplicationContext 里面存在 DefaultListableBeanFactory  ，而DefaultListableBeanFactory  本质上又是一个档案馆，也就是第一大步骤获取到的bean定义信息等各种核心信息都会存到这里，后续IOC容器创建的bean信息来源基本上都来自于这里。

上面的含义也就是  AnnotationConfigApplicationContext 组合了档案馆信息
