---
title: 剑指源码-Mybatis-(一)-序章
index_img: 'https://tva4.sinaimg.cn/large/9bd9b167ly1fwsfchsldlj21hc0u0x2k.jpg'
banner_img: 'https://tva1.sinaimg.cn/large/9bd9b167gy1g4li06ofsnj21hc0xc4qp.jpg'
abbrlink: 2569fe28
date: 2022-07-18 20:23:03
tags:
categories:
 - Java
 - Mybatis源码
description:
---

 Mybatis介绍，起源。常见的ORM框架，为什么选择Mybatis，搭建Mybatis源码环境条件

<!-- more -->

## 介绍

> MyBatis 是 Java 生态中非常著名的一款 ORM 框架

## JDBC流程

`Java DataBase Connectivity`  Java 程序与关系型数据库交互的统一 API

JDBC 由两部分 API 构成：第一部分是面向 Java 开发者的 Java API，它是一个统一的、标准的 Java API，独立于各个数据库产品的接口规范；第二部分是面向数据库驱动程序开发者的 API，它是由各个数据库厂家提供的数据库驱动，是第一部分接口规范的底层实现，用于连接具体的数据库产品

### JDBC 操作的核心步骤

1. 注册数据库驱动类，指定数据库地址，其中包括 DB 的用户名、密码及其他连接信息
2. 调用 DriverManager.getConnection() 方法创建 Connection 连接到数据库
3. 调用 Connection 的 createStatement() 或 prepareStatement() 方法，创建 Statement 对象，此时会指定 SQL（或是 SQL 语句模板 + SQL 参数）
4. 通过 Statement 对象执行 SQL 语句，得到 ResultSet 对象，也就是查询结果集
5. 遍历 ResultSet，从结果集中读取数据，并将每一行数据库记录转换成一个 JavaBean 对象
6. 关闭 ResultSet 结果集、Statement 对象及数据库 Connection，从而释放这些对象占用的底层资源

## 常见ORM框架对比

### Hibernate

Hibernate 是 Java 生态中著名的 ORM 框架之一。Hibernate 现在也在扩展自己的生态，开始支持多种异构数据的持久化，不仅仅提供 ORM 框架，还提供了 Hibernate Search 来支持全文搜索，提供 validation 来进行数据校验，提供 Hibernate OGM 来支持 NoSQL 解决方案。

### Spring Data JPA

> Spring Data JPA 是符合 JPA 规范的一个 Repository 层的实现

JPA 是在 JDK 5.0 后提出的 Java 持久化规范（JSR 338）。JPA 规范本身是为了整合市面上已有的 ORM 框架，结束 Hibernate、EclipseLink、JDO 等 ORM 框架各自为战的割裂局面，简化 Java 持久层开发

JPA 有三个核心部分：**ORM 映射元数据**、**操作实体对象 API** 和**面向对象的查询语言（JPQL）**

### Mybatis

优点：MyBatis 相较于 Hibernate 和各类 JPA 实现框架更加灵活、更加轻量级、更加可控。

我们可以在 MyBatis 的 Mapper 映射文件中，直接编写原生的 SQL 语句，应用底层数据库产品的方言，这就给了我们直接优化 SQL 语句的机会。我们还可以按照数据库的使用规则，让原生 SQL 语句选择我们期望的索引，从而保证服务的性能。

在编写原生 SQL 语句时，我们也能够更加方便地控制结果集中的列，而不是查询所有列并映射对象后返回，这在列比较多的时候也能起到一定的优化效果

缺点：sql 需要更加仔细的调试


## 选择Mybatis源码阅读原因

- MyBatis 本身就是一款设计非常精良、架构设计非常清晰的持久层框架，并且 MyBatis 中还使用到了很多经典的设计模式，例如，工厂方法模式、适配器模式、装饰器模式、代理模式等。
- MyBatis 提供了很多扩展点，例如，MyBatis 的插件机制、对第三方日志框架和第三方数据源的兼容等
- 开发人员使用 MyBatis 上手会非常快，具有很强的易用性和可靠性

## 源码搭建

下载源代码 git clone  https://github.com/mybatis/mybatis-3.git

从标签版本切换分支出来 `git branch mybatis3 mybatis-3.5.9`

### Mybatis 架构简介

![img](https://file.hyqup.cn/img/webp.webp)

<center>图片来自网络</center>

1. API接口层
2. 数据处理层
3. 基础支撑层

#### 接口层

> 提供给外部使用的接口API，开发人员通过这些本地API来操纵数据库

 MyBatis 会根据接口声明的方法信息，通过**动态代理机制生成一个Mapper 实例**，当调用接口方法时，根据这个方法的方法名和参数类型，确定Statement Id，底层还是通过 SqlSession.select/update( “statementId”, parameter) 等来实现对数据库的操作

### 数据处理层

> MyBatis 的核心，负责具体的SQL查找、SQL解析、SQL执行和执行结果映射处理等，它主要的目的是根据调用的请求完成一次数据库操作

主要功能

- 通过传入参数构建动态SQL语句
- SQL语句的执行以及封装查询结果集

### 基础支撑层

基础支撑层是整个MyBatis框架的地基，负责最基础的功能支撑，共有九大基础工具

![ title=](https://file.hyqup.cn/img/1460000041398816.png)

<center>图片来自网络</center>

## 核心组件

`SqlSession` 作为MyBatis工作的主要顶层API，表示和数据库交互的会话，完成必要数据库增删改查功能

`Executor` MyBatis执行器，是MyBatis 调度的核心，负责SQL语句的生成和查询缓存的维护

`StatementHandler` 封装了JDBC Statement操作，负责对JDBC statement 的操作，如设置参数、将Statement结果集转换成List集合。

`ParameterHandler` 负责对用户传递的参数转换成JDBC Statement 所需要的参数，

`ResultSetHandler` 负责将JDBC返回的ResultSet结果集对象转换成List类型的集合；

`TypeHandler` 负责java数据类型和jdbc数据类型之间的映射和转换

`MappedStatement` MappedStatement维护了一条`<select|update|delete|insert>`节点的封装，

`SqlSource` 负责根据用户传递的parameterObject，动态地生成SQL语句，将信息封装到BoundSql对象中，并返回

`BoundSql` 表示动态生成的SQL语句以及相应的参数信息

`Configuration` MyBatis所有的配置信息都维持在Configuration对象之中。
