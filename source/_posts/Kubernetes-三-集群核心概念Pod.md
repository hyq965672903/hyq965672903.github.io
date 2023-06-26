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

### pod的YAML资源清单格式

```shell
# yaml格式的pod定义文件完整内容：
apiVersion: v1       #必选，api版本号，例如v1
kind: Pod       	#必选，Pod
metadata:       	#必选，元数据
  name: string       #必选，Pod名称
  namespace: string    #Pod所属的命名空间,默认在default的namespace
  labels:     		 # 自定义标签
    name: string     #自定义标签名字
  annotations:        #自定义注释列表
    name: string
spec:         #必选，Pod中容器的详细定义(期望)
  containers:      #必选，Pod中容器列表
  - name: string     #必选，容器名称
    image: string    #必选，容器的镜像名称
    imagePullPolicy: [Always | Never | IfNotPresent] #获取镜像的策略 Alawys表示下载镜像 IfnotPresent表示优先使用本地镜像，否则下载镜像，Nerver表示仅使用本地镜像
    command: [string]    #容器的启动命令列表，如不指定，使用打包时使用的启动命令
    args: [string]     #容器的启动命令参数列表
    workingDir: string     #容器的工作目录
    volumeMounts:    #挂载到容器内部的存储卷配置
    - name: string     #引用pod定义的共享存储卷的名称，需用volumes[]部分定义的的卷名
      mountPath: string    #存储卷在容器内mount的绝对路径，应少于512字符
      readOnly: boolean    #是否为只读模式
    ports:       #需要暴露的端口库号列表
    - name: string     #端口号名称
      containerPort: int   #容器需要监听的端口号
      hostPort: int    #容器所在主机需要监听的端口号，默认与Container相同
      protocol: string     #端口协议，支持TCP和UDP，默认TCP
    env:       #容器运行前需设置的环境变量列表
    - name: string     #环境变量名称
      value: string    #环境变量的值
    resources:       #资源限制和请求的设置
      limits:      #资源限制的设置
        cpu: string    #Cpu的限制，单位为core数，将用于docker run --cpu-shares参数
        memory: string     #内存限制，单位可以为Mib/Gib，将用于docker run --memory参数
      requests:      #资源请求的设置
        cpu: string    #Cpu请求，容器启动的初始可用数量
        memory: string     #内存清求，容器启动的初始可用数量
    livenessProbe:     #对Pod内个容器健康检查的设置，当探测无响应几次后将自动重启该容器，检查方法有exec、httpGet和tcpSocket，对一个容器只需设置其中一种方法即可
      exec:      #对Pod容器内检查方式设置为exec方式
        command: [string]  #exec方式需要制定的命令或脚本
      httpGet:       #对Pod内个容器健康检查方法设置为HttpGet，需要制定Path、port
        path: string
        port: number
        host: string
        scheme: string
        HttpHeaders:
        - name: string
          value: string
      tcpSocket:     #对Pod内个容器健康检查方式设置为tcpSocket方式
         port: number
       initialDelaySeconds: 0  #容器启动完成后首次探测的时间，单位为秒
       timeoutSeconds: 0   #对容器健康检查探测等待响应的超时时间，单位秒，默认1秒
       periodSeconds: 0    #对容器监控检查的定期探测时间设置，单位秒，默认10秒一次
       successThreshold: 0
       failureThreshold: 0
       securityContext:
         privileged:false
    restartPolicy: [Always | Never | OnFailure] # Pod的重启策略，Always表示一旦不管以何种方式终止运行，kubelet都将重启，OnFailure表示只有Pod以非0退出码退出才重启，Nerver表示不再重启该Pod
    nodeSelector: obeject  # 设置NodeSelector表示将该Pod调度到包含这个label的node上，以key：value的格式指定
    imagePullSecrets:    #Pull镜像时使用的secret名称，以key：secretkey格式指定
    - name: string
    hostNetwork: false     #是否使用主机网络模式，默认为false，如果设置为true，表示使用宿主机网络
    volumes:       #在该pod上定义共享存储卷列表
    - name: string     #共享存储卷名称 （volumes类型有很多种）
      emptyDir: {}     #类型为emtyDir的存储卷，与Pod同生命周期的一个临时目录。为空值
      hostPath: string     #类型为hostPath的存储卷，表示挂载Pod所在宿主机的目录
        path: string     #Pod所在宿主机的目录，将被用于同期中mount的目录
      secret:      #类型为secret的存储卷，挂载集群与定义的secret对象到容器内部
        scretname: string  
        items:     
        - key: string
          path: string
      configMap:     #类型为configMap的存储卷，挂载预定义的configMap对象到容器内部
        name: string
        items:
        - key: string
          path: string
```

注：如果不清楚某个参数的写法，可使用 kubectl explain 命令完成

```shell
kubectl explain pod
kubectl explain pod.spec
```

## pod调度

> 以使用约束把pod调度到指定的node节点

### 调度约束方法

- nodeName 用于将pod调度到指定的node名称上
- nodeSelector 用于将pod调度到匹配Label的node上

**第一种**：spec.nodeName 将容器调度到指定节点上

```shell
[root@k8s-master1 ~]# vim pod-nodename.yml
apiVersion: v1
kind: Pod
metadata:
  name: pod-nodename
spec:
  nodeName: k8s-worker1                    # 通过nodeName调度到k8s-worker1节点
  containers:
  - name: nginx
    image: nginx:1.15-alpine
```

**第二种**：spec.nodeSelector nodeSelector节点选择器调度到指定节点上去

```shell
[root@k8s-master1 ~]# vim pod-nodeselector.yml
apiVersion: v1
kind: Pod
metadata:
  name: pod-nodeselect
spec:
  nodeSelector:                         # nodeSelector节点选择器
    bussiness: game                     # 指定调度到标签为bussiness=game的节点
  containers:
  - name: nginx
    image: nginx:1.15-alpine
```

## 生命周期

### pod生命周期

* 有些pod(比如运行httpd服务),正常情况下会一直运行中,但如果手动删除它,此pod会终止
* 也有些pod(比如执行计算任务)，任务计算完后就会自动终止

上面两种场景中,pod从创建到终止的过程就是pod的生命周期。

#### 容器启动

启动后钩子`post-start`执行后执行做**健康检查**

* 第一个健康检查叫存活状态检查(`liveness probe` )，用来检查主容器**存活状态**的

* 第二个健康检查叫准备就绪检查(`readiness probe`)，用来检查主容器是否**启动就绪**

#### 容器重启策略

*  **Always**：表示容器挂了总是重启，这是默认策略 

*  **OnFailures**：表示容器状态为错误时才重启，也就是容器正常终止时不重启 

*  **Never**：表示容器挂了不予重启 

*  对于Always这种策略，容器只要挂了，就会立即重启，这样是很耗费资源的。所以Always重启策略是这么做的：第一次容器挂了立即重启，如果再挂了就要延时10s重启，第三次挂了就等20s重启...... 依次类推 

### HealthCheck健康检查

当Pod启动时，容器可能会因为某种错误(服务未启动或端口不正确)而无法访问等。

| 方式                         | 说明                                                         |
| ---------------------------- | ------------------------------------------------------------ |
| Liveness Probe(存活状态探测) | 指示容器是否正在运行。如果存活态探测失败，则 kubelet 会杀死容器， 并且容器将根据其[重启策略](https://kubernetes.io/zh/docs/concepts/workloads/pods/pod-lifecycle/#restart-policy)决定未来。如果容器不提供存活探针， 则默认状态为 `Success`。 |
| readiness Probe(就绪型探测)  | 指示容器是否准备好为请求提供服务。如果就绪态探测失败， 端点控制器将从与 Pod 匹配的所有服务的端点列表中删除该 Pod 的 IP 地址。 初始延迟之前的就绪态的状态值默认为 `Failure`。 如果容器不提供就绪态探针，则默认状态为 `Success`。注：检查后不健康，将容器设置为Notready;如果使用service来访问,流量不会转发给此种状态的pod |
| startup Probe                | 指示容器中的应用是否已经启动。如果提供了启动探针，则所有其他探针都会被 禁用，直到此探针成功为止。如果启动探测失败，`kubelet` 将杀死容器，而容器依其 [重启策略](https://kubernetes.io/zh/docs/concepts/workloads/pods/pod-lifecycle/#restart-policy)进行重启。 如果容器没有提供启动探测，则默认状态为 `Success`。 |

#### Probe探测方式

| 方式    | 说明                                                         |
| ------- | ------------------------------------------------------------ |
| Exec    | 执行命令                                                     |
| HTTPGet | http请求某一个URL路径                                        |
| TCP     | tcp连接某一个端口                                            |
| gRPC    | 使用 [gRPC](https://grpc.io/) 执行一个远程过程调用。 目标应该实现 [gRPC健康检查](https://grpc.io/grpc/core/md_doc_health-checking.html)。 如果响应的状态是 "SERVING"，则认为诊断成功。 gRPC 探针是一个 alpha 特性，只有在你启用了 "GRPCContainerProbe" [特性门控](https://kubernetes.io/zh/docs/reference/command-line-tools-reference/feature-gate/)时才能使用。 |

####  liveness-exec案例

1、准备资源清单文件pod-liveness-exec.yml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: liveness-exec
  namespace: default
spec:
  containers:
  - name: liveness
    image: busybox
    imagePullPolicy: IfNotPresent
    args:
    - /bin/sh
    - -c
    - touch /tmp/healthy; sleep 30; rm -rf /tmp/healthy; sleep 600
    livenessProbe:
      exec:
        command:
        - cat
        - /tmp/healthy
      initialDelaySeconds: 5 				# pod启动延迟5秒后探测
      periodSeconds: 5 						# 每5秒探测1次
```

2、应用yaml文件

```shell
kubectl apply -f pod-liveness-exec.yml
```

#### liveness-httpget案例

1、准备资源清单文件pod-liveness-httpget.yml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: liveness-httpget
  namespace: default
spec:
  containers:
  - name: liveness
    image: nginx:1.15-alpine
    imagePullPolicy: IfNotPresent
    ports:							    # 指定容器端口，这一段不写也行，端口由镜像决定 
    - name: http						# 自定义名称，不需要与下面的port: http对应
      containerPort: 80					# 类似dockerfile里的expose 80
    livenessProbe:
      httpGet:                          # 使用httpGet方式
        port: http                      # http协议,也可以直接写80端口
        path: /index.html               # 探测家目录下的index.html
      initialDelaySeconds: 3            # 延迟3秒开始探测
      periodSeconds: 5                  # 每隔5s钟探测一次
```

2、应用YAML文件

```shell
kubectl apply -f pod-liveness-httpget.yml
```

#### liveness-tcp案例

1、准备资源文件pod-liveness-tcp.yml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: liveness-tcp
  namespace: default
spec:
  containers:
  - name: liveness
    image: nginx:1.15-alpine
    imagePullPolicy: IfNotPresent
    ports:
    - name: http
      containerPort: 80
    livenessProbe:
      tcpSocket:                        # 使用tcp连接方式
        port: 80                        # 连接80端口进行探测
      initialDelaySeconds: 3
      periodSeconds: 5
```

2、应用YAML文件

```shell
kubectl apply -f pod-liveness-tcp.yml
```

#### readiness案例

1、准备资源配置文件pod-readiness-httpget.yml

```shell
[root@k8s-master1 ~]# vim pod-readiness-httpget.yml
apiVersion: v1
kind: Pod
metadata:
  name: readiness-httpget
  namespace: default
spec:
  containers:
  - name: readiness
    image: nginx:1.15-alpine
    imagePullPolicy: IfNotPresent
    ports:
    - name: http
      containerPort: 80
    readinessProbe:                     # 这里由liveness换成了readiness
      httpGet:
        port: http
        path: /index.html
      initialDelaySeconds: 3
      periodSeconds: 5
```

2、应用YAML文件

```shell
kubectl apply -f pod-readiness-httpget.yml
```

### post-start

> 容器启动后执行的命令

1、准备资源清单文件pod-poststart.yml

```shell
apiVersion: v1
kind: Pod
metadata:
  name: poststart
  namespace: default
spec:
  containers:
  - name: poststart
    image: nginx:1.15-alpine
    imagePullPolicy: IfNotPresent
    lifecycle:                                       # 生命周期事件
      postStart:
        exec:
          command: ["mkdir","-p","/usr/share/nginx/html/haha"]
```

2、应用资源文件

```shell
kubectl apply -f pod-poststart.yml
```

3、查看是否创建成功

```shell
kubectl exec -it poststart -- ls /usr/share/nginx/html -l
```

### pre-stop

> 容器终止前执行的命令

1、准备资源清单文件prestop.yml

```shell
apiVersion: v1
kind: Pod
metadata:
  name: prestop
  namespace: default
spec:
  containers:
  - name: prestop
    image: nginx:1.15-alpine
    imagePullPolicy: IfNotPresent
    lifecycle:                                       # 生命周期事件
      preStop:                                       # preStop
        exec:
          command: ["/bin/sh","-c","sleep 60000000"]     # 容器终止前sleep 60000000秒
```

2、应用资源文件

```shell
kubectl apply -f prestop.yml
```

3、删除验证

```shell
kubectl delete -f prestop.yml
```

