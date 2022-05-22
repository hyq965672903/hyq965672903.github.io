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

```java
public class BeanNameUrlHandlerMapping extends AbstractDetectingUrlHandlerMapping {

   /**
    * Checks name and aliases of the given bean for URLs, starting with "/".
    */
   @Override
   protected String[] determineUrlsForHandler(String beanName) {
      List<String> urls = new ArrayList<>();
      if (beanName.startsWith("/")) {
         urls.add(beanName);
      }
      String[] aliases = obtainApplicationContext().getAliases(beanName);
      for (String alias : aliases) {
         if (alias.startsWith("/")) {
            urls.add(alias);
         }
      }
      return StringUtils.toStringArray(urls);
   }

}
```

该实现类可以把IOC容器中name以"/" 开头的Bean注册为handler,

### SimpleUrlHandlerMapping

手动配置url与handler的映射,初始化的时候就注册进去了

### RequestMappingHandlerMapping

这是最常用的HandlerMapping实现，通过它，我们可以使用注解的形式来标识url与handler的映射：

 AbstractHandlerMethodMapping.initHandlerMethods然后执行到 processCandidateBean->detectHandlerMethods再到交由子类实现getMappingForMethod

RequestMappingHandlerMapping中getMappingForMethod

```java
private RequestMappingInfo createRequestMappingInfo(AnnotatedElement element) {
   RequestMapping requestMapping = AnnotatedElementUtils.findMergedAnnotation(element, RequestMapping.class);
   RequestCondition<?> condition = (element instanceof Class ?
         getCustomTypeCondition((Class<?>) element) : getCustomMethodCondition((Method) element));
   return (requestMapping != null ? createRequestMappingInfo(requestMapping, condition) : null);
}
```

会去判断实现RequestMapping的方法实现，而加入RequestMappingInfo信息中

### RouterFunctionMapping

支持webflux相关

## HandlerAdapter的实现类

- **HttpRequestHandlerAdapter** 判断是否是实现HttpRequestHandler接口
- **SimpleControllerHandlerAdapter**判断是否实现Controller接口
- **RequestMappingHandlerAdapter** 判断是不是HandlerMethod，注解类型的都是这种


```java
public interface HandlerAdapter {

	/**
	 * 是否支持该处理器
	 */
   boolean supports(Object handler);

	/**
	 * 执行处理器，返回 ModelAndView 结果
	 */
   @Nullable
   ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception;

	/**
	 * 返回请求的最新更新时间，如果不支持该操作，则返回 -1 即可
	 */
   @Deprecated
   long getLastModified(HttpServletRequest request, Object handler);

}
```

### HttpRequestHandlerAdapter

如果这个处理器实现了 `HttpRequestHandler` 接口，则使用 `HttpRequestHandlerAdapter` 调用该处理器的 `handleRequest(HttpServletRequest request, HttpServletResponse response)`方法去处理器请求

### SimpleControllerHandlerAdapter

和 HttpRequestHandlerAdapter 差不多，如果这个处理器实现了 `Controoler` 接口，则使用 `HttpRequestHandlerAdapter` 调用该处理器的 `handleRequest(HttpServletRequest request, HttpServletResponse response)`方法去处理器请求，直接返回处理器执行后返回 `ModelAndView`

### RequestMappingHandlerAdapter

实现 BeanFactoryAware、InitializingBean 接口，继承 AbstractHandlerMethodAdapter 抽象类，基于 `@RequestMapping` 注解的 HandlerMethod 处理器的 HandlerMethodAdapter 实现类

几个主要的属性对象：

- `HandlerMethodArgumentResolverComposite argumentResolvers`：参数处理器组合对象
- `HandlerMethodReturnValueHandlerComposite returnValueHandlers`：返回值处理器组合对象
- `List<HttpMessageConverter<?>> messageConverters`：HTTP 消息转换器集合对象
- `List<Object> requestResponseBodyAdvice`： RequestResponseAdvice 集合对象

#### InitializingBean对参数初始化

```java
@Override
public void afterPropertiesSet() throws Exception {
   Assert.notNull(this.applicationContext, "ApplicationContext is required");

   if (CollectionUtils.isEmpty(this.messageReaders)) {
      ServerCodecConfigurer codecConfigurer = ServerCodecConfigurer.create();
      this.messageReaders = codecConfigurer.getReaders();
   }
   if (this.argumentResolverConfigurer == null) {
      this.argumentResolverConfigurer = new ArgumentResolverConfigurer();
   }
   if (this.reactiveAdapterRegistry == null) {
      this.reactiveAdapterRegistry = ReactiveAdapterRegistry.getSharedInstance();
   }

   this.methodResolver = new ControllerMethodResolver(this.argumentResolverConfigurer,
         this.reactiveAdapterRegistry, this.applicationContext, this.messageReaders);

   this.modelInitializer = new ModelInitializer(this.methodResolver, this.reactiveAdapterRegistry);
}
```

#### 参数解析器

#### 返回值处理器







