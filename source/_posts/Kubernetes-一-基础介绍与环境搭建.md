---
title: Kubernetes(一)-基础介绍与环境搭建
index_img: /img/default.png
banner_img: /img/default.png
date: 2023-06-12 20:38:33
tags: 
 - k8s
categories:
 - 云原生
description:
---

Kubernetes 介绍、主要功能、本地虚拟机环境搭建、

<!-- more -->

## Kubernetes介绍

> 希腊语：舵手、飞行员

- Kubernetes是一个全新的基于容器技术的分布式架构领先方案, 它是Google在2014年6月开源的一个容器集群管理系统，使用Go语言开发，Kubernetes也叫K8S。K8S是Google内部一个叫Borg的容器集群管理系统衍生出来的，Borg已经在Google大规模生产运行十年之久。K8S主要用于自动化部署、扩展和管理容器应用，提供了资源调度、部署管理、服务发现、扩容缩容、监控等一整套功能。2015年7月，Kubernetes v1.0正式发布。

- Kubernetes作为一个容器集群管理系统，用于管理容器云平台中多个主机上的容器应用，Kubernetes的目标是让部署容器化的应用变得简单且高效，所以 Kubernetes 提供了应用部署，规划，更新，维护的一整套完整的机制。
- 除了Docker容器之外，Kubernetes还支持其他多种容器，如 Containerd、rkt、CoreOS 等

## 认识Kubernetes

### 起源

- 源自于谷歌Borg


- 使用golang语言开发


- 简称为k8s

### 归属

现归属于CNCF

- 云原生(CloudNative)计算基金会

- 是一个开源软件基金会，致力于使云计算普遍性和持续性

- 官方：http://www.cncf.io

### kubernetes版本

- 2014年9月第一个正式版本
- 2015年7月1.0版本正式发布
- 现在最新版本为1.27
- 主要贡献者：Google,Redhat,Microsoft,IBM,Intel
- 代码托管github:<https://github.com/kubernetes/>

### 架构说明

- Master Node
  - 中心节点
  - manager
  - 简单叫法
    - master节点
- Minion Node
  - 工作节点
  - worker
  - 简单叫点
    - node节点
    - worker节点

### 节点组件

#### Master节点组件

master节点是集群管理中心，它的组件可以在集群内任意节点运行，但是为了方便管理所以会在一台主机上运行Master所有组件，**并且不在此主机上运行用户容器**



Master组件包括：

- `kube-apiserver`


​      用于暴露kubernetes API，任何的资源请求/调用操作都是通过kube-apiserver提供的接口进行。

- `kube-controller-manager`


​      控制器管理器，用于对控制器进行管理，它们是集群中处理常规任务的后台线程。

- `kube-scheduler`

  监视新创建没有分配到Node的Pod，为Pod选择一个Node运行。

- `ETCD`

  是kubernetes提供默认的存储系统，保存所有集群数据。

#### Node节点组件

node节点用于运行以及维护Pod, 管理volume(CVI)和网络(CNI)，维护pod及service等信息

Node组件包括：

- `kubelet` 
  - 负责维护容器的生命周期(创建pod，销毁pod)，同时也负责Volume(CVI)和网络(CNI)的管理
- `kube-proxy` 
  - 通过在主机上维护网络规则并执行连接转发来实现service(iptables/ipvs)
  - 随时与apiserver通信，把Service或Pod改变提交给apiserver，保存至etcd（可做高可用集群）中，负责service实现，从内部pod至service和从外部node到service访问。
- `Container Runtime`
  - 容器运行时(Container Runtime)
  - 负责镜像管理以及Pod和容器的真正运行
  - 支持docker/containerd/Rkt/Pouch/Kata等多种运行时

