---
title: Kubernetes(二)-快速使用与核心概念
index_img: /img/default.png
banner_img: /img/default.png
date: 2023-06-21 17:25:01
tags:
categories:
description:
---

 kubectl基础操作、node管理、以及核心概念NameSpace、Pod、Controller、Service

<!-- more -->

# kubectl

## kubectl使用帮助

```shell
kubectl -h
```

## kubectl命令说明

![image-20230621172951828](E:\git_repo\hyq965672903.github.io\source\_posts\Kubernetes-二-快速使用与核心概念.assets\image-20230621172951828.png)

![image-20230621173005878](E:\git_repo\hyq965672903.github.io\source\_posts\Kubernetes-二-快速使用与核心概念.assets\image-20230621173005878.png)

## kubectl命令补全

```shell
yum install -y bash-completion
source /usr/share/bash-completion/bash_completion
source <(kubectl completion bash)
```

# node相关命令

```shell
#查看集群信息
kubectl cluster-info
#查看集群节点信息
kubectl get nodes
#集群节点信息
kubectl get nodes -o wide
#查看节点描述详细信息
kubectl describe node xxx
```

**给节点打标签** lable

```shell
#设置节点标签,标签以key val的形式
kubectl label node xxx  key=val
#查看所有节点标签
kubectl get node --show-labels
```

