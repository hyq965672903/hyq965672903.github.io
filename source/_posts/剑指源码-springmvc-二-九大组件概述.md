---
title: 剑指源码-springmvc-(二)-九大组件概述
index_img: /img/default.png
banner_img: /img/default.png
date: 2022-05-16 19:52:16
tags:
categories:
description:
---

 概述springMVC九大组件的相关概念，及其基本作用，了解其初始化实现机制

<!-- more -->

## 了解初始化的时机

![image-20220516212304671](https://file.hyqup.cn/img/image-20220516212304671.png)

初始化web容器的时候initWebApplicationContext，有设置监听器wac.addApplicationListener(new SourceFilteringListener(wac, new ContextRefreshListener()));

而FrameworkServlet中执行了onRefresh 方法，由子类实现

```
public void onApplicationEvent(ContextRefreshedEvent event) {
   this.refreshEventReceived = true;
   synchronized (this.onRefreshMonitor) {
      onRefresh(event.getApplicationContext());
   }
}
```

这里DispatcherServlet里面实现了onRefresh方法这里面会对九大组件进行初始化

```
@Override
protected void onRefresh(ApplicationContext context) {
   initStrategies(context);
}

/**
 * Initialize the strategy objects that this servlet uses.
 * <p>May be overridden in subclasses in order to initialize further strategy objects.
 */
protected void initStrategies(ApplicationContext context) {
   initMultipartResolver(context);
   initLocaleResolver(context);
   initThemeResolver(context);
   initHandlerMappings(context);
   initHandlerAdapters(context);
   initHandlerExceptionResolvers(context);
   initRequestToViewNameTranslator(context);
   initViewResolvers(context);
   initFlashMapManager(context);
}
```

## 九大组件介绍

> https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#mvc-servlet-special-bean-types官方这里列举了八大组件，RequestToViewNameTranslator介绍相对较少
>
> 这里的组件除了 MultipartResolver，都有默认实现，也就是说如果我们要实现文件上传，就需要自己实现一个MultipartResolver

前置概念：`Handler`、`HandlerMapping`和`HandlerAdapter`

Handler：处理器，实际上处理请求的就是Handler

HandlerMapping：处理器映射器，查找Handler的，SpringMVC会接收很多request请求，如何确定哪一个请求是哪一个Handler，就是靠HandlerMapping

HandlerAdapter：处理器适配器，灵活的调用Handler处理具体逻辑



- **HandlerMapping（处理器映射器）**

  根据`request`查找请求对应的 `Handler` 和 `Interceptor`

- **HandlerAdapter（处理器适配器）**

  用来适配找到`Handler`对应的适配器，并进行执行

- **HandlerExceptionResolver（异常解析器 ）**

  处理 `Handler` 产⽣的异常，根据异常设置`ModelAndView`，之后交给渲染⽅法进⾏渲染

- **ViewResolver（视图解析器）**

  将 `String` 类型的视图名和 `Locale` 解析为 `View` 类型的视图

- **RequestToViewNameTranslator（请求视图名转换器）**

  从请求中获取 `ViewName`

- **LocaleResolver（区域化解析器）**

  从请求中解析出 `Locale`

- **ThemeResolver（主题解析器）**

   从请求中解析出主题名，并获取主题具体的资源

- **MultipartResolver（分片解析器）**

  封装普通的请求，使其拥有⽂件上传的功能

- **FlashMapManager（闪存管理器）**

   ⽤于重定向时的参数传递

