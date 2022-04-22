---
title: 剑指源码-spring(一)-序章
index_img: /img/default.png
banner_img: /img/default.png
date: 2022-04-22 19:53:11
tags:
categories:
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

