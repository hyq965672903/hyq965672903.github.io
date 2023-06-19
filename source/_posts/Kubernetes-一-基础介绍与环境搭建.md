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

## Kubernetes环境搭建

> 使用kubeadm本地化部署目前最新版本kubernetes版本1.27，其他部署方式也可以使用`kubeasz`、`kubekey` 等方式部署

### Linux环境准备

#### 主机系统说明

使用VMware来搭建虚拟机， aliyun镜像库 https://developer.aliyun.com/mirror/

centos7下载链接：https://mirrors.aliyun.com/centos/7.9.2009/isos/x86_64/CentOS-7-x86_64-Minimal-2207-02.iso

#### 虚拟机IP配置

参考：https://www.cnblogs.com/mayhot/p/15964506.html

修改ip的方式

```shell
# cd到网络配置文件路径
cd /etc/sysconfig/network-scripts/
# 编辑ifcfg-en33
vi ifcfg-en33
# 重启网卡
systemctl restart network
# 查看ip
ip addr
```

```shell
TYPE=Ethernet
PROXY_METHOD=none
BROWSER_ONLY=no
BOOTPROTO=static
DEFROUTE=yes
IPV4_FAILURE_FATAL=no
IPV6INIT=yes
IPV6_AUTOCONF=yes
IPV6_DEFROUTE=yes
IPV6_FAILURE_FATAL=no
IPV6_ADDR_GEN_MODE=stable-privacy
NAME=ens33
UUID=56d45dc8-a17d-4eca-852c-97167c783f01
DEVICE=ens33
ONBOOT=yes
IPADDR=192.168.100.11
NETMASK=255.255.255.0
GATEWAY=192.168.100.2
DNS1=192.168.100.2
DNS2=114.114.114.114

```

改动如下 `IPADDR`自己分配 、`GATEWAY` 查看虚拟机的网络`NAT`网络配置

> `BOOTPROTO=static`   
>
> `ONBOOT=yes`
> `IPADDR=192.168.100.11`
> `NETMASK=255.255.255.0`
> `GATEWAY=192.168.100.2`
> `DNS1=192.168.100.2`
> `DNS2=114.114.114.114`

如果重启网络还是连接不上，可能是NetworkManager导致的，关闭这个服务

```shell
systemctl stop NetworkManager
systemctl disable NetworkManager
```

参考：https://www.cnblogs.com/python-wen/p/11607969.html


> 我这里采用静态IP配置，网关设值为192.168.100.0  掩码是255.255.255.0 所以后续分配ip 就可以从 192.168.100.1~192.168.100.255  

#### 主机硬件配置说明

| IP             | CPU  | 内存 | 硬盘 | 主机名   |
| -------------- | ---- | ---- | ---- | -------- |
| 192.168.100.11 | 4C   | 6G   | 50g  | master01 |
| 192.168.100.12 | 4C   | 6G   | 50g  | worker01 |
| 192.168.100.13 | 4C   | 6G   | 50g  | worker02 |

> 注意：这里分配6g内存并不会直接占用系统6g内存给当前虚拟机使用，而是动态去申请的

配置方式从原生的静态IP的纯净的系统中关机，克隆。克隆后重新设置静态ip，然后重启，使用shell工具链接，我这里使用FinalShell链接。

#### 准备工作

##### 主机名配置

```shell
# 设置主机名
hostnamectl set-hostname xxx
```

##### 主机名与IP地址解析

```shell
# 在hosts后面追加内容
vi /etc/hosts
192.168.100.11 master01
192.168.100.12 worker01
192.168.100.13 worker02
```

##### 关闭防火墙配置

```shell
systemctl disable firewalld
systemctl stop firewalld
firewall-cmd --state
```

##### SELINUX配置

> SELinux在Kubernetes中的作用是提供额外的安全层，增强容器化应用程序和整个集群的安全性。它限制容器的访问权限、提供安全策略、保护文件系统，并记录安全事件，有助于保护集群免受恶意行为和攻击。

```shell
sed -ri 's/SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config
```

##### 时间同步设置

```shell
yum install ntpdate -y
ntpdate time1.aliyun.com
```

##### 配置内核转发及网桥过滤

添加内核转发及网桥过滤配置文件

```shell
cat > /etc/sysctl.d/k8s.conf <<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
vm.swappiness = 0
EOF
```

加载内核转发及网桥过滤配置文件

```shell
sysctl -p /etc/sysctl.d/k8s.conf
```



##### 安装ipset及ipvsadm

> 主要用于实现service转发。

安装ipset、ipvsadm

```shell
yum -y install ipset ipvsadm
```

配置ipvsadm模块加载方式

```shell
cat > /etc/sysconfig/modules/ipvs.modules <<EOF
#!/bin/bash
modprobe -- ip_vs
modprobe -- ip_vs_rr
modprobe -- ip_vs_wrr
modprobe -- ip_vs_sh
modprobe -- nf_conntrack
EOF
```

授权、运行、检查是否加载

```shell
chmod 755 /etc/sysconfig/modules/ipvs.modules && bash /etc/sysconfig/modules/ipvs.modules && lsmod | grep -e ip_vs -e nf_conntrack
```

##### 关闭swap分区

在下面这行添加#注释

```shell
vi /etc/fstab

# /dev/mapper/centos-swap swap                    swap    defaults        0 0
```

### 安装版本说明

> k8s官方于 2020 年 12 月宣布弃用 dockershim，此后k8s从 1.2.0 到 1.2.3 版本如果使用 Docker 作为容器运行时会在 kubelet 启动时会打印一个弃用的警告日志，而最终k8s官方在 2022 年 4 月 的 Kubernetes 1.24 版本中完全移除了 dockershim（[弃用dockershim相关问题官方说明](https://link.zhihu.com/?target=https%3A//kubernetes.io/zh-cn/blog/2022/02/17/dockershim-faq/)）

k8s官方在1.24版本以后移除了docker ，后续采用`k8s+containerd`方式进行搭配使用，如果后续还需使用docker 需要安装`cri-docker` 其实也就是`k8s+docker+cri-docker`

对于 k8s+containerd 和 k8s+docker 的两种方案网上也有网友进行了性能测试对比，前者的运行速度、效率都要比后者高，且各大公有云厂商也都往 containerd 切换，因此 k8s+containerd 的组合就成了目前最合适的方案了

我们这里第一简单安装采用低版本的k8s+docker就行，后续会继续出一篇新版本 k8s1.27版本的来做

版本如下：

查阅地址：https://github.com/kubernetes/kubernetes/blob/release-1.22/build/dependencies.yaml

`kubernetes 1.21.0`+`docker 20.10`

### Docker环境准备（所有节点均需要安装）

> docker 20.10 版本安装

#### 获取yum 源

```shell
wget https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo -O /etc/yum.repos.d/docker-ce.repo

# 列出所有的docker 版本 选择指定的版本安装
yum list docker-ce.x86_64 --showduplicates | sort -r
```

#### 安装Docker

```shell
# --setopt=obsoletes=0  告诉Yum在处理软件包依赖关系时不考虑旧的或过时的软件包
yum -y install --setopt=obsoletes=0 docker-ce-20.10.23-3.el7
```

#### 设置docker开机启动并启动docker 

```shell
# 开机启动
systemctl enable docker
# 启动docker
systemctl start docker
```

#### 修改cgroup方式

> cgroup（控制组）是一种用于限制和隔离资源的Linux内核功能。它允许您在共享的主机上为容器分配和管理资源，例如CPU、内存、磁盘和网络等

```shell
# 在/etc/docker/daemon.json添加如下内容
vi /etc/docker/daemon.json
{
        "exec-opts": ["native.cgroupdriver=systemd"]
}
```

#### 配置docker镜像加速

```shell
# 在/etc/docker/daemon.json添加"registry-mirrors": ["https://jjwt39jg.mirror.aliyuncs.com"]
# 下面为当前最终版本
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "registry-mirrors": [
  	"http://jjwt39jg.mirror.aliyuncs.com",
  	"https://registry.docker-cn.com",
	"http://hub-mirror.c.163.com",
	"https://docker.mirrors.ustc.edu.cn"
  ]
}

```

重启docker

```shell
sudo systemctl daemon-reload
sudo systemctl restart docker
```

#### dokcer卸载

```shell
#杀死所有运行容器
docker kill $(docker ps -a -q)
#删除所有容器
docker rm $(docker ps -a -q)
#删除所有镜像
docker rmi $(docker images -q)

#停止docker相关服务
sudo systemctl stop docker.socket
sudo systemctl stop docker.service

#停止docker服务
systemctl stop docker

#删除存储目录
rm -rf /etc/docker
rm -rf /run/docker
rm -rf /var/lib/dockershim
rm -rf /var/lib/docker

#查看docker 安装的包
yum list installed | grep docker

# 卸载docker相关安装包
yum remove docker-*
```



### **安装containerd** 

> 我们当前安装 `kubernetes 1.21.0`+`docker 20.10`，此步骤跳过

#### 安装依赖软件包

```shell
yum -y install yum-utils device-mapper-persistent-data lvm2
```

#### 添加阿里Docker源

```shell
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
```

#### 添加overlay和netfilter模块

```shell
cat >>/etc/modules-load.d/containerd.conf <<EOF
overlay
br_netfilter
EOF
```

#### 安装Containerd，这里安装最新版本

```shell
yum -y install containerd.io
```

#### 创建Containerd的配置文件

```shell
mkdir -p /etc/containerd
 
containerd config default > /etc/containerd/config.toml
 
sed -i '/SystemdCgroup/s/false/true/g' /etc/containerd/config.toml
 
sed -i '/sandbox_image/s/registry.k8s.io/registry.aliyuncs.com\/google_containers/g' /etc/containerd/config.toml
```

#### 启动containerd

```shell
systemctl enable containerd
systemctl start containerd
```

#### 验证containerd是否安装成功

```shell
containerd -v
```

### Kubernetes 1.21.0 （单master）集群部署



#### kubeadm、kubelet、kubectl安装

|          | kubeadm                | kubelet                                       | kubectl                |
| -------- | ---------------------- | --------------------------------------------- | ---------------------- |
| 版本     | 1.21.0                 | 1.21.0                                        | 1.21.0                 |
| 安装位置 | 集群所有主机           | 集群所有主机                                  | 集群所有主机           |
| 作用     | 初始化集群、管理集群等 | 用于接收api-server指令，对pod生命周期进行管理 | 集群应用命令行管理工具 |

##### 配置yum源

```shell
# k8s源,没有就创建
vi /etc/yum.repos.d/kubernetes.repo

[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=0
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
```

##### 安装

```shell
yum install -y kubelet-1.21.0 kubeadm-1.21.0 kubectl-1.21.0
```

##### 配置kubelet

> 保证docker使用的cgroupdriver与kubelet使用的cgroup的一致性

```shell
# vi /etc/sysconfig/kubelet
KUBELET_EXTRA_ARGS="--cgroup-driver=systemd"
```

##### 设置kubelet为开机自启动并启动

```shell
systemctl enable kubelet && systemctl restart kubelet
```

#### 集群初始化（master初始化）

```shell
#镜像清理
docker system prune -a
```



##### 方式一：先下载镜像

 **集群镜像准备**

```shell
kubeadm config images list --kubernetes-version=v1.21.0

#返回如下
k8s.gcr.io/kube-apiserver:v1.21.0
k8s.gcr.io/kube-controller-manager:v1.21.0
k8s.gcr.io/kube-scheduler:v1.21.0
k8s.gcr.io/kube-proxy:v1.21.0
k8s.gcr.io/pause:3.4.1
k8s.gcr.io/etcd:3.4.13-0
k8s.gcr.io/coredns/coredns:v1.8.0
```

**脚本下载**

> 官网 k8s.gcr.io 由于网络原因下载不下来
>
> 这里选用镜像下载
>
>  registry.cn-hangzhou.aliyuncs.com/google_containers/ 该镜像中 pause:3.4.1和 etcd:3.4.13-0 找不到 原因目前未知
>
>  registry.aliyuncs.com/google_containers/ 目前可行 就用它了

```shell
# vi image_download.sh

#!/bin/bash
images_list='
registry.aliyuncs.com/google_containers/kube-apiserver:v1.21.0
registry.aliyuncs.com/google_containers/kube-controller-manager:v1.21.0
registry.aliyuncs.com/google_containers/kube-scheduler:v1.21.0
registry.aliyuncs.com/google_containers/kube-proxy:v1.21.0
registry.aliyuncs.com/google_containers/pause:3.4.1
registry.aliyuncs.com/google_containers/etcd:3.4.13-0
registry.aliyuncs.com/google_containers/coredns/coredns:v1.8.0

for i in $images_list
do
        docker pull $i
done

docker save -o k8s-1-27-0.tar $images_list
```

**执行**

```shell
sh image_download.sh
```

**然后执行集群初始化**

```shell
kubeadm init --kubernetes-version=v1.21.0 --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=192.168.100.11
```

###### 遇到的问题

方式一执行**集群初始化**的时候还是会走k8s.gcr.io，所以应该从阿里云下下来后，重新tag 打成 k8s.gcr.io下面的包再执行初始化

##### 方式二：使用阿里云镜像

```shell
kubeadm init --kubernetes-version=v1.21.0 --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=192.168.100.11 --image-repository=registry.aliyuncs.com/google_containers
```

###### 遇到的问题

failed to pull image registry.aliyuncs.com/google_containers/coredns/coredns:v1.8.0

原因：

**安装时需要从 k8s.gcr.io 拉取镜像，但是该网站被我国屏蔽了，国内没法正常访问导致没法正常进行kubernetes正常安装,从Docker官方默认镜像平台拉取镜像并重新打tag的方式来绕过对 k8s.gcr.io 的访问**

```shell
#从docker官网拉取
docker pull coredns/coredns:1.8.0
#重新打标签
docker tag coredns/coredns:1.8.0 registry.aliyuncs.com/google_containers/coredns/coredns:v1.8.0
# 删除旧的镜像
docker rmi coredns/coredns:1.8.0
```



此时会生成从节点加入主节点的链接

```shell
[init] Using Kubernetes version: v1.21.0
[preflight] Running pre-flight checks
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Generating "ca" certificate and key
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local master01] and IPs [10.96.0.1 192.168.100.11]
[certs] Generating "apiserver-kubelet-client" certificate and key
[certs] Generating "front-proxy-ca" certificate and key
[certs] Generating "front-proxy-client" certificate and key
[certs] Generating "etcd/ca" certificate and key
[certs] Generating "etcd/server" certificate and key
[certs] etcd/server serving cert is signed for DNS names [localhost master01] and IPs [192.168.100.11 127.0.0.1 ::1]
[certs] Generating "etcd/peer" certificate and key
[certs] etcd/peer serving cert is signed for DNS names [localhost master01] and IPs [192.168.100.11 127.0.0.1 ::1]
[certs] Generating "etcd/healthcheck-client" certificate and key
[certs] Generating "apiserver-etcd-client" certificate and key
[certs] Generating "sa" key and public key
[kubeconfig] Using kubeconfig folder "/etc/kubernetes"
[kubeconfig] Writing "admin.conf" kubeconfig file
[kubeconfig] Writing "kubelet.conf" kubeconfig file
[kubeconfig] Writing "controller-manager.conf" kubeconfig file
[kubeconfig] Writing "scheduler.conf" kubeconfig file
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Starting the kubelet
[control-plane] Using manifest folder "/etc/kubernetes/manifests"
[control-plane] Creating static Pod manifest for "kube-apiserver"
[control-plane] Creating static Pod manifest for "kube-controller-manager"
[control-plane] Creating static Pod manifest for "kube-scheduler"
[etcd] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"
[wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests". This can take up to 4m0s
[kubelet-check] Initial timeout of 40s passed.
[apiclient] All control plane components are healthy after 56.002741 seconds
[upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config-1.21" in namespace kube-system with the configuration for the kubelets in the cluster
[upload-certs] Skipping phase. Please see --upload-certs
[mark-control-plane] Marking the node master01 as control-plane by adding the labels: [node-role.kubernetes.io/master(deprecated) node-role.kubernetes.io/control-plane node.kubernetes.io/exclude-from-external-load-balancers]
[mark-control-plane] Marking the node master01 as control-plane by adding the taints [node-role.kubernetes.io/master:NoSchedule]
[bootstrap-token] Using token: 4x919n.wofqxskn85v5skmj
[bootstrap-token] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles
[bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to get nodes
[bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstrap-token] configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstrap-token] configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[bootstrap-token] Creating the "cluster-info" ConfigMap in the "kube-public" namespace
[kubelet-finalize] Updating "/etc/kubernetes/kubelet.conf" to point to a rotatable kubelet client certificate and key
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.100.11:6443 --token 4x919n.wofqxskn85v5skmj \
        --discovery-token-ca-cert-hash sha256:d0f8229aec07486e0f42181ef44069762b57910f1dd8d78edb9b5e64ccf82b9c 
```



然后更新生成的信息 主节点执行(**master01执行**)

```shell
  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
  
  export KUBECONFIG=/etc/kubernetes/admin.conf
```

两个从节点（worker01,worker02）执行加入操作，然后在两个从节点 worker01 和worker02上使用kubeadm 加入操作

```shell
kubeadm join 192.168.100.11:6443 --token 4x919n.wofqxskn85v5skmj \
        --discovery-token-ca-cert-hash sha256:d0f8229aec07486e0f42181ef44069762b57910f1dd8d78edb9b5e64ccf82b9c 
```

> 如果加入报错节点存在可以执行重置后重新加入 `kubeadm reset`

如果忘了可以使用下面的这行命令重新生成

```shell
 kubeadm token create --print-join-command
```

在主节点上查看从节点是否加入

```shell
kubectl  get node
```

#### 集群网络准备

> 使用calico部署集群网络
>
> 安装参考网址：https://projectcalico.docs.tigera.io/about/about-calico
>
> 看 https://docs.tigera.io/archive/v3.23/getting-started/kubernetes/requirements 介绍
>
> 我们这里k8s用的版本是1.21.0 所对应的calico 版本是v3.23

##### 第一种：基于operator安装calico

###### 下载operator资源清单文件

如果不能直接应用（网络原因 可以先找个下载下来再使用apply应用）


```shell
# 网络原因，宿主机(需要具备访问的条件)手动访问下面地址，将里面的内容放到tigera-operator.yaml中
https://projectcalico.docs.tigera.io/archive/v3.23/manifests/tigera-operator.yaml
mkdir calicodir
cd calicodir
# 应用资源清单文件，创建operator  
kubectl apply --server-side -f tigera-operator.yaml
```

> 上面文件过大  `--server-side` 目的解决 tigera-operator.yaml 过大导致创建失败的问题，停止使用 Client Side Apply（运行 kubectl apply 时的当前默认设置），而是使用 Server Side Apply，它不会将 last-applied-configuration 注释添加到对象。
>
> kubectl delete -f tigera-operator.yaml 先删除再创建也可以
>
> 参考：https://www.cnblogs.com/lzjloveit/p/17223453.html

```shell
# 网络原因，宿主机(需要具备访问的条件)手动访问下面地址，将里面的内容放到custom-resources.yaml中
https://projectcalico.docs.tigera.io/archive/v3.23/manifests/custom-resources.yaml
#打开 custom-resources.yaml文件将cidr 改为上面 kubeadm 初始化的时候设置的 --pod-network-cidr的配置信息
cidr: 192.168.0.0/16  改为      cidr: 10.244.0.0/16 

# 执行是需要保证上面tigera-operator.yaml 已经执行成功
kubectl apply -f custom-resources.yaml
```

##### 第二种:基于calico.yml安装

###### 下载calico配置文件

> 这里 也是使用3.23版本

```shell
wget  https://docs.projectcalico.org/v3.23/manifests/calico.yaml  --no-check-certificate
```

这里下载不下来就本地下载后传入服务器

###### 修改配置文件

```text
vi calico.yaml
将
# - name: CALICO_IPV4POOL_CIDR
#   value: "192.168.0.0/16"
修改为
- name: CALICO_IPV4POOL_CIDR
  value: "10.244.0.0/16"


#然后直接搜索 CLUSTER_TYPE，找到下面这段
- name: CLUSTER_TYPE
   value: "k8s,bgp"
#然后添加一个和 CLUSTER_TYPE 同级的IP_AUTODETECTION_METHOD字段，具体如下：
# value 就是指定你的网卡名字，我这里网卡是 ens33，然后直接配置的通配符 ens.*
- name: IP_AUTODETECTION_METHOD  
  value: "interface=ens.*"
```

 重点注意：**这里不能添加时候不能使用tab只能使用空格键当做空格**，不然创建的时候会报错

###### 手动加载镜像（由于网络原因）

```shell
[root@master01 calicodir]# cat calico.yaml |grep 'image:'
          image: docker.io/calico/cni:v3.23.5
          image: docker.io/calico/cni:v3.23.5
          image: docker.io/calico/node:v3.23.5
          image: docker.io/calico/node:v3.23.5
```

calico 用的是3.23.5版本

手动下载，后上传服务器解压，cd 到images目录，使用 `docker load -i  xxxx.tar` 将镜像载入到当前服务器中

https://github.com/projectcalico/calico/releases/tag/v3.23.5 

###### 修改calico 文件

>  修改镜从阿里云上海地区拉取

```shell
cat calico.yaml |grep 'image:'
# 此操作会保证当前calico配置文件使用的镜像和当前载入的镜像名一致
sed -i 's#docker.io/##g' calico.yaml
```

###### 执行创建calico.yaml创建网络

```shell
kubectl apply -f calico.yaml
```

参考：

- https://www.cnblogs.com/khtt/p/16563088.html

- https://www.lixueduan.com/posts/kubernetes/01-install/#3-%E5%AE%89%E8%A3%85-calicohttpsprojectcalicodocstigeraiogetting-startedkubernetesquickstart

##### 验证网络情况删除重装相关

###### 停止kubelet服务

```shell
systemctl stop kubelet
systemctl disable kubelet
```

###### 使用 kubeadm 重置

```
kubeadm reset
```

###### 卸载相关应用

```shell
sudo yum remove -y kubeadm kubectl kubelet kubernetes-cni kube*   
sudo yum autoremove -y
```

###### 配置清理

```shell
rm -rf /etc/systemd/system/kubelet.service
rm -rf /etc/systemd/system/kube*
```

###### 手动清理kubernetes

```shell
sudo rm -rf ~/.kube
sudo rm -rf /etc/kubernetes/
sudo rm -rf /var/lib/kube*
```

此时删除失败会有占用 可执行 `umount $(df -HT | grep '/var/lib/kubelet/pods' | awk '{print $7}')` 后再清理

##### 子节点（worker01、worker02）加入主节点(master01)

```shell
kubeadm join 192.168.100.11:6443 --token vcc8xt.lc2t495ujjjf4yr9 --discovery-token-ca-cert-hash sha256:d0f8229aec07486e0f42181ef44069762b57910f1dd8d78edb9b5e64ccf82b9c
```

注意：**子节点也需要通过docker  load -i  xxx.tar 加载calico镜像**

然后删除现有的cali-node 会自动重启

```shell
[root@master01 ~]# kubectl get pods -A
NAMESPACE     NAME                                       READY   STATUS    RESTARTS   AGE
kube-system   calico-kube-controllers-74df58766b-sxtsr   1/1     Running   2          79m
kube-system   calico-node-llkns                          1/1     Running   2          79m
kube-system   calico-node-sths7                          1/1     Running   0          8m32s
kube-system   calico-node-t2mnv                          1/1     Running   0          13s
kube-system   coredns-545d6fc579-p2f8r                   1/1     Running   2          6h57m
kube-system   coredns-545d6fc579-wl8dn                   1/1     Running   2          6h57m
kube-system   etcd-master01                              1/1     Running   6          6h58m
kube-system   kube-apiserver-master01                    1/1     Running   6          6h58m
kube-system   kube-controller-manager-master01           1/1     Running   6          6h58m
kube-system   kube-proxy-87l6q                           1/1     Running   0          28m
kube-system   kube-proxy-c5r5l                           1/1     Running   5          6h57m
kube-system   kube-proxy-vb8w9                           1/1     Running   0          29m
kube-system   kube-scheduler-master01                    1/1     Running   5          6h58m

```

删除命令

```shell
 kubectl delete pod calico-node-xxx -n kube-system
```

常用基础命令

```shell
#查看所有的节点
kubectl get nodes

#真实情况
kubectl get cs

#查看管理相关的pod运行情况(calico也在这有版本再 calico-system)
kubectl get pods -n kube-system
```

