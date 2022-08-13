---
title: 剑指源码-springmvc-(三)-HandlerMapping和HandlerAdapter细节
index_img: https://tva4.sinaimg.cn/large/9bd9b167ly1g2rkq7yxx2j21hc0u0ai6.jpg
banner_img: https://tva4.sinaimg.cn/large/9bd9b167gy1g2rl48pag4j21hc0u0dtj.jpg
abbrlink: 5a47f1a6
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

> 这个时候会准备参数解析器和返回值处理器



```java
	@Override
	public void afterPropertiesSet() {
		// Do this first, it may add ResponseBody advice beans
		initControllerAdviceCache();

		if (this.argumentResolvers == null) {
			List<HandlerMethodArgumentResolver> resolvers = getDefaultArgumentResolvers();
			this.argumentResolvers = new HandlerMethodArgumentResolverComposite().addResolvers(resolvers);
		}
		if (this.initBinderArgumentResolvers == null) {
			List<HandlerMethodArgumentResolver> resolvers = getDefaultInitBinderArgumentResolvers();
			this.initBinderArgumentResolvers = new HandlerMethodArgumentResolverComposite().addResolvers(resolvers);
		}
		if (this.returnValueHandlers == null) {
			List<HandlerMethodReturnValueHandler> handlers = getDefaultReturnValueHandlers();
			this.returnValueHandlers = new HandlerMethodReturnValueHandlerComposite().addHandlers(handlers);
		}
	}
```

#### 参数解析器和返回值处理器的前置工作

 RequestMappingHandlerAdapter.invokeHandlerMethod

```java
@Nullable
	protected ModelAndView invokeHandlerMethod(HttpServletRequest request,
			HttpServletResponse response, HandlerMethod handlerMethod) throws Exception {

		ServletWebRequest webRequest = new ServletWebRequest(request, response);
		try {
			WebDataBinderFactory binderFactory = getDataBinderFactory(handlerMethod);
			ModelFactory modelFactory = getModelFactory(handlerMethod, binderFactory);
			/**
			 * 用于组合执行期间所需要的一些组件
			 */
			ServletInvocableHandlerMethod invocableMethod = createInvocableHandlerMethod(handlerMethod);
			if (this.argumentResolvers != null) {
				invocableMethod.setHandlerMethodArgumentResolvers(this.argumentResolvers);
			}
			if (this.returnValueHandlers != null) {
				invocableMethod.setHandlerMethodReturnValueHandlers(this.returnValueHandlers);
			}
			invocableMethod.setDataBinderFactory(binderFactory);
			invocableMethod.setParameterNameDiscoverer(this.parameterNameDiscoverer);

			ModelAndViewContainer mavContainer = new ModelAndViewContainer();
			mavContainer.addAllAttributes(RequestContextUtils.getInputFlashMap(request));
			modelFactory.initModel(webRequest, mavContainer, invocableMethod);
			mavContainer.setIgnoreDefaultModelOnRedirect(this.ignoreDefaultModelOnRedirect);

			AsyncWebRequest asyncWebRequest = WebAsyncUtils.createAsyncWebRequest(request, response);
			asyncWebRequest.setTimeout(this.asyncRequestTimeout);

			WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);
			asyncManager.setTaskExecutor(this.taskExecutor);
			asyncManager.setAsyncWebRequest(asyncWebRequest);
			asyncManager.registerCallableInterceptors(this.callableInterceptors);
			asyncManager.registerDeferredResultInterceptors(this.deferredResultInterceptors);

			if (asyncManager.hasConcurrentResult()) {
				Object result = asyncManager.getConcurrentResult();
				mavContainer = (ModelAndViewContainer) asyncManager.getConcurrentResultContext()[0];
				asyncManager.clearConcurrentResult();
				LogFormatUtils.traceDebug(logger, traceOn -> {
					String formatted = LogFormatUtils.formatValue(result, !traceOn);
					return "Resume with async result [" + formatted + "]";
				});
				invocableMethod = invocableMethod.wrapConcurrentResult(result);
			}
			/**
			 * 具体执行期间，会传入临时容器mavContainer
			 */
			invocableMethod.invokeAndHandle(webRequest, mavContainer);
			if (asyncManager.isConcurrentHandlingStarted()) {
				return null;
			}
			/**
			 * 从mavContainer抽取ModelAndView
			 */
			return getModelAndView(mavContainer, modelFactory, webRequest);
		}
		finally {
			webRequest.requestCompleted();
		}
	}
```

未来反射解析目标方法中的每一个值 argumentResolvers

```
if (this.argumentResolvers != null) {
   invocableMethod.setHandlerMethodArgumentResolvers(this.argumentResolvers);
}
```

返回值处理器，未来用于处理目标方法执行后的返回值，无论目标方法返回什么会转换为适配器所使用的ModelAndView

```
 if (this.returnValueHandlers != null) {
     invocableMethod.setHandlerMethodReturnValueHandlers(this.returnValueHandlers);
  }
```

```
ServletInvocableHandlerMethod invocableMethod = createInvocableHandlerMethod(handlerMethod);
```

把该有的组件组合到ServletInvocableHandlerMethod对象中去，最终使用ServletInvocableHandlerMethod进行invokeAndHandle，将数据最后封装到ModelAndViewContainer中去，最后再将ModelAndViewContainer（临时容器，每一次请求都是新new 的对象，同一次请求期间共享数据）抽取ModelAndView（数据和视图）。

#### 参数解析器

> 27个参数解析器

 ServletInvocableHandlerMethod.invokeAndHandle-->invokeForRequest-->getMethodArgumentValues

```
protected Object[] getMethodArgumentValues(NativeWebRequest request, @Nullable ModelAndViewContainer mavContainer,
			Object... providedArgs) throws Exception {

		MethodParameter[] parameters = getMethodParameters();
		if (ObjectUtils.isEmpty(parameters)) {
			return EMPTY_ARGS;
		}

		Object[] args = new Object[parameters.length];
		/**
		 * 遍历所有的参数使用参数解析器去解析，首个参数解析器解析到了就跳过，执行下一个参数解析
		 */
		for (int i = 0; i < parameters.length; i++) {
			MethodParameter parameter = parameters[i];
			parameter.initParameterNameDiscovery(this.parameterNameDiscoverer);
			args[i] = findProvidedArgument(parameter, providedArgs);
			if (args[i] != null) {
				continue;
			}
			/**
			 * 这里判断采用门面以27种都去判断一遍
			 */
			if (!this.resolvers.supportsParameter(parameter)) {
				throw new IllegalStateException(formatArgumentError(parameter, "No suitable resolver"));
			}
			try {
				/**
				 * 支持了就开始解析
				 */
				args[i] = this.resolvers.resolveArgument(parameter, mavContainer, request, this.dataBinderFactory);
			}
			catch (Exception ex) {
				// Leave stack trace for later, exception may actually be resolved and handled...
				if (logger.isDebugEnabled()) {
					String exMsg = ex.getMessage();
					if (exMsg != null && !exMsg.contains(parameter.getExecutable().toGenericString())) {
						logger.debug(formatArgumentError(parameter, exMsg));
					}
				}
				throw ex;
			}
		}
		return args;
	}
```

 

#### 返回值处理器

> 15个返回值处理器

 ServletInvocableHandlerMethod.invokeAndHandle

```java
public void invokeAndHandle(ServletWebRequest webRequest, ModelAndViewContainer mavContainer,
      Object... providedArgs) throws Exception {
   /**
    * 目标方法的放射执行
    */
   Object returnValue = invokeForRequest(webRequest, mavContainer, providedArgs);
   setResponseStatus(webRequest);

   if (returnValue == null) {
      if (isRequestNotModified(webRequest) || getResponseStatus() != null || mavContainer.isRequestHandled()) {
         disableContentCachingIfNecessary(webRequest);
         mavContainer.setRequestHandled(true);
         return;
      }
   }
   else if (StringUtils.hasText(getResponseStatusReason())) {
      mavContainer.setRequestHandled(true);
      return;
   }

   mavContainer.setRequestHandled(false);
   Assert.state(this.returnValueHandlers != null, "No return value handlers");
   try {
      this.returnValueHandlers.handleReturnValue(
            returnValue, getReturnValueType(returnValue), mavContainer, webRequest);
   }
   catch (Exception ex) {
      if (logger.isTraceEnabled()) {
         logger.trace(formatErrorForReturnValue(returnValue), ex);
      }
      throw ex;
   }
}
```

找到 returnValueHandlers 执行handleReturnValue

```java
public void handleReturnValue(@Nullable Object returnValue, MethodParameter returnType,
      ModelAndViewContainer mavContainer, NativeWebRequest webRequest) throws Exception {
   /**
    * 找到合适的返回值处理器
    */
   HandlerMethodReturnValueHandler handler = selectHandler(returnValue, returnType);
   if (handler == null) {
      throw new IllegalArgumentException("Unknown return value type: " + returnType.getParameterType().getName());
   }
   /**
    * 执行返回值处理器的处理方法
    */
   handler.handleReturnValue(returnValue, returnType, mavContainer, webRequest);
}
```

逻辑依然是以先找到为准

接下来执行后置的视图解析器相关业务流程

#### 视图解析器

##### 核心接口

```java
public interface View {


   String RESPONSE_STATUS_ATTRIBUTE = View.class.getName() + ".responseStatus";


   String PATH_VARIABLES = View.class.getName() + ".pathVariables";


   String SELECTED_CONTENT_TYPE = View.class.getName() + ".selectedContentType";



   @Nullable
   default String getContentType() {
      return null;
   }

   void render(@Nullable Map<String, ?> model, HttpServletRequest request, HttpServletResponse response)
         throws Exception;

}
```

```
public interface ViewResolver {


   @Nullable
   View resolveViewName(String viewName, Locale locale) throws Exception;

}
```

View 视图 

ViewResolver  视图解析

##### 流程分析

  HandlerAdapter执行完成后会返回ModelAndView对象（里面包含viewName）

所有ViewResolver去解析viewName，一得到就返回，得到View对象，最后调用VIew对象render渲染
