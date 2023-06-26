---
title: Kubernetes(三)-集群核心概念Pod
index_img: /img/default.png
banner_img: /img/default.png
date: 2023-06-26 15:39:40
tags:
categories:
description:
---

Pod的介绍、创建、调度及生命周期

<!-- more -->

## 介绍

### 定义

- Pod(豌豆荚) 是Kubernetes集群管理（创建、部署）与调度的最小计算单元，表示处于运行状态的一组容器
- Pod不是进程，而是容器运行的环境。
- 一个Pod可以封装**一个容器或多个容器(主容器或sidecar边车容器)**
- 一个pod内的多个容器之间共享部分命名空间，例如：Net Namespace,UTS Namespace,IPC Namespace及存储资源
- pod内的IP不是固定的，集群外不能直接访问pod

### 分类

- ###### 静态Pod 

  也称之为“无控制器管理的自主式pod”，直接由特定节点上的 `kubelet` 守护进程管理， 不需要API 服务器看到它们

- ###### 控制器管理的pod

​		控制器可以控制pod的副本数，扩容与裁剪，版本更新与回滚等

### 查看pod

```shell
#查看pod
kubectl get pod
#查看命名空间下的pod
kubectl get pod -n kube-system
```

