---
title: 剑指源码-Mybatis-(二)-主流程分析与核心组件关系
index_img: 'https://tva2.sinaimg.cn/large/9bd9b167ly1fwrt21udjcj21hc0u0tkq.jpg'
banner_img: 'https://tva3.sinaimg.cn/large/9bd9b167gy1g2ql4y760jj21hc0u07wh.jpg'
abbrlink: 597465dd
date: 2022-08-11 20:54:29
tags:
categories:
 - Java
 - Mybatis源码
description:
---

 demo整体流程概述，分析核心组件的作用以及关系

<!-- more -->

> demo地址：https://github.com/hyq965672903/sourcecode-learn-mybatis.git

```java
    /**
     * 使用MyBatis API方式
     * @throws IOException
     */
    @Test
    public void test1() throws IOException {
        String resource = "mybatis-config.xml";
        InputStream inputStream = Resources.getResourceAsStream(resource);
        SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream);

        SqlSession session = sqlSessionFactory.openSession();
        try {
            User user = (User) session.selectOne("cn.hyqup.mapper.UserMapper.selectAll", 1);
            System.out.println(user);
        } finally {
            session.close();
        }
    }
    /**
     * 通过 SqlSession.getMapper(XXXMapper.class)  接口方式
     */
    @Test
    public void test2() throws IOException {
        String resource = "mybatis-config.xml";
        InputStream inputStream = Resources.getResourceAsStream(resource);
        SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream);

        SqlSession session = sqlSessionFactory.openSession(); // ExecutorType.BATCH
        try {
            UserMapper mapper = session.getMapper(UserMapper.class);

            List<User> users = mapper.selectAll();
            System.out.println(users);
        } finally {
            session.close();
        }
    }
```

## 整体流程分析

>  通过 SqlSession.getMapper(XXXMapper.class)  接口方式为例，流程分析

###  创建SqlSessionFactory的过程

1. 首先根据mybatis-config.xml创建一个`inputStream`

2. 然后new一个SqlSessionFactoryBuilder对象，使用inputStream调用build方法得到`SqlSessionFactory`对象

   ```java
   public SqlSessionFactory build(InputStream inputStream, String environment, Properties properties) {
     try {
       XMLConfigBuilder parser = new XMLConfigBuilder(inputStream, environment, properties);
       return build(parser.parse());
     } catch (Exception e) {
       throw ExceptionFactory.wrapException("Error building SqlSession.", e);
     } finally {
       ErrorContext.instance().reset();
       try {
          if (inputStream != null) {
            inputStream.close();
          }
       } catch (IOException e) {
         // Intentionally ignore. Prefer previous error.
       }
     }
   }
   ```

3. `XMLConfigBuilder`创建的parser解析器对象，会去解析每一个xml节点信息，对应的相关配置信息保存在一个`Configuration`对象中

4. 同时这个时候会解析到mapper信息，`XMLConfigBuilder`内部构建了一个`XMLMapperBuilder`，`XMLMapperBuilder`对象会将mappper信息解析到`Configuration`的`mappedStatements`这个Map对象中。**而这个`mappedStatements`包含多个`MappedStatement`，`MappedStatement`中包含了mapper中每一个标签信息**

5. 解析完mapper之后，将mapper的信息放入`Configuration`对象中，返回`Configuration`对象

6. build中传入的Configuration对象被用来创建了一个`DefaultSqlSessionFactory`对象。<u>**创建sqlSessionFactory所做的工作就是，将所有配置文件的信息解析并且保存在Configuration对象中，返回包含了Configuration对象的DefaultSqlSessionFactory对象。**</u>

### 获取sqlSession对象的过程

1. 首先调用`DefaultSqlSessionFactory`的`openSession`()方法
2. openSession里面调用`openSessionFromDataSource`(configuration.getDefaultExecutorType(), null, false)方法
3. `openSessionFromDataSource`里面会创建Transaction一个事务，同时会创建一个Executor对象
4. 根据`Executor`在全局配置中的类型（目前类型有3种，simple，batch，reuse），创建出SimpleExecutor、`ReuseExecutor`或者`BatchExecutor`。Executor是一个接口，里面规定了一些增删改查的接口方法。默认就是simple类型
5. 如果开启了二级缓存，会通过CacheExecutor对Executor进行包装，CacheExecutor本质就是**装饰者模式**
6. executor = (Executor) interceptorChain.pluginAll(executor)，pluginAll方法会拿到每一个拦截器的plugin方法包装一下executor，开发插件就是这里织入
7. 最后创建DefaultSqlSession并返回

```java
  private SqlSession openSessionFromDataSource(ExecutorType execType, TransactionIsolationLevel level, boolean autoCommit) {
    Transaction tx = null;
    try {
      final Environment environment = configuration.getEnvironment();
      final TransactionFactory transactionFactory = getTransactionFactoryFromEnvironment(environment);
      tx = transactionFactory.newTransaction(environment.getDataSource(), level, autoCommit);
        //这里去创建Executor
      final Executor executor = configuration.newExecutor(tx, execType);
      return new DefaultSqlSession(configuration, executor, autoCommit);
    } catch (Exception e) {
      closeTransaction(tx); // may have fetched a connection so lets call close()
      throw ExceptionFactory.wrapException("Error opening session.  Cause: " + e, e);
    } finally {
      ErrorContext.instance().reset();
    }
  }
```

创建Executor代码逻辑

```java
public Executor newExecutor(Transaction transaction, ExecutorType executorType) {
  executorType = executorType == null ? defaultExecutorType : executorType;
  executorType = executorType == null ? ExecutorType.SIMPLE : executorType;
  Executor executor;
  if (ExecutorType.BATCH == executorType) {
    executor = new BatchExecutor(this, transaction);
  } else if (ExecutorType.REUSE == executorType) {
    executor = new ReuseExecutor(this, transaction);
  } else {
    executor = new SimpleExecutor(this, transaction);
  }
  if (cacheEnabled) {
    executor = new CachingExecutor(executor);
  }
  executor = (Executor) interceptorChain.pluginAll(executor);
  return executor;
}
```

### 获取Mapper的代理对象（MapperProxy）的过程

1. 调用`DefaultSqlSession`对象的getMapper(type)方法。（type是Mapper接口的class，例如XXXMapper.class）
2. 进一步里面会调用`configuration`.getMapper(type,sqlSession),调用`configuration`对象中`mapperRegistry`的getMapper
3. 然后根据class类型获取`MapperProxyFactory`，调用其`newInstance`，进而使用`MapperProxy`动态代理创建代理对象，并返回

### 执行增删改查方法的过程

1. `Mapper`都是接口，所以最终执行都会执行到`MapperProxy`的`invoke`方法
2. `MapperProxy`的invoke会执行`MapperMethod`的execute，execute方法会判断当前要执行的sql语句的类型
3. 当前是selectAll，于是执行到`executeForMany`方法，进而`sqlSession`执行到selectList方法
4. selectList方法中获取`MappedStatement`对象，调用`executor`(CachingExecutor)执行query
5. 获取BoundSql对象，里面包含sql语句的详细信息。sql语句是什么、sql语句的参数是什么等详细信息
6. 调用真正的`simpleExecutor`的query方法进行查询，query是`BaseExecutor`中的，真正执行到doQuery方法
7. doQuery里面会创建`StatementHandler`，默认就是`PreparedStatementHandler`。`StatementHandler`被用于创建Statement对象。同时这里`StatementHandler`会执行interceptorChain的pluginAll进行插件加载
8. 在`StatementHandler`会调用父类`BaseStatementHandler`的构造方法进一步创建`ParameterHandler`和`ResultSetHandler`
   1. `ParameterHandler`用于处理参数设置的 ，设置sql语句的预编译参数
   2. `ResultSetHandler`用于处理结果返回封装的

9. `ParameterHandler`和`ResultsetHandler`都会借助`TypeHandler`来进行数据库类型和Java Bean类型的转换
10. `ParameterHandler`设置参数的时候是调用的`TypeHandler`给sql预编译设置参数
11. 查出数据之后，使用`ResultHandler`处理结果，里面也是使用`TypeHandler`包装value值
12. 后续操作完成数据库连接相关
13. 流程结束

> **Executor、ParameterHandler、ResultsetHandler、TypeHandler四大对象**之间的关系
>
> Executor用于执行增删改查操作；StatementHandler用ParameterHandler设置参数；使用ResultsetHandler处理结果；过程中又采用TypeHandler来实现数据库类型和JavaBean类型之间的转换





