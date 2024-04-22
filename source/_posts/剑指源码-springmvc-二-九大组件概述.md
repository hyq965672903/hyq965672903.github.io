---
title: 剑指源码-springmvc-(二)-九大组件概述
index_img: https://tva2.sinaimg.cn/large/9bd9b167gy1g2rmxmh3vqj21hc0u0kjl.jpg
banner_img: https://tva4.sinaimg.cn/large/9bd9b167gy1g2rn8ha2csj21hc0u04qr.jpg
abbrlink: fd09ed1d
date: 2022-05-16 23:31:37
tags:
categories:
 - Java
 - SpringMVC源码
description:
---

概述springMVC九大组件的相关概念，及其基本作用，了解其初始化实现机制

<!-- more -->

## 了解初始化的时机

![image-20220516212304671](http://hyqup-blog-upyun.test.upcdn.net/img/image-20220516212304671.png)

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

## SpringMVC的请求链路

> SpringMVC的请求最基础原理是一个DispatcherServlet来处理请求，DispatcherServlet的最顶层也是一个Servlet，所以也是遵循Servlet规范来执行

从`DispatcherServlet` 的Diagram可以看出，上三层分别是Servlet、GenericServlet、HttpServlet。这三个属于Servlet规范然后是HttpServletBean FrameworkServlet以及DispatcherServlet 后三个是属于Spring家族的

当前端发送一个请求时候，执行到`Servlet.service()`--->`GenericServlet.service()`--->`HttpServlet.service()`执行到相应的doGet/doPost--->FrameworkServlet会重写doGet/doPost，全部去执行`processRequest()`，留给子类的`doService()`--->DispatcherServlet实现doService，最终调用到`doDispatch()`

所以，分析SpringMVC请求链路，从重分析`doDispatch`逻辑

```java
	@SuppressWarnings("deprecation")
	protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
		HttpServletRequest processedRequest = request;
		HandlerExecutionChain mappedHandler = null;
		// 文件上传的标识
		boolean multipartRequestParsed = false;
		// 异步请求支持
		WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);

		try {
			ModelAndView mv = null;
			Exception dispatchException = null;

			try {
				//检查是否文件上传,判断
				processedRequest = checkMultipart(request);
				multipartRequestParsed = (processedRequest != request);

				// Determine handler for the current request.
				// 使用HandlerMapping决定使用哪个Handler处理当前请求，会构造出 目标方法+拦截器链
				mappedHandler = getHandler(processedRequest);
				if (mappedHandler == null) {
					noHandlerFound(processedRequest, response);
					return;
				}

				// Determine handler adapter for the current request.
				// 使用HandlerAdapter 知道适配器
				HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());

				// Process last-modified header, if supported by the handler.
				String method = request.getMethod();
				boolean isGet = HttpMethod.GET.matches(method);
				if (isGet || HttpMethod.HEAD.matches(method)) {
					long lastModified = ha.getLastModified(request, mappedHandler.getHandler());
					if (new ServletWebRequest(request, response).checkNotModified(lastModified) && isGet) {
						return;
					}
				}
				
				// 处理拦截器链的 preHandle （前置）方法
				if (!mappedHandler.applyPreHandle(processedRequest, response)) {
					return;
				}

				// Actually invoke the handler.
				// 真正反射执行
				mv = ha.handle(processedRequest, response, mappedHandler.getHandler());

				if (asyncManager.isConcurrentHandlingStarted()) {
					return;
				}

				applyDefaultViewName(processedRequest, mv);
				// 处理拦截器链的 preHandle （后置）方法
				mappedHandler.applyPostHandle(processedRequest, response, mv);
			}
			catch (Exception ex) {
				dispatchException = ex;
			}
			catch (Throwable err) {
				// As of 4.3, we're processing Errors thrown from handler methods as well,
				// making them available for @ExceptionHandler methods and other scenarios.
				dispatchException = new NestedServletException("Handler dispatch failed", err);
			}
			processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);
		}
		catch (Exception ex) {
			triggerAfterCompletion(processedRequest, response, mappedHandler, ex);
		}
		catch (Throwable err) {
			triggerAfterCompletion(processedRequest, response, mappedHandler,
					new NestedServletException("Handler processing failed", err));
		}
		finally {
			if (asyncManager.isConcurrentHandlingStarted()) {
				// Instead of postHandle and afterCompletion
				if (mappedHandler != null) {
					mappedHandler.applyAfterConcurrentHandlingStarted(processedRequest, response);
				}
			}
			else {
				// Clean up any resources used by a multipart request.
				if (multipartRequestParsed) {
					cleanupMultipart(processedRequest);
				}
			}
		}
	}
```

核心流程如上，这里面会涉及到HandlerMaping 的几种实现,以及HandlerAdapter的几种实现，下一章节详细分析

这里拦截器概念引入。和以前的拦截器过滤器

### 过滤器、拦截器、监听器对比

- **过滤器**

  Filter过滤器是Servlet容器层面的，过滤器是对数据进行过滤，预处理过程

- **拦截器**

  Interceptor拦截器和Filter和Listener有本质上的不同，前面二者都是依赖于Servlet容器，而Interceptor则是依赖于Spring框架，是aop的一种表现，基于Java的动态代理实现的。

  实现步骤：

  1. 声明拦截器的类：通过实现 HandlerInterceptor接口，实现preHandle、postHandle和afterCompletion方法。
  2. 通过配置类配置拦截器：通过实现WebMvcConfigurer接口，实现addInterceptors方法。

- **监听器**

  Listener监听器也是Servlet层面，可以用于监听Web应用中某些对象、信息的创建、销毁和修改等动作发生，然后做出相应的响应处理

  监听器分为3类：

  1. ServletContext：对应application，实现接口ServletContextListener。在整个Web服务中只有一个，在Web服务关闭时销毁。可用于做数据缓存，例如结合redis，在Web服务创建时从数据库拉取数据到缓存服务器。
  2. HttpSession：对应session会话，实现接口HttpSessionListener。在会话起始时创建，一端关闭会话后销毁。可用作获取在线用户数量。
  3. ServletRequest：对应request，实现接口ServletRequestListener。request对象是客户发送请求时创建的，用于封装请求数据，请求处理完毕后销毁。可用作封装用户信息。
