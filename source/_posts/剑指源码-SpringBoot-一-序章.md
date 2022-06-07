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
