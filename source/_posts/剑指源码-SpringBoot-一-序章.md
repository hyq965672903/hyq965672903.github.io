---
title: 剑指源码-SpringBoot-(一)-序章
index_img: /img/default.png
banner_img: /img/default.png
date: 2022-06-07 22:26:00
tags:
categories:
description:
---

SpringBoot前置，了解EnableMVC实现原理以及作用

<!-- more -->

> **@EnableMvc表示由定制类完全替换默认的springMVC配置，一旦开启**

自定springmvc的组件的时候会导致默认的组件失效，采用EnableMvc这种方式会将系统默认加载，再将我们自定义的组件加载，这样都会生效

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@Documented
@Import(DelegatingWebMvcConfiguration.class)
public @interface EnableWebMvc {
}
```

EnableWebMvc导入了DelegatingWebMvcConfiguration

```java
@Configuration(proxyBeanMethods = false)
public class DelegatingWebMvcConfiguration extends WebMvcConfigurationSupport {

   private final WebMvcConfigurerComposite configurers = new WebMvcConfigurerComposite();


   @Autowired(required = false)
   public void setConfigurers(List<WebMvcConfigurer> configurers) {
      if (!CollectionUtils.isEmpty(configurers)) {
         this.configurers.addWebMvcConfigurers(configurers);
      }
   }
   ...
```

DelegatingWebMvcConfiguration会加载所有的WebMvcConfigurer并添加到configurers，以模板方法的模式。父类会加载SpringMVC自己本身有的组件，我们也可以自定义这些组件

## SpringBoot导入很多配置类

> @SpringBootApplication是一个复合型注解，包含了@EnableAutoConfiguration注解
>
> 而@EnableAutoConfiguration 又包含如下
>
> ```
> @AutoConfigurationPackage
> @Import(AutoConfigurationImportSelector.class)
> ```
>
> @AutoConfigurationPackage又导入了@Import(AutoConfigurationPackages.Registrar.class)

@AutoConfigurationPackage指定了我们以后要扫描哪些包下的所有组件

@Import(AutoConfigurationImportSelector.class) 导入组件，去类路径下找META-INF/spring.factories，这些配置类根据条件以@Bean的形式导入容器

## Spring在容器刷新的onRefresh()的时候启动Tomcat

 `ServletWebServerFactoryAutoConfiguration`

@Import ServletWebServerFactoryAutoConfiguration.BeanPostProcessorsRegistrar.class

另外还导入了三种web服务器，Tomcat,Jetty,Undertow。默认是Tomcat

而在EmbeddedTomcat中又采用TomcatServletWebServerFactory工厂来创建容器

 TomcatServletWebServerFactory中就是传统的以new的形式创建了Tomcat服务器

```java
@Override
public WebServer getWebServer(ServletContextInitializer... initializers) {
   if (this.disableMBeanRegistry) {
      Registry.disableRegistry();
   }
   Tomcat tomcat = new Tomcat();
   File baseDir = (this.baseDirectory != null) ? this.baseDirectory : createTempDir("tomcat");
   tomcat.setBaseDir(baseDir.getAbsolutePath());
   for (LifecycleListener listener : this.serverLifecycleListeners) {
      tomcat.getServer().addLifecycleListener(listener);
   }
   Connector connector = new Connector(this.protocol);
   connector.setThrowOnFailure(true);
   tomcat.getService().addConnector(connector);
   customizeConnector(connector);
   tomcat.setConnector(connector);
   tomcat.getHost().setAutoDeploy(false);
   configureEngine(tomcat.getEngine());
   for (Connector additionalConnector : this.additionalTomcatConnectors) {
      tomcat.getService().addConnector(additionalConnector);
   }
   prepareContext(tomcat.getHost(), initializers);
   return getTomcatWebServer(tomcat);
}
```

### 调用时机

 ServletWebServerApplicationContext 在容器刷新12大步refresh的时候执行onfresh来启动web容器

```
@Override
protected void onRefresh() {
   super.onRefresh();
   try {
      createWebServer();
   }
   catch (Throwable ex) {
      throw new ApplicationContextException("Unable to start web server", ex);
   }
}
```

