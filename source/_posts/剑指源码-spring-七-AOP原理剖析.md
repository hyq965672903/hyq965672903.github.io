---
title: 剑指源码-spring-(七)-AOP原理剖析
index_img: 'http://hyqup-blog-upyun.test.upcdn.net/img/wallhaven-odgolp.jpg'
banner_img: 'http://hyqup-blog-upyun.test.upcdn.net/img/wallhaven-nrmo9m.jpg'
categories:
  - Java
  - Spring源码
abbrlink: 5b245593
date: 2022-05-12 22:22:15
tags:
  - AOP
description:
---

本章节探究AOP的使用及其原理剖析，了解AOP的详细使用及其注意事项，探究AOP是怎样在整个Bean生命周期干预来实现的

<!-- more -->

## AOP演示

由于这里源码编译环境采用aspectJ相关，演示测试环境有点麻烦，我们这里采用springboot 新建项目引入AOP,使用SpringTest来进行演示及Debug

### 1、引入依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-aop</artifactId>
</dependency>
```

### 2、编写测试

```java
package com.hyq.practice.aop;//package cn.hyqup.spring.aop;

import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.EnableAspectJAutoProxy;

/**
 * Copyright © 2022 灼华. All rights reserved.
 *
 * @author create by hyq
 * @version 0.1
 * @date 2022/5/11
 * @description:
 */

@EnableAspectJAutoProxy //开启自动代理
@Configuration
public class AopConfig {
}
```

```java
package com.hyq.practice.aop;

import org.aspectj.lang.JoinPoint;
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.*;
import org.springframework.stereotype.Component;

import java.util.Arrays;

/**
 * Copyright © 2022 灼华. All rights reserved.
 *
 * @author create by hyq
 * @version 0.1
 * @date 2022/5/11
 * @description:
 */
@Component
@Aspect
public class LogAspect {
   public LogAspect(){
      System.out.println("LogAspect...");
   }
	/**
	 * 定义切入点：对要拦截的方法进行定义与限制，如包、类
	 *
	 * 1、execution(public * *(..)) 任意的公共方法
	 * 2、execution（* set*（..）） 以set开头的所有的方法
	 * 3、execution（* com.hyq.practice.aop.LogAspect.*（..））   com.hyq.practice.aop.LogAspect这个类里的所有的方法
	 * 4、execution（* com.hyq.practice.aop.*.*（..））   com.hyq.practice.aop包下的所有的类的所有的方法
	 * 5、execution（* com.hyq.practice.aop..*.*（..））   com.hyq.practice.aop包及子包下所有的类的所有的方法
	 * 6、execution(* com.hyq.practice.aop..*.*(String,?,Long))   com.hyq.practice.aop包及子包下所有的类的有三个参数，第一个参数为String类型，第二个参数为任意类型，第三个参数为Long类型的方法
	 * 7、execution(@annotation(xxx))
	 */
	@Pointcut("execution(* com.hyq.practice.aop.HelloService.sayHello(..))")
	private void cutMethod() {

	}



   /**
    * 前置通知：在目标方法执行前调用
    */
   @Before("cutMethod()")
   public void logStart(JoinPoint joinPoint){
      String name = joinPoint.getSignature().getName();
      System.out.println("前置通知==>"+name+"....【args: "+ Arrays.asList(joinPoint.getArgs()) +"】");
   }

   /**
    * 后置通知：在目标方法执行后调用，若目标方法出现异常，则不执行
    */
   @AfterReturning(value = "cutMethod()",returning = "result")
   public void logReturn(JoinPoint joinPoint,Object result){
      String name = joinPoint.getSignature().getName();
      System.out.println("后置通知(不发生异常执行)()==>"+name+"....【args: "+ Arrays.asList(joinPoint.getArgs()) +"】【result: "+result+"】");
   }


   /**
    * 后置/最终通知：无论目标方法在执行过程中出现一场都会在它之后调用
    */
   @After("cutMethod()")
   public void logEnd(JoinPoint joinPoint){
      String name = joinPoint.getSignature().getName();
      System.out.println("后置通知(必定执行)()==>"+name+"....【args: "+ Arrays.asList(joinPoint.getArgs()) +"】");
   }


   /**
    * 异常通知：目标方法抛出异常时执行
    */
   @AfterThrowing(value = "cutMethod()",throwing = "e")
   public void logError(JoinPoint joinPoint,Exception e){
      String name = joinPoint.getSignature().getName();
      System.out.println("异常通知()==>"+name+"....【args: "+ Arrays.asList(joinPoint.getArgs()) +"】【exception: "+e+"】");
   }

   /**
    * 环绕通知：灵活自由的在目标方法中切入代码
    */
   @Around("cutMethod()")
   public void around(ProceedingJoinPoint joinPoint) throws Throwable {
      String methodName = joinPoint.getSignature().getName();
      System.out.println("环绕方法执行前通知()==> --》 method name " + methodName + " args " + Arrays.asList(joinPoint.getArgs()));
      joinPoint.proceed();
      System.out.println("环绕方法执行后通知()==> --》 method name " + methodName + " args " + Arrays.asList(joinPoint.getArgs()));
   }
}
```

```java
package com.hyq.practice.aop;

import org.springframework.stereotype.Component;

/**
 * Copyright © 2022 灼华. All rights reserved.
 *
 * @author create by hyq
 * @version 0.1
 * @date 2022/5/11
 * @description:
 */
@Component
public class HelloService {
   public HelloService(){
      System.out.println("....");
   }

   public String sayHello(String name){
      System.out.println("Hello "+name);
      return name;
   }
}
```

```java
package com.hyq.practice;

import com.hyq.practice.aop.HelloService;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;

@SpringBootTest
class PracticeApplicationTests {
    @Autowired
    HelloService helloService;

    @Test
    void contextLoads() {
        helloService.sayHello("123");
    }

}
```

**执行结果：**

环绕方法执行前通知()==> --》 method name sayHello args [123]
前置通知==>sayHello....【args: [123]】
Hello 123
后置通知(不发生异常执行)()==>sayHello....【args: [123]】【result: 123】
后置通知(必定执行)()==>sayHello....【args: [123]】
环绕方法执行后通知()==> --》 method name sayHello args [123]

### 总结

**正常：前置=>目标方法(环绕通知)=>返回通知=>后置通知**

**异常：前置=>目标方法(环绕通知)=>异常通知=>后置通知**



## 原理剖析

1、了解AOP为生命周期条件，断点在bean12步骤里面为容器添加了什么组件，使用debug来剖析

2、AOP实现是 @EnableAspectJAutoProxy 启用AOP功能

### AOP定义阶段

给容器中利用AspectJAutoProxyRegistrar（实现了ImportBeanDefinitionRegistrar）

```java
AopConfigUtils.registerAspectJAnnotationAutoProxyCreatorIfNecessary(registry);
```

这个给容器中导入了 AnnotationAwareAspectJAutoProxyCreator 组件

AnnotationAwareAspectJAutoProxyCreator 本质上是一个BeanPostProcessor。

容器刷新第五步 invokeBeanFactoryPostProcessors(beanFactory); 的时候会将AnnotationAwareAspectJAutoProxyCreator注册到容器中去，这里面 ReflectiveAspectJAdvisorFactory (创建增强器的工厂) 和 BeanFactoryAspectJAdvisorsBuilderAdapter（增强器适配器建造者）把这些工具准备好。

后续在正常创建其他对象的时候干预，判断是切面相关 执行BeanPostProcessor 里面的逻辑来增强



在创建其他Bean的时候（**第一次**）,回去遍历所有类来构建增强器 aspectJAdvisorsBuilder.buildAspectJAdvisors()

### 构建增强器的过程

ps: 案例中这个时候会创建 LogAspect 相关的生成增强器

aspectJAdvisorsBuilder.buildAspectJAdvisors()

```java
public List<Advisor> buildAspectJAdvisors() {
   List<String> aspectNames = this.aspectBeanNames;

   if (aspectNames == null) {
      synchronized (this) {
         aspectNames = this.aspectBeanNames;
         if (aspectNames == null) {
            List<Advisor> advisors = new ArrayList<>();
            aspectNames = new ArrayList<>();
            String[] beanNames = BeanFactoryUtils.beanNamesForTypeIncludingAncestors(
                  this.beanFactory, Object.class, true, false);
            for (String beanName : beanNames) {
               if (!isEligibleBean(beanName)) {
                  continue;
               }
               // We must be careful not to instantiate beans eagerly as in this case they
               // would be cached by the Spring container but would not have been weaved.
               Class<?> beanType = this.beanFactory.getType(beanName, false);
               if (beanType == null) {
                  continue;
               }
               if (this.advisorFactory.isAspect(beanType)) {
                  aspectNames.add(beanName);
                  AspectMetadata amd = new AspectMetadata(beanType, beanName);
                  if (amd.getAjType().getPerClause().getKind() == PerClauseKind.SINGLETON) {
                     MetadataAwareAspectInstanceFactory factory =
                           new BeanFactoryAspectInstanceFactory(this.beanFactory, beanName);
                      //得到增强器
                     List<Advisor> classAdvisors = this.advisorFactory.getAdvisors(factory);
                     if (this.beanFactory.isSingleton(beanName)) {
                        this.advisorsCache.put(beanName, classAdvisors);
                     }
                     else {
                        this.aspectFactoryCache.put(beanName, factory);
                     }
                     advisors.addAll(classAdvisors);
                  }
                  else {
                     // Per target or per this.
                     if (this.beanFactory.isSingleton(beanName)) {
                        throw new IllegalArgumentException("Bean with name '" + beanName +
                              "' is a singleton, but aspect instantiation model is not singleton");
                     }
                     MetadataAwareAspectInstanceFactory factory =
                           new PrototypeAspectInstanceFactory(this.beanFactory, beanName);
                     this.aspectFactoryCache.put(beanName, factory);
                     advisors.addAll(this.advisorFactory.getAdvisors(factory));
                  }
               }
            }
            this.aspectBeanNames = aspectNames;
            return advisors;
         }
      }
   }

   if (aspectNames.isEmpty()) {
      return Collections.emptyList();
   }
   List<Advisor> advisors = new ArrayList<>();
   for (String aspectName : aspectNames) {
      List<Advisor> cachedAdvisors = this.advisorsCache.get(aspectName);
      if (cachedAdvisors != null) {
         advisors.addAll(cachedAdvisors);
      }
      else {
         MetadataAwareAspectInstanceFactory factory = this.aspectFactoryCache.get(aspectName);
         advisors.addAll(this.advisorFactory.getAdvisors(factory));
      }
   }
   return advisors;
}
```

构建好了会放在advisorsCache 缓存中去

得到增强器 this.advisorFactory.getAdvisors(factory);

```java
@Override
public List<Advisor> getAdvisors(MetadataAwareAspectInstanceFactory aspectInstanceFactory) {
   Class<?> aspectClass = aspectInstanceFactory.getAspectMetadata().getAspectClass();
   String aspectName = aspectInstanceFactory.getAspectMetadata().getAspectName();
   validate(aspectClass);

   // We need to wrap the MetadataAwareAspectInstanceFactory with a decorator
   // so that it will only instantiate once.
   MetadataAwareAspectInstanceFactory lazySingletonAspectInstanceFactory =
         new LazySingletonAspectInstanceFactoryDecorator(aspectInstanceFactory);

   List<Advisor> advisors = new ArrayList<>();// 收集到所有的增强器
   for (Method method : getAdvisorMethods(aspectClass)) {
      // Prior to Spring Framework 5.2.7, advisors.size() was supplied as the declarationOrderInAspect
      // to getAdvisor(...) to represent the "current position" in the declared methods list.
      // However, since Java 7 the "current position" is not valid since the JDK no longer
      // returns declared methods in the order in which they are declared in the source code.
      // Thus, we now hard code the declarationOrderInAspect to 0 for all advice methods
      // discovered via reflection in order to support reliable advice ordering across JVM launches.
      // Specifically, a value of 0 aligns with the default value used in
      // AspectJPrecedenceComparator.getAspectDeclarationOrder(Advisor).
       // 创建增强器
      Advisor advisor = getAdvisor(method, lazySingletonAspectInstanceFactory, 0, aspectName);
      if (advisor != null) {
         advisors.add(advisor);
      }
   }

   // If it's a per target aspect, emit the dummy instantiating aspect.
   if (!advisors.isEmpty() && lazySingletonAspectInstanceFactory.getAspectMetadata().isLazilyInstantiated()) {
      Advisor instantiationAdvisor = new SyntheticInstantiationAdvisor(lazySingletonAspectInstanceFactory);
      advisors.add(0, instantiationAdvisor);
   }

   // Find introduction fields.
   for (Field field : aspectClass.getDeclaredFields()) {
      Advisor advisor = getDeclareParentsAdvisor(field);
      if (advisor != null) {
         advisors.add(advisor);
      }
   }

   return advisors;
}
```

### 干扰切入点生成代理对象

ps:干扰需要切入的对象的HelloService创建过程，会生成相关代理对象，并转为拦截器链，以便于后续执行过程

容器刷新12大步之 finishBeanFactoryInitialization(beanFactory);里面 beanFactory.preInstantiateSingletons(); 完成对剩下单实例创建的过程中

创建过程中BeanPostProcessor.postProcessAfterInitialization AOP开始增强，这个过程的时候bean已经生成，初始化的时候进行干预Bean

代码位置AbstractAutoProxyCreator.postProcessAfterInitialization

```java
@Override
public Object postProcessAfterInitialization(@Nullable Object bean, String beanName) {
   if (bean != null) {
      Object cacheKey = getCacheKey(bean.getClass(), beanName);
      if (this.earlyProxyReferences.remove(cacheKey) != bean) {
         return wrapIfNecessary(bean, beanName, cacheKey);
      }
   }
   return bean;
}

```



这里会去找当前bean 的增强器，采用MethodMatcher通过正则表达式匹配切入点和目标类的方法。匹配完拿到目标类的增强器的时候，会执行extendAdvisors 创建一个增强器链，在第0位增加一个ExposeInvocationInterceptor拦截器（方法拦截器），接下来拿到了增强器就为当前目标类创建代理，创建时候判断当前对象是否有接口实现，**如果有则使用jdk动态代理创建，如果没有接口实现则采用CGlib来进行创建代理**

```java
protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
   if (StringUtils.hasLength(beanName) && this.targetSourcedBeans.contains(beanName)) {
      return bean;
   }
   if (Boolean.FALSE.equals(this.advisedBeans.get(cacheKey))) {
      return bean;
   }
   if (isInfrastructureClass(bean.getClass()) || shouldSkip(bean.getClass(), beanName)) {
      this.advisedBeans.put(cacheKey, Boolean.FALSE);
      return bean;
   }

   // Create proxy if we have advice.
   Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);
   if (specificInterceptors != DO_NOT_PROXY) {
      this.advisedBeans.put(cacheKey, Boolean.TRUE);
      Object proxy = createProxy(
            bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));
      this.proxyTypes.put(cacheKey, proxy.getClass());
      return proxy;
   }

   this.advisedBeans.put(cacheKey, Boolean.FALSE);
   return bean;
}
```

### 目标方法的执行

当目标方法执行的时候，如helloService.sayHello。这个时候会来到这个代理类设置的回调（DynamicAdvisedInterceptor）里面执行intercept。执行期间将之前的增强器通过getInterceptorsAndDynamicInterceptionAdvice转化为方法拦截器 执行invoke 的时候，链式调用。

代理对象（HelloService）包含拦截器，拦截器中包含增强器，执行的过程中会执行过滤器链，**责任链模式**

- ExposeInvocationInterceptor 线程共享数据

- MethodBeforeAdviceInterceptor 前置通知拦截器

- MethodBeforeAdvice后置通知拦截器

- AfterReturningAdviceInterceptor返回通知拦截器

- AspectJAfterThrowingAdvice异常通知拦截器

  拦截器链执行细节就不展开了

