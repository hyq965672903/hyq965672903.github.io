---
title: Kubernetes(五)-集群核心概念Service
index_img: /img/default.png
banner_img: /img/default.png
date: 2023-06-26 22:50:33
tags:
categories:
description:
---

Service概念及作用

<!-- more -->

## 概述

> 使用kubernetes集群运行工作负载时，由于Pod经常处于用后即焚状态，Pod经常被重新生成，因此Pod对应的IP地址也会经常变化，导致无法直接访问Pod提供的服务，Kubernetes中使用了Service来解决这一问题，即在Pod前面使用Service对Pod进行代理，无论Pod怎样变化 ，只要有Label，就可以让Service能够联系上Pod，把PodIP地址添加到Service对应的端点列表（Endpoints）实现对Pod IP跟踪，进而实现通过Service访问Pod目的。

- 通过service为pod客户端提供访问pod方法，即可客户端访问pod入口
- 通过标签动态感知pod IP地址变化等
- 防止pod失联
- 定义访问pod访问策略
- 通过label-selector相关联
- 通过Service实现Pod的负载均衡（TCP/UDP 4层）
- 底层实现由kube-proxy通过userspace、iptables、ipvs三种代理模式

## service类型

- **ClusterIP**
  - 默认，分配一个集群内部可以访问的虚拟IP

- **NodePort**
  - 在每个Node上分配一个端口作为外部访问入口
  - nodePort端口范围为:30000-32767

- **LoadBalancer**
  - 工作在特定的Cloud Provider上，例如Google Cloud，AWS，OpenStack

- **ExternalName**
  - 表示把集群外部的服务引入到集群内部中来，即实现了集群内部pod和集群外部的服务进行通信

### Service参数

- **`port`**             访问service使用的端口

- **`targetPort`**  Pod中容器端口

- **`nodePort`**   通过Node实现外网用户访问k8s集群内service (30000-32767)

## Service创建

### NodePort类型

创建资源清单文件

```yaml
[root@master01 ~]# cat 05_create_nodeport_service_app.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-app
  labels:
    app: nginx-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx-app
  template:
    metadata:
      labels:
        app: nginx-app
    spec:
      containers:
      - name: c1
        image: nginx:1.15-alpine
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-app
spec:
  type: NodePort
  selector:
    app: nginx-app
  ports:
  - protocol: TCP
    nodePort: 30001
    port: 8060
    targetPort: 80
```

应用资源清单文件

```shell
kubectl apply -f 05_create_nodeport_service_app.yaml
```

验证创建

```shell
kubectl get deployment.apps
kubectl get svc
kubectl get endpoints
```
