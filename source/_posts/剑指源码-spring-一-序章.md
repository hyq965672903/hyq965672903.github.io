---
title: 剑指源码-spring(一)-序章
index_img: /img/default.png
banner_img: /img/default.png
date: 2022-04-22 19:53:11
tags:
categories:
description:
---

 快速了解spring IOC容器的启动流程

<!-- more -->

这里以我们开发常用的注解形式

1. 创建AnnotationConfigApplicationContext 对象，这个对象继承于GenericApplicationContext，而GenericApplicationContext对象中有DefaultListableBeanFactory；
2. 创建 DefaultListableBeanFactory之后会解析和扫描注解上的所有bean信息封装成BeanDefinition信息，BeanDefinition中有Spring的所有bean的相关定义描述信息；
3. 先将bean进行实例化，然后根据BeanDefinition的信息，底层采用反射的信息进行创建具体的bean对象；
4. 将实例化后的bean对象放到一个Map中，后续应用就从Map中获取Bean对象，该Map也就是我们的IOC容器



ps：其中有许多细节都没说，比如这里的Map 存在多个，解决了什么问题，创建的过程细节等等，接下来就开是进入源码解析的具体学习



##  准备工作

## 源码版本

  github地址  [v5.2.0.RELEASE](https://github.com/spring-projects/spring-framework/releases/tag/v5.2.0.RELEASE)  

源码分析地址

https://github.com/hyq965672903/sourcecode-learn-spring.git

相关环境：

```text
1、操作系统：Windows10
2、JDK 版本：Jdk8
3、IDE 工具：IntelliJ IDEA 2021.2
4、项目构建工具：gradle-5.6.2
```

## 遇到的问题及解决方法

1、配置环境变量  GRADLE_USER_HOME

类似于maven 的本地仓库地址 

![image-20220423085321489](https://file.hyqup.cn/img/image-20220423085321489.png)

2、gradle 本地配置阿里云镜像

gradle 安装目录下 init.d 下面新加init.gradle文件，内容如下

```groovy

allprojects{
    repositories {
        def ALIYUN_REPOSITORY_URL = 'http://maven.aliyun.com/nexus/content/groups/public'
        def ALIYUN_JCENTER_URL = 'http://maven.aliyun.com/nexus/content/repositories/jcenter'
        def GRADLE_LOCAL_RELEASE_URL = 'https://repo.gradle.org/gradle/libs-releases-local'
        def ALIYUN_SPRING_RELEASE_URL = 'https://maven.aliyun.com/repository/spring-plugin'

        all { ArtifactRepository repo ->
            if(repo instanceof MavenArtifactRepository){
                def url = repo.url.toString()
                if (url.startsWith('https://repo1.maven.org/maven2')) {
                    project.logger.lifecycle "Repository ${repo.url} replaced by $ALIYUN_REPOSITORY_URL."
                    remove repo
                }
                if (url.startsWith('https://jcenter.bintray.com/')) {
                    project.logger.lifecycle "Repository ${repo.url} replaced by $ALIYUN_JCENTER_URL."
                    remove repo
                }
                if (url.startsWith('http://repo.spring.io/plugins-release')) {
                    project.logger.lifecycle "Repository ${repo.url} replaced by $ALIYUN_SPRING_RELEASE_URL."
                    remove repo
                }

            }
        }
        maven {
            url ALIYUN_REPOSITORY_URL
        }

        maven {
            url ALIYUN_JCENTER_URL
        }
        maven {
            url ALIYUN_SPRING_RELEASE_URL
        }
        maven {
            url GRADLE_LOCAL_RELEASE_URL
        }

    }


}

```

3、Idea配置本地变量

![image-20220423090000807](https://file.hyqup.cn/img/image-20220423090000807.png)



