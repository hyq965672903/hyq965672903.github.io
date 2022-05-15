---
title: 剑指向源码-springmvc(一)-序章
index_img: https://file.hyqup.cn/img/wallhaven-nkorgd.jpg
banner_img: https://file.hyqup.cn/img/wallhaven-mdvo58.jpg
date: 2022-05-14 09:42:38
tags:
categories:
-Java
-SpringMVC源码
description:
---

搭建springmvc源码环境。准备工作与前置工作分析

<!-- more -->

> 前置基础：了解SPI,Service Provider Interface,服务提供者接口 jdk提供给“服务提供厂商”或者“插件开发者”使用的接口,不需要修改原来作为接口的jar的情况下，将原来实现的那个jar替换为另外一种实现的jar即可。

使用规范：

定义服务的通用接口，针对通用的服务接口，提供具体的实现类。

在jar包的META-INF/services/目录中，新建一个文件，文件名为 接口的"全限定名"。 文件内容为该接口的具体实现类的"全限定名"。

将spi所在jar放在主程序的classpath中

服务调用方用java.util.ServiceLoader，用服务接口为参数，去动态加载具体的实现类到JVM中。

## 模块搭建

### 1、新增springmvcsource-test

![image-20220514094502540](https://file.hyqup.cn/img/image-20220514094502540.png)

### 2、引入相关依赖

springmvcsource-test 的build.gradle  dependencies下加入依赖

#### 2.1 源码webmvc

```groovy
implementation (project(":spring-webmvc"))
```

#### 2.2  servlet依赖

```groovy
implementation  group: 'javax.servlet', name: 'javax.servlet-api', version: '4.0.1'
```

ps:这里是**implementation**，这样lib才会打入war,从而Tomcat启动springmvc相关才会生效（非常重要），这里卡了我大半天。

不然访问时候总是404 找不到路径

### 2.3 gradle  补充知识点

目前Gradle版本支持的依赖配置有：implementation、api、compileOnly、runtimeOnly和annotationProcessor，已经废弃的配置有：compile、provided、apk、providedCompile。此外依赖配置还可以加一些配置项，例如AndroidTestImplementation、debugApi等等。

常用的是implementation、api、compileOnly三个依赖配置，含义如下：

- implementation
   与compile对应，会添加依赖到编译路径，并且会将依赖打包到输出（aar或apk），但是在编译时不会将依赖的实现暴露给其他module，也就是只有在运行时其他module才能访问这个依赖中的实现。使用这个配置，可以显著提升构建时间，因为它可以减少重新编译的module的数量。建议，尽量使用这个依赖配置。
- api
   与compile对应，功能完全一样，会添加依赖到编译路径，并且会将依赖打包到输出（aar或apk），与implementation不同，这个依赖可以传递，其他module无论在编译时和运行时都可以访问这个依赖的实现，也就是会泄漏一些不应该不使用的实现。举个例子，A依赖B，B依赖C，如果都是使用api配置的话，A可以直接使用C中的类（编译时和运行时），而如果是使用implementation配置的话，在编译时，A是无法访问C中的类的。
- compileOnly
   与provided对应，Gradle把依赖加到编译路径，编译时使用，不会打包到输出（aar或apk）。这可以减少输出的体积，在只在编译时需要，在运行时可选的情况，很有用。
- runtimeOnly
   与apk对应，gradle添加依赖只打包到APK，运行时使用，但不会添加到编译路径。这个没有使用过。
- annotationProcessor
   与compile对应，用于注解处理器的依赖配置。

### 3、编码

```java
package cn.hyqup.web;

import cn.hyqup.web.config.AppConfig;
import org.springframework.web.WebApplicationInitializer;
import org.springframework.web.context.support.AnnotationConfigWebApplicationContext;
import org.springframework.web.servlet.DispatcherServlet;

import javax.servlet.ServletContext;
import javax.servlet.ServletException;
import javax.servlet.ServletRegistration;

/**
 * Copyright © 2022 灼华. All rights reserved.
 *
 * @author create by hyq
 * @version 0.1
 * @date 2022/5/14
 * @description:
 */
public class AppStarter implements WebApplicationInitializer {
	@Override
	public void onStartup(ServletContext servletContext) throws ServletException {
		// 创建一个webIOC容器，并注册主配置类吗  注解版的配置类注册进去
		AnnotationConfigWebApplicationContext context = new AnnotationConfigWebApplicationContext();
		context.register(AppConfig.class);

		// 创建并注册 DispatcherServlet
		DispatcherServlet servlet = new DispatcherServlet(context);
		ServletRegistration.Dynamic registration = servletContext.addServlet("app", servlet);
		registration.setLoadOnStartup(1);
		registration.addMapping("/");
	}
}
```

WebApplicationInitializer需要**servlet3.0以上**，**Tomcat6.0以上**（6.0以上支持servlet3.0）

上面的配置相当于web.xml配置了，上述代码采用api形式植入

```xml
<web-app>

    <listener>
        <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
    </listener>

    <context-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>/WEB-INF/app-context.xml</param-value>
    </context-param>

    <servlet>
        <servlet-name>app</servlet-name>
        <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
        <init-param>
            <param-name>contextConfigLocation</param-name>
            <param-value></param-value>
        </init-param>
        <load-on-startup>1</load-on-startup>
    </servlet>

    <servlet-mapping>
        <servlet-name>app</servlet-name>
        <url-pattern>/app/*</url-pattern>
    </servlet-mapping>

</web-app>
```



```java
package cn.hyqup.web.config;

import org.springframework.context.annotation.ComponentScan;
import org.springframework.context.annotation.Configuration;

/**
 * Copyright © 2022 灼华. All rights reserved.
 *
 * @author create by hyq
 * @version 0.1
 * @date 2022/5/14
 * @description:
 */
@ComponentScan("cn.hyqup.web")
@Configuration
public class AppConfig {
}
```

说明：配置了AppStarter，相当于Tomcat一启动就加载

1. 创建了容器，spring基本功能完成
2. 注册了一个根servlet:DispatcherServlet，后面所有的请求都由 DispatcherServlet 处理

```java
package cn.hyqup.web.controller;

import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

/**
 * Copyright © 2022 灼华. All rights reserved.
 *
 * @author create by hyq
 * @version 0.1
 * @date 2022/5/14
 * @description:
 */
@RestController
public class HelloController {
   @GetMapping("/say")
   public String say() {
      return "Hello SpringMVC";
   }
}
```

### 4、部署访问

http://localhost:8080/webmvc/say

## 原理剖析

> SpringMVC基于SPI启动了web容器，servlet定义ServletContainerInitializer。
>

### 主流程分析启动web容器

servlet定义ServletContainerInitializer，

![image-20220514235410561](https://file.hyqup.cn/img/image-20220514235410561.png)

```java
@HandlesTypes(WebApplicationInitializer.class)
public class SpringServletContainerInitializer implements ServletContainerInitializer {

   @Override
   public void onStartup(@Nullable Set<Class<?>> webAppInitializerClasses, ServletContext servletContext)
         throws ServletException {

      List<WebApplicationInitializer> initializers = Collections.emptyList();

      if (webAppInitializerClasses != null) {
         initializers = new ArrayList<>(webAppInitializerClasses.size());
         for (Class<?> waiClass : webAppInitializerClasses) {
            // Be defensive: Some servlet containers provide us with invalid classes,
            // no matter what @HandlesTypes says...
            if (!waiClass.isInterface() && !Modifier.isAbstract(waiClass.getModifiers()) &&
                  WebApplicationInitializer.class.isAssignableFrom(waiClass)) {
               try {
                  initializers.add((WebApplicationInitializer)
                        ReflectionUtils.accessibleConstructor(waiClass).newInstance());
               }
               catch (Throwable ex) {
                  throw new ServletException("Failed to instantiate WebApplicationInitializer class", ex);
               }
            }
         }
      }

      if (initializers.isEmpty()) {
         servletContext.log("No Spring WebApplicationInitializer types detected on classpath");
         return;
      }

      servletContext.log(initializers.size() + " Spring WebApplicationInitializers detected on classpath");
      AnnotationAwareOrderComparator.sort(initializers);
      for (WebApplicationInitializer initializer : initializers) {
         initializer.onStartup(servletContext);
      }
   }

}
```

HandlesTypes 感兴趣的类WebApplicationInitializer，会去找到所有实现了WebApplicationInitializer的类，我们这里AppStarter就是实现了WebApplicationInitializer，启动时候执行WebApplicationInitializer.onStartup

接下来逻辑：

```java
	// 创建一个webIOC容器，并注册主配置类吗  注解版的配置类注册进去
	AnnotationConfigWebApplicationContext context = new AnnotationConfigWebApplicationContext();
	context.register(AppConfig.class);

	// 创建并注册 DispatcherServlet
	DispatcherServlet servlet = new DispatcherServlet(context);
	ServletRegistration.Dynamic registration = servletContext.addServlet("app", servlet);
	registration.setLoadOnStartup(1);
	registration.addMapping("/");
```

准备一个空的webioc容器，准备一个DispatcherServlet，并将ioc容器传过去，并注册到ServletContext（Tomcat）中去。DispatcherServlet本质上也是一个servlet,所以servlet在执行init的时候就会将该ioc容器执行spring相关的refresh()逻辑将容器刷新。具体初始化逻辑会在HttpServletBean init重写，init有个抽象方法initServletBean，FrameworkServlet来实现initServletBean

```java
@Override
protected final void initServletBean() throws ServletException {
   getServletContext().log("Initializing Spring " + getClass().getSimpleName() + " '" + getServletName() + "'");
   if (logger.isInfoEnabled()) {
      logger.info("Initializing Servlet '" + getServletName() + "'");
   }
   long startTime = System.currentTimeMillis();

   try {
       //初始化web容器
      this.webApplicationContext = initWebApplicationContext();
      initFrameworkServlet();
   }
   catch (ServletException | RuntimeException ex) {
      logger.error("Context initialization failed", ex);
      throw ex;
   }

   if (logger.isDebugEnabled()) {
      String value = this.enableLoggingRequestDetails ?
            "shown which may lead to unsafe logging of potentially sensitive data" :
            "masked to prevent unsafe logging of potentially sensitive data";
      logger.debug("enableLoggingRequestDetails='" + this.enableLoggingRequestDetails +
            "': request parameters and headers will be " + value);
   }

   if (logger.isInfoEnabled()) {
      logger.info("Completed initialization in " + (System.currentTimeMillis() - startTime) + " ms");
   }
}
```

初始化代码

```java
protected WebApplicationContext initWebApplicationContext() {
   WebApplicationContext rootContext =
         WebApplicationContextUtils.getWebApplicationContext(getServletContext());
   WebApplicationContext wac = null;

   if (this.webApplicationContext != null) {
      // A context instance was injected at construction time -> use it
      wac = this.webApplicationContext;
      if (wac instanceof ConfigurableWebApplicationContext) {
         ConfigurableWebApplicationContext cwac = (ConfigurableWebApplicationContext) wac;
         if (!cwac.isActive()) {
            // The context has not yet been refreshed -> provide services such as
            // setting the parent context, setting the application context id, etc
            if (cwac.getParent() == null) {
               // The context instance was injected without an explicit parent -> set
               // the root application context (if any; may be null) as the parent
               cwac.setParent(rootContext);
            }
            configureAndRefreshWebApplicationContext(cwac);
         }
      }
   }
   if (wac == null) {
      // No context instance was injected at construction time -> see if one
      // has been registered in the servlet context. If one exists, it is assumed
      // that the parent context (if any) has already been set and that the
      // user has performed any initialization such as setting the context id
      wac = findWebApplicationContext();
   }
   if (wac == null) {
      // No context instance is defined for this servlet -> create a local one
      wac = createWebApplicationContext(rootContext);
   }

   if (!this.refreshEventReceived) {
      // Either the context is not a ConfigurableApplicationContext with refresh
      // support or the context injected at construction time had already been
      // refreshed -> trigger initial onRefresh manually here.
      synchronized (this.onRefreshMonitor) {
         onRefresh(wac);
      }
   }

   if (this.publishContext) {
      // Publish the context as a servlet context attribute.
      String attrName = getServletContextAttributeName();
      getServletContext().setAttribute(attrName, wac);
   }

   return wac;
}
```

#### 父子容器的概念引入

以前xml配置springMVC的时候步骤，需要在web.xml中配置

1、在web.xml配置ContextLoadListener,指定Spring配置文件位置

2、在web.xml配置DispatcherServlet,指定SpringMVC配置文件的位置

3、以上操作就会产生父子容器

父容器：Root Spring配置文件进行包扫描并保存的组件的容器

子容器：SpringMVC配置文件进行包扫描并保存的所有组件的容器

cwac.setParent(rootContext);好处是容器之间的隔离，类似于Java中类加载的双亲委派模型

### 基于两个事件回调启动了Spring和SpringMVC

#### 第一个父容器相关

AbstractAnnotationConfigDispatcherServletInitializer

![image-20220515145511744](https://file.hyqup.cn/img/image-20220515145511744.png)

AbstractContextLoaderInitializer 注册ContextLoaderListener，web应用启动以后（Tomcat加载应用以后）会执行ContextLoaderListener里面contextInitialized的逻辑，监听器机制，servlet的标准

```java
@Override
public void onStartup(ServletContext servletContext) throws ServletException {
   registerContextLoaderListener(servletContext);
}
```

```
protected void registerContextLoaderListener(ServletContext servletContext) {
   WebApplicationContext rootAppContext = createRootApplicationContext();
   if (rootAppContext != null) {
      ContextLoaderListener listener = new ContextLoaderListener(rootAppContext);
      listener.setContextInitializers(getRootApplicationContextInitializers());
      servletContext.addListener(listener);
   }
   else {
      logger.debug("No ContextLoaderListener registered, as " +
            "createRootApplicationContext() did not return an application context");
   }
}
```

createRootApplicationContext由**孙子类**AbstractAnnotationConfigDispatcherServletInitializer 实现

```java
protected WebApplicationContext createRootApplicationContext() {
   Class<?>[] configClasses = getRootConfigClasses();
   if (!ObjectUtils.isEmpty(configClasses)) {
      AnnotationConfigWebApplicationContext context = new AnnotationConfigWebApplicationContext();
      context.register(configClasses);
      return context;
   }
   else {
      return null;
   }
}
```

而 getRootConfigClasses 就是获取自子类的配置文件，也就是我们所说父子容器中父容器Spring组件相关的配置类

#### 第二个子容器相关

第二个我们发现 AbstractAnnotationConfigDispatcherServletInitializer  有一个getRootConfigClasses 同时也有一个getRootConfigClasses，获取和servlet相关的。追溯发现

AbstractDispatcherServletInitializer

```java
protected void registerDispatcherServlet(ServletContext servletContext) {
   String servletName = getServletName();
   Assert.hasLength(servletName, "getServletName() must not return null or empty");

   WebApplicationContext servletAppContext = createServletApplicationContext();
   Assert.notNull(servletAppContext, "createServletApplicationContext() must not return null");

   FrameworkServlet dispatcherServlet = createDispatcherServlet(servletAppContext);
   Assert.notNull(dispatcherServlet, "createDispatcherServlet(WebApplicationContext) must not return null");
   dispatcherServlet.setContextInitializers(getServletApplicationContextInitializers());

   ServletRegistration.Dynamic registration = servletContext.addServlet(servletName, dispatcherServlet);
   if (registration == null) {
      throw new IllegalStateException("Failed to register servlet with name '" + servletName + "'. " +
            "Check if there is another servlet registered under the same name.");
   }

   registration.setLoadOnStartup(1);
   registration.addMapping(getServletMappings());
   registration.setAsyncSupported(isAsyncSupported());

   Filter[] filters = getServletFilters();
   if (!ObjectUtils.isEmpty(filters)) {
      for (Filter filter : filters) {
         registerServletFilter(servletContext, filter);
      }
   }

   customizeRegistration(registration);
}
```

createServletApplicationContext 又是由**孙子类**AbstractAnnotationConfigDispatcherServletInitializer实现

```java
@Override
protected WebApplicationContext createServletApplicationContext() {
   AnnotationConfigWebApplicationContext context = new AnnotationConfigWebApplicationContext();
   Class<?>[] configClasses = getServletConfigClasses();
   if (!ObjectUtils.isEmpty(configClasses)) {
      context.register(configClasses);
   }
   return context;
}
```

#### demo演示

```java
@ComponentScan(value = "cn.hyqup.web",excludeFilters = {
      @ComponentScan.Filter(type = FilterType.ANNOTATION,value = Controller.class)
})
@Configuration
public class SpringConfig {
}
```

```java
@ComponentScan(value = "cn.hyqup.web",includeFilters = {
      @ComponentScan.Filter(type = FilterType.ANNOTATION,value = Controller.class)
},useDefaultFilters = false)
@Configuration
public class SpringMVCConfig {
}
```

```java
public class QuickAppStater extends AbstractAnnotationConfigDispatcherServletInitializer {
   /**
    * 根容器的配置（Spring容器相关的配置）
    * @return
    */
   @Override
   protected Class<?>[] getRootConfigClasses() {
      return new Class<?>[]{SpringConfig.class };
   }

   /**
    * web容器相关的配置（SpringMVC的配置类）
    * @return
    */
   @Override
   protected Class<?>[] getServletConfigClasses() {
      return new Class<?>[]{SpringMVCConfig.class };
   }

   /**
    * Servlet的映射路径
    * @return
    */
   @Override
   protected String[] getServletMappings() {
      return new String[]{"/"};
   }
}
```

继承了AbstractAnnotationConfigDispatcherServletInitializer，实现了快速启动spring容器和springmvc容器

### 整体流程分析

#### 代码阶段

1. Tomcat启动后会扫描所有的WebApplicationInitializer执行onStartup方法，会扫描到我们所写的QuickAppStater，执行onStartup，会执行super.onStartup
2. 会执行到AbstractDispatcherServletInitializer的方法首先执行父类AbstractContextLoaderInitializer去执行registerContextLoaderListener，会根据spring的配置类创建一个空容器AnnotationConfigWebApplicationContext，注解版本的。此时容器还未刷新，没有功能 
3. AbstractDispatcherServletInitializer其次会执行registerDispatcherServlet注册DispatcherServlet，根据web相关的配置类AnnotationConfigWebApplicationContext，第二个容器创建出来

#### 流程分析

1. 首先按照代码阶段会将我们的父子容器创建出来，也就是两个AnnotationConfigWebApplicationContext，
2. （父容器）web启动完成的时候，Tomcat触发监听器启动根容器，也就是ContextLoaderListener里面的contextInitialized进行initWebApplicationContext初始化，也就是会执行到refresh()逻辑装配Spring相关的容器（比如AOP、事务、IOC、自动装配（包括了service层、dao层））
3. （子容器）由于DispatcherServlet本质上就是一个servlet,所以tomcat启动之后回调用init方法初始化，这个时候就是执行到对DispatcherServlet里面的AnnotationConfigWebApplicationContext进行refresh装配web相关的SpringMVC相关的容器（比如@Controller相关的）

注意：由于设计上是子容器刷新的时候父容器已经刷新完毕，并且将父容器设置到子容器对象中，所以我们在controller（子容器）装配service（父容器）正常，而service装配controller就是不支持的
