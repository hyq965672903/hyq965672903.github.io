---
title: 剑指源码-springmvc-(三)-HandlerMapping和HandlerAdapter细节
index_img: /img/default.png
banner_img: /img/default.png
date: 2022-05-19 22:48:06
tags:
categories:
description:
---

HandlerMapping的几种实现与HandlerAdapter几种原理及其内部细节剖析

<!-- more -->

## 初始化

> HandlerMapping和HandlerAdapter初始化流程基本一致

1、DispatcherServlet创建对象后，Tomcat调用初始化回调钩子initServletBean()

2、容器启动完成后，Spring发送事件，执行到DispatcherServlet.onRefresh()

3、onRefresh执行九大组件的初始化



HandlerMapping初始化流程（RequestMappingHandlerMapping为例）

- 创建配置中的HandlerMapping对象3种
- 启动createBean使用IOC创建容器
- 基于Spring的原理，实现了InitializingBean,容器启动后续执行afterPropertiesSet
- 拿到子容器（Web容器）的所有组件，扫描@Controller或者@RequestMapping
- 将分析到的信息放于HandlerMapping的registry对象中，以便后续使用



## HandlerMapping的实现类

- **BeanNameUrlHandlerMapping**  以bean的名字作为url路径，进行映射

- **SimpleUrlHandlerMapping** 手动配置url与handler的映射

- **RequestMappingHandlerMapping** 使用注解的形式来标识url与handler的映射,平时这种使用最多，重点分析

- **RouterFunctionMapping**  支持函数式以及webFlux相关的功能

  

### **BeanNameUrlHandlerMapping**  

### SimpleUrlHandlerMapping

### RequestMappingHandlerMapping

### RouterFunctionMapping

## HandlerAdapter的实现类

- **HttpRequestHandlerAdapter** 判断是否是实现HttpRequestHandler接口

- **SimpleControllerHandlerAdapter**判断是否实现Controller接口

- **RequestMappingHandlerAdapter** 判断是不是HandlerMethod，注解类型的都是这种

  

### HttpRequestHandlerAdapter

### SimpleControllerHandlerAdapter

### RequestMappingHandlerAdapter





