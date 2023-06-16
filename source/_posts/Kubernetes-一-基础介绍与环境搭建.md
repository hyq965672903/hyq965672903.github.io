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
DNS1=114.114.114.114

```

改动如下 `IPADDR`自己分配 、`GATEWAY` 查看虚拟机的网络`NAT`网络配置

> `BOOTPROTO=static`   
>
> `ONBOOT=yes`
> `IPADDR=192.168.101.21`
> `NETMASK=255.255.255.0`
> `GATEWAY=192.168.101.1`
> `DNS1=192.168.101.1`
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
| 192.168.101.21 | 4C   | 6G   | 50g  | master01 |
| 192.168.101.22 | 4C   | 6G   | 50g  | worker01 |
| 192.168.101.23 | 4C   | 6G   | 50g  | worker02 |

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
192.168.101.21 master01
192.168.101.22 worker01
192.168.101.23 worker02
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

### Docker环境准备（所有节点均需要安装）

#### 获取yum 源

```shell
wget https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo -O /etc/yum.repos.d/docker-ce.repo
```

#### 安装Docker

```shell
yum -y install docker-ce
```

也可以 列出所有的docker 版本 选择指定的版本安装

```shell
# eg:

yum list docker-ce.x86_64 --showduplicates | sort -r
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

重启docker

```shell
systemctl restart docker
```

### **安装containerd** 

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



### Kubernetes 1.27.0 集群部署

#### kubeadm、kubelet、kubectl安装

|          | kubeadm                | kubelet                                       | kubectl                |
| -------- | ---------------------- | --------------------------------------------- | ---------------------- |
| 版本     | 1.27.0                 | 1.27.0                                        | 1.27.0                 |
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

```
yum install -y kubelet-1.27.0 kubeadm-1.27.0 kubectl-1.27.0
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

##### 方式一：先下载镜像

 **集群镜像准备**

```shell
kubeadm config images list --kubernetes-version=v1.27.0

#返回如下
W0615 07:46:40.126110  102911 images.go:80] could not find officially supported version of etcd for Kubernetes v1.27.0, falling back to the nearest etcd version (3.5.7-0)
registry.k8s.io/kube-apiserver:v1.27.0
registry.k8s.io/kube-controller-manager:v1.27.0
registry.k8s.io/kube-scheduler:v1.27.0
registry.k8s.io/kube-proxy:v1.27.0
registry.k8s.io/pause:3.9
registry.k8s.io/etcd:3.5.7-0
registry.k8s.io/coredns/coredns:v1.10.1
```

**脚本下载**

```shell
# vi image_download.sh

#!/bin/bash
images_list='
registry.k8s.io/kube-apiserver:v1.27.0
registry.k8s.io/kube-controller-manager:v1.27.0
registry.k8s.io/kube-scheduler:v1.27.0
registry.k8s.io/kube-proxy:v1.27.0
registry.k8s.io/pause:3.9
registry.k8s.io/etcd:3.5.7-0
registry.k8s.io/coredns/coredns:v1.10.1'

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
kubeadm init --kubernetes-version=v1.27.0 --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=192.168.101.21
```

##### 方式二：使用阿里云镜像

```shell
kubeadm init --kubernetes-version=v1.27.0 --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=192.168.101.21 --image-repository=registry.aliyuncs.com/google_containers
```

此时会生成从节点加入主节点的链接

```
kubeadm join 192.168.101.21:6443 --token 717ri3.p9e2hvb7d1qsgaj1 \
        --discovery-token-ca-cert-hash sha256:67aaf3b9aeeece3114f3b2dcd9348210d33c2f85e20f22ef348cb84e01fd6fd1 
```

然后在两个从节点 worker01 和worker02上使用kubeadm 加入操作
