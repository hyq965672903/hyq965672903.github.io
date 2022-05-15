---
title: 剑指源码-spring(三)-Bean初始化过程引出BeanPostProcessor
index_img: 'https://file.hyqup.cn/img/wallhaven-01w1o1.jpg'
banner_img: 'https://file.hyqup.cn/img/wallhaven-01mqk3.jpg'
categories:
  - Java
  - Spring源码
abbrlink: 8f18475c
date: 2022-04-24 20:32:14
tags:
description:
---

 堆栈分析简单的了解下Bean初始化的过程，以xxxAware为例分析，引出BeanPostProcessor的概念

<!-- more -->

## 准备工作

```java
@Data
@Component
public class Person  implements ApplicationContextAware {

   private ApplicationContext applicationContext;
   private String name;


   public Person() {
      System.out.println("person 被创建...");
   }

   @Override
   public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
      //  利用回调机制，将IOC注入进来
      this.applicationContext=applicationContext;
   }
}
```

```java
@Configuration
public class MainConfig {

   @Bean
   public Person person() {
      Person person = new Person();
      person.setName("李四");
      return person;
   }
}
```

```java
public class AnnotationMainTest {
   public static void main(String[] args) {
      ApplicationContext applicationContext = new AnnotationConfigApplicationContext(MainConfig.class);
      Person bean = applicationContext.getBean(Person.class);
      System.out.println(bean);
   }
}
```

注意断点位置

![image-20220424205139520](https://file.hyqup.cn/img/image-20220424205139520.png)

堆栈追踪分析

![image-20220424205250721](https://file.hyqup.cn/img/image-20220424205250721.png)

## 堆栈分析

###  AnnotationConfigApplicationContext  构造函数

```java
public AnnotationConfigApplicationContext(Class<?>... componentClasses) {
   this();
   register(componentClasses);
   //刷新工厂
   refresh();
}
```

### AbstractApplicationContext  refresh（）

```java
@Override
public void refresh() throws BeansException, IllegalStateException {
   synchronized (this.startupShutdownMonitor) {
      StartupStep contextRefresh = this.applicationStartup.start("spring.context.refresh");

      // Prepare this context for refreshing.
      prepareRefresh();

      // Tell the subclass to refresh the internal bean factory.
      //告诉子类刷新内部 bean 工厂。
      ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

      // Prepare the bean factory for use in this context.
      prepareBeanFactory(beanFactory);

      try {
         // Allows post-processing of the bean factory in context subclasses.
         postProcessBeanFactory(beanFactory);

         StartupStep beanPostProcess = this.applicationStartup.start("spring.context.beans.post-process");
         // Invoke factory processors registered as beans in the context.
         invokeBeanFactoryPostProcessors(beanFactory);

         // Register bean processors that intercept bean creation.
         registerBeanPostProcessors(beanFactory);
         beanPostProcess.end();

         // Initialize message source for this context.
         initMessageSource();

         // Initialize event multicaster for this context.
         initApplicationEventMulticaster();

         // Initialize other special beans in specific context subclasses.
         onRefresh();

         // Check for listener beans and register them.
         registerListeners();

         // Instantiate all remaining (non-lazy-init) singletons.
         // 完成Bean 工厂的初始化
         finishBeanFactoryInitialization(beanFactory);

         // Last step: publish corresponding event.
         finishRefresh();
      }

      catch (BeansException ex) {
         if (logger.isWarnEnabled()) {
            logger.warn("Exception encountered during context initialization - " +
                  "cancelling refresh attempt: " + ex);
         }

         // Destroy already created singletons to avoid dangling resources.
         destroyBeans();

         // Reset 'active' flag.
         cancelRefresh(ex);

         // Propagate exception to caller.
         throw ex;
      }

      finally {
         // Reset common introspection caches in Spring's core, since we
         // might not ever need metadata for singleton beans anymore...
         resetCommonCaches();
         contextRefresh.end();
      }
   }
}
```

refresh  会调用finishBeanFactoryInitialization 完成bean工厂的初始化

### AbstractApplicationContext finishBeanFactoryInitialization()

```java
protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {
   // Initialize conversion service for this context.
   if (beanFactory.containsBean(CONVERSION_SERVICE_BEAN_NAME) &&
         beanFactory.isTypeMatch(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class)) {
      beanFactory.setConversionService(
            beanFactory.getBean(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class));
   }

   // Register a default embedded value resolver if no BeanFactoryPostProcessor
   // (such as a PropertySourcesPlaceholderConfigurer bean) registered any before:
   // at this point, primarily for resolution in annotation attribute values.
   if (!beanFactory.hasEmbeddedValueResolver()) {
      beanFactory.addEmbeddedValueResolver(strVal -> getEnvironment().resolvePlaceholders(strVal));
   }

   // Initialize LoadTimeWeaverAware beans early to allow for registering their transformers early.
   String[] weaverAwareNames = beanFactory.getBeanNamesForType(LoadTimeWeaverAware.class, false, false);
   for (String weaverAwareName : weaverAwareNames) {
      getBean(weaverAwareName);
   }

   // Stop using the temporary ClassLoader for type matching.
   beanFactory.setTempClassLoader(null);

   // Allow for caching all bean definition metadata, not expecting further changes.
   beanFactory.freezeConfiguration();

   // Instantiate all remaining (non-lazy-init) singletons.
   // 初始化所有的单实例bean
   beanFactory.preInstantiateSingletons();
}
```

beanFactory.preInstantiateSingletons() 初始化所有单实例bean ,这里是由 ConfigurableListableBeanFactory的子类**<u>DefaultListableBeanFactory</u>** 来实现

```java
@Override
public void preInstantiateSingletons() throws BeansException {
   if (logger.isTraceEnabled()) {
      logger.trace("Pre-instantiating singletons in " + this);
   }

   // Iterate over a copy to allow for init methods which in turn register new bean definitions.
   // While this may not be part of the regular factory bootstrap, it does otherwise work fine.
   List<String> beanNames = new ArrayList<>(this.beanDefinitionNames);

   // Trigger initialization of all non-lazy singleton beans...
   for (String beanName : beanNames) {
      RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
      if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {
         if (isFactoryBean(beanName)) {
            Object bean = getBean(FACTORY_BEAN_PREFIX + beanName);
            if (bean instanceof FactoryBean) {
               FactoryBean<?> factory = (FactoryBean<?>) bean;
               boolean isEagerInit;
               if (System.getSecurityManager() != null && factory instanceof SmartFactoryBean) {
                  isEagerInit = AccessController.doPrivileged(
                        (PrivilegedAction<Boolean>) ((SmartFactoryBean<?>) factory)::isEagerInit,
                        getAccessControlContext());
               }
               else {
                  isEagerInit = (factory instanceof SmartFactoryBean &&
                        ((SmartFactoryBean<?>) factory).isEagerInit());
               }
               if (isEagerInit) {
                  getBean(beanName);
               }
            }
         }
         else {
            getBean(beanName);
         }
      }
   }
```

getBean 由父类AbstractBeanFactory 实现，并且会继续调用doGetBean

### *AbstractBeanFactory doGetBean() 

doGetBean 非常重要且这里的逻辑比较多，重点分析

```java
protected <T> T doGetBean(
      String name, @Nullable Class<T> requiredType, @Nullable Object[] args, boolean typeCheckOnly)
      throws BeansException {
   // 转换bean的名字
   String beanName = transformedBeanName(name);
   Object beanInstance;

   // Eagerly check singleton cache for manually registered singletons.
   //检查单例缓存以获取手动注册的单例
   Object sharedInstance = getSingleton(beanName);
   if (sharedInstance != null && args == null) {
      if (logger.isTraceEnabled()) {
         if (isSingletonCurrentlyInCreation(beanName)) {
            logger.trace("Returning eagerly cached instance of singleton bean '" + beanName +
                  "' that is not fully initialized yet - a consequence of a circular reference");
         }
         else {
            logger.trace("Returning cached instance of singleton bean '" + beanName + "'");
         }
      }
      beanInstance = getObjectForBeanInstance(sharedInstance, name, beanName, null);
   }

   else {
      // Fail if we're already creating this bean instance:
      // We're assumably within a circular reference.
      if (isPrototypeCurrentlyInCreation(beanName)) {
         throw new BeanCurrentlyInCreationException(beanName);
      }

      // Check if bean definition exists in this factory.
      BeanFactory parentBeanFactory = getParentBeanFactory();
      if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
         // Not found -> check parent.
         String nameToLookup = originalBeanName(name);
         if (parentBeanFactory instanceof AbstractBeanFactory) {
            return ((AbstractBeanFactory) parentBeanFactory).doGetBean(
                  nameToLookup, requiredType, args, typeCheckOnly);
         }
         else if (args != null) {
            // Delegation to parent with explicit args.
            return (T) parentBeanFactory.getBean(nameToLookup, args);
         }
         else if (requiredType != null) {
            // No args -> delegate to standard getBean method.
            return parentBeanFactory.getBean(nameToLookup, requiredType);
         }
         else {
            return (T) parentBeanFactory.getBean(nameToLookup);
         }
      }

      if (!typeCheckOnly) {
         markBeanAsCreated(beanName);
      }

      StartupStep beanCreation = this.applicationStartup.start("spring.beans.instantiate")
            .tag("beanName", name);
      try {
         if (requiredType != null) {
            beanCreation.tag("beanType", requiredType::toString);
         }
         RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
         checkMergedBeanDefinition(mbd, beanName, args);

         // Guarantee initialization of beans that the current bean depends on.
         String[] dependsOn = mbd.getDependsOn();
         if (dependsOn != null) {
            //  看当前Bean 有没有依赖其他bean
            for (String dep : dependsOn) {
               if (isDependent(beanName, dep)) {
                  throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                        "Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");
               }
               registerDependentBean(dep, beanName);
               try {
                  //  如果依赖了，就先创建其他的Bean
                  getBean(dep);
               }
               catch (NoSuchBeanDefinitionException ex) {
                  throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                        "'" + beanName + "' depends on missing bean '" + dep + "'", ex);
               }
            }
         }

         // Create bean instance.
         if (mbd.isSingleton()) {
            sharedInstance = getSingleton(beanName, () -> {
               try {
                  // 具体创建Bean 的方法
                  return createBean(beanName, mbd, args);
               }
               catch (BeansException ex) {
                  // Explicitly remove instance from singleton cache: It might have been put there
                  // eagerly by the creation process, to allow for circular reference resolution.
                  // Also remove any beans that received a temporary reference to the bean.
                  destroySingleton(beanName);
                  throw ex;
               }
            });
            beanInstance = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
         }

         else if (mbd.isPrototype()) {
            // It's a prototype -> create a new instance.
            Object prototypeInstance = null;
            try {
               beforePrototypeCreation(beanName);
               prototypeInstance = createBean(beanName, mbd, args);
            }
            finally {
               afterPrototypeCreation(beanName);
            }
            beanInstance = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
         }

         else {
            String scopeName = mbd.getScope();
            if (!StringUtils.hasLength(scopeName)) {
               throw new IllegalStateException("No scope name defined for bean ´" + beanName + "'");
            }
            Scope scope = this.scopes.get(scopeName);
            if (scope == null) {
               throw new IllegalStateException("No Scope registered for scope name '" + scopeName + "'");
            }
            try {
               Object scopedInstance = scope.get(beanName, () -> {
                  beforePrototypeCreation(beanName);
                  try {
                     return createBean(beanName, mbd, args);
                  }
                  finally {
                     afterPrototypeCreation(beanName);
                  }
               });
               beanInstance = getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
            }
            catch (IllegalStateException ex) {
               throw new ScopeNotActiveException(beanName, scopeName, ex);
            }
         }
      }
      catch (BeansException ex) {
         beanCreation.tag("exception", ex.getClass().toString());
         beanCreation.tag("message", String.valueOf(ex.getMessage()));
         cleanupAfterBeanCreationFailure(beanName);
         throw ex;
      }
      finally {
         beanCreation.end();
      }
   }

   return adaptBeanInstance(name, beanInstance, requiredType);
}
```

这里面会调用到createBean() 来执行创建Bean

1. AbstractAutowireCapableBeanFactory.createBean
2. AbstractAutowireCapableBeanFactory.doCreateBean
3. AbstractAutowireCapableBeanFactory.createBeanInstance
4. AbstractAutowireCapableBeanFactory.instantiateBean

上面的执行链路来创建bean，instantiateBean 中会得到一个InstantiationStrategy 策略模式来得到一个创建bean的策略这里有两种实现

![image-20220424220927669](https://file.hyqup.cn/img/image-20220424220927669.png)

- jdk 动态反射方式创建：SimpleInstantiationStrategy

- cglib创建子类的方式：CglibSubclassingInstantiationStrategy

*** 重点  AbstractAutowireCapableBeanFactory 在doCreateBean 之后回去执行两个核心的方法

```java
populateBean(beanName, mbd, instanceWrapper);
exposedObject = initializeBean(beanName, exposedObject, mbd);
```

populateBean负责装配bean中其他的依赖对象

initializeBean 则负责**<u>初始化Bean实例</u>**

###  AbstractAutowireCapableBeanFactory.initializeBean

```
protected Object initializeBean(String beanName, Object bean, @Nullable RootBeanDefinition mbd) {
   if (System.getSecurityManager() != null) {
      AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
         invokeAwareMethods(beanName, bean);
         return null;
      }, getAccessControlContext());
   }
   else {
      invokeAwareMethods(beanName, bean);
   }

   Object wrappedBean = bean;
   if (mbd == null || !mbd.isSynthetic()) {
      wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
   }

   try {
      invokeInitMethods(beanName, wrappedBean, mbd);
   }
   catch (Throwable ex) {
      throw new BeanCreationException(
            (mbd != null ? mbd.getResourceDescription() : null),
            beanName, "Invocation of init method failed", ex);
   }
   if (mbd == null || !mbd.isSynthetic()) {
      wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
   }

   return wrappedBean;
}
```

初始化中有个关键的方法applyBeanPostProcessorsAfterInitialization

### AbstractAutowireCapableBeanFactory.applyBeanPostProcessorsBeforeInitialization

```java
@Override
public Object applyBeanPostProcessorsBeforeInitialization(Object existingBean, String beanName)
      throws BeansException {

   Object result = existingBean;
   for (BeanPostProcessor processor : getBeanPostProcessors()) {
      Object current = processor.postProcessBeforeInitialization(result, beanName);
      if (current == null) {
         return result;
      }
      result = current;
   }
   return result;
}
```

而applyBeanPostProcessorsAfterInitialization 会将所有的 BeanPostProcessor 拿取出来进行执行

### ApplicationContextAwareProcessor .postProcessBeforeInitialization 继而invokeAwareInterfaces

```java
@Override
@Nullable
public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
   if (!(bean instanceof EnvironmentAware || bean instanceof EmbeddedValueResolverAware ||
         bean instanceof ResourceLoaderAware || bean instanceof ApplicationEventPublisherAware ||
         bean instanceof MessageSourceAware || bean instanceof ApplicationContextAware ||
         bean instanceof ApplicationStartupAware)) {
      return bean;
   }

   AccessControlContext acc = null;

   if (System.getSecurityManager() != null) {
      acc = this.applicationContext.getBeanFactory().getAccessControlContext();
   }

   if (acc != null) {
      AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
         invokeAwareInterfaces(bean);
         return null;
      }, acc);
   }
   else {
      invokeAwareInterfaces(bean);
   }

   return bean;
}

private void invokeAwareInterfaces(Object bean) {
   if (bean instanceof EnvironmentAware) {
      ((EnvironmentAware) bean).setEnvironment(this.applicationContext.getEnvironment());
   }
   if (bean instanceof EmbeddedValueResolverAware) {
      ((EmbeddedValueResolverAware) bean).setEmbeddedValueResolver(this.embeddedValueResolver);
   }
   if (bean instanceof ResourceLoaderAware) {
      ((ResourceLoaderAware) bean).setResourceLoader(this.applicationContext);
   }
   if (bean instanceof ApplicationEventPublisherAware) {
      ((ApplicationEventPublisherAware) bean).setApplicationEventPublisher(this.applicationContext);
   }
   if (bean instanceof MessageSourceAware) {
      ((MessageSourceAware) bean).setMessageSource(this.applicationContext);
   }
   if (bean instanceof ApplicationStartupAware) {
      ((ApplicationStartupAware) bean).setApplicationStartup(this.applicationContext.getApplicationStartup());
   }
   if (bean instanceof ApplicationContextAware) {
      ((ApplicationContextAware) bean).setApplicationContext(this.applicationContext);
   }
}
```

至此，这里执行的arare 这个BeanPostProcessor的接口回调。

整体流程下来，也就是为什么我们Bean实现 了xxxAware,就能给我们的组件注入相关对象。

后续章节会细细讲解

![image-20220424223748792](https://file.hyqup.cn/img/image-20220424223748792.png)

## 整体的流程图

![简单了解Bean创建过程.drawio](https://file.hyqup.cn/img/%E7%AE%80%E5%8D%95%E4%BA%86%E8%A7%A3Bean%E5%88%9B%E5%BB%BA%E8%BF%87%E7%A8%8B.drawio.png)

## 后续

这里我们了解了Aware 的实现方式，同时也引出了一个新的概念 BeanPostProcessor

Bean的功能增强全部都是BeanPostProcessor+InitializingBean  这两个特性来实现的，包括AOP、事务、通知、WEB等功能基本上全是这样来搞出来的，后续章节会一一揭开它们的面纱

