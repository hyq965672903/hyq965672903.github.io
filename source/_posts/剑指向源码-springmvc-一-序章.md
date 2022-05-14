---
title: 剑指向源码-springmvc(一)-序章
index_img: /img/default.png
banner_img: /img/default.png
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

