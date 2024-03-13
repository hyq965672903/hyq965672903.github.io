---
title: Kubernetes(四)-集群核心概念Controller
index_img: /img/default.png
banner_img: /img/default.png
date: 2023-06-26 22:45:33
tags:
categories:
description:
---

Controller作用分类、进阶案例使用

<!-- more -->

## Controller 作用及分类

> controller用于控制pod，当发生各种故障导致系统状态发生变化时，会尝试将系统状态修复到“期望状态”。

### 分类

* **Deployment**   部署无状态应用，控制pod升级,回退
* **ReplicaSet**       副本集,控制pod扩容,裁减
* **ReplicationController**(相当于ReplicaSet的老版本,现在建议使用Deployments加ReplicaSet替代RC)
* **StatefulSets**     部署有状态应用，结合Service、存储等实现对有状态应用部署
* **DaemonSet**     守护进程集，运行在所有集群节点(包括master), 比如使用filebeat,node_exporter
* **Jobs**                  一次性
* **Cronjob**            周期性

注：一般是 Deployment 和 ReplicaSet

### Deployment
