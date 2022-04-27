---
title: 剑指源码-spring-(五)-Bean初始化流程
index_img: /img/default.png
banner_img: /img/default.png
date: 2022-04-27 21:51:19
tags:
categories:
description:
---

 Bean的初始化流程，Spring容器中获取任何一个Bean都是采用getBean的流程，本章节详细了解getBean到底干了什么事，走了什么流程

<!-- more -->

spring在刷新容器的12大步骤的第11步的时候 finishBeanFactoryInitialization(beanFactory);而 finishBeanFactoryInitialization里面的beanFactory.preInstantiateSingletons();便是入口函数.

初始化所有非懒加载的Bean(preInstantiateSingletons)

## 整体流程分析图

![Bean初始化流程.drawio](https://file.hyqup.cn/img/Bean%E5%88%9D%E5%A7%8B%E5%8C%96%E6%B5%81%E7%A8%8B.drawio.png)
