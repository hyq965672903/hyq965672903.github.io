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

# 声明式文件yaml

| 参数名                                      | 字段类型 | 说明                                                         |
| ------------------------------------------- | -------- | ------------------------------------------------------------ |
| version                                     | String   | 这里是指的是K8S API的版本，目前基本上是v1，可以用 kubectl api-versions命令查询 |
| kind                                        | String   | 这里指的是yam文件定义的资源类型和角色，比如:Pod              |
| metadata                                    | Object   | 元数据对象，固定值就写 metadata                              |
| metadata.name                               | String   | 元数据对象的名字，这里由我们编写，比如命名Pod的名字          |
| metadata.namespace                          | String   | 元数据对象的命名空间，由我们自身定义                         |
| Spec                                        | Object   | 详细定义对象，固定值就写Spec                                 |
| spec. containers[]                          | list     | 这里是Spec对象的容器列表定义，是个列表                       |
| spec containers [].name                     | String   | 这里定义容器的名字                                           |
| spec.containers [].image                    | String   | 这里定义要用到的镜像名称                                     |
| spec.containers [].imagePullPolicy          | String   | 定义镜像拉取策路，有 Always、 Never、Ifnotpresent三个值可选：(1) Always:意思是每次都尝试重新拉取镜像；(2) Never:表示仅使用本地镜像；(3) IfNotPresent:如果本地有镜像就使用本地镜像，没有就拉取在线镜像。上面三个值都没设置的话，默认是 Always。 |
| spec containers [].command[]                | List     | 指定容器启动命令，因为是数组可以指定多个。不指定则使用镜像打包时使用的启动命令。 |
| spec.containers [].args                     | List     | 指定容器启动命令参数，因为是数组可以指定多个.                |
| spec.containers [].workDir                  | String   | 指定容器的工作目录                                           |
| spec.containers[]. volumeMounts[]           | List     | 指定容器内部的存储卷配置                                     |
| spec.containers[]. volumeMounts[].name      | String   | 指定可以被容器挂载的存储卷的名称                             |
| spec.containers[]. volumeMounts[].mountPath | String   | 指定可以被容器挂载的存储卷的路径                             |
| spec.containers[]. volumeMounts[].readOnly  | String   | 设置存储卷路径的读写模式，ture或者 false，默认为读写模式     |
| spec.containers [].ports[]                  | String   | 指容器需要用到的端口列表                                     |
| spec.containers [].ports[].name             | String   | 指定端口名称                                                 |
| spec.containers [].ports[].containerPort    | String   | 指定容器需要监听的端口号                                     |
| spec.containers [].ports[].hostPort         | String   | 指定容器所在主机需要监听的端口号，默认跟上面 containerPort相同，注意设置了 hostPort同一台主机无法启动该容器的相同副本(因为主机的端口号不能相同，这样会冲突) |
| spec.containers [].ports[].protocol         | String   | 指定端口协议，支持TCP和UDP，默认值为TCP                      |
| spec.containers [].env[]                    | List     | 指定容器运行前需设的环境变量列表                             |
| spec.containers [].env[].name               | String   | 指定环境变量名称                                             |
| spec.containers [].env[].value              | String   | 指定环境变量值                                               |
| spec.containers[].resources                 | Object   | 指定资源 限制和资源请求的值（这里开始就是设置容器的资源上限） |
| spec.containers[].resources.limits          | Object   | 指定设置容器运行时资源的运行上限                             |
| spec.containers[].resources.limits.cpu      | String   | 指定CPU限制，单位为core数，将用于docker run -- cpu-shares参数 |
| spec.containers[].resources.limits.memory   | String   | 指定MEM内存的限制，单位为MiB、GiB                            |
| spec.containers[].resources.requests        | Object   | 指定容器启动和调度时的限制设置                               |
| spec.containers[].resources.requests.cpu    | String   | CPU请求，单位为core数，容器启动时初始化可用数量              |
| spec.containers[].resources.requests.memory | String   | 内存请求，单位为MiB、GiB，容器启动时初始化可用数量           |
| sepc.restartPolicy                          | String   | 定义Pod的重启策略，可选值为Always、OnFailure,默认值为Always。1.Always:Pod一旦终止运行，则无论容器时如何终止的，kubelet服务都将重启它。2.OnFailure:只有Pod以非零退出码终止时，kubelet才会重启该容器。如果容器正常结束（退出码为0），则kubelet将不会重启它。3.Never:Pod终止后，kubelet将退出码报告给Master,不会重启该Pod。 |
| spec.nodeSelector                           | Object   | 定义Node的Label过滤标签，以key:value格式指定。               |
| spec.imagePullSecrets                       | Object   | 定义pull镜像时使用secret名称，以name:secretkey格式指定。     |
| spec.hostNetwork                            | Boolean  | 定义是否使用主机网络模式，默认值为false。设置true表示使用宿主机网络，不使用docker网桥，同时设置了true将无法在同一台宿主机上启动第二个副本。 |

```shell
#查看子段的配置说明
kubectl explain xxx
#eg:kubectl explain metadata
```

## 案例

### 创建Namespace

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: test
```

### 创建pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod
spec:
  containers:
  - name: nginx
    image: nginx:latest
    imagePullPolicy: IfNotPresent
```

# 集群命名空间-Namespace

## 概念

* Namespace是对一组资源和对象的抽象集合.
* 常见的 pod, service, deployment 等都是属于某一个namespace的（默认是 default）
* 不是所有资源都属于namespace，如nodes, persistent volume，namespace 等资源则不属于任何 namespace

## 查看

```shell
#获取namespace
kubectl get namespaces
#获取一个namespace下面的所有资源
kubectl get all --namespace=xx
# eg: kubectl get pod --namespace=kube-system

# 获取某一种类型,这里xx 可以pod等
kubectl get xx --namespace=yyy
# eg: kubectl get pod --namespace=kube-system
```

## 创建

命令行

```shell
#命令行创建
kubectl create namespace xxx
#获取namespace
kubectl get ns
```

YAML文件创建

ns.yaml

```yaml
apiVersion: v1							# api版本号
kind: Namespace							# 类型为namespace
metadata:								# 定义namespace的元数据属性
  name: ns2					    		# 定义name属性为ns2
```

```shell
kubectl apply -f ns.yml
```

