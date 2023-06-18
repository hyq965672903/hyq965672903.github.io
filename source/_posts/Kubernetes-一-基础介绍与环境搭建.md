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

#### 配置docker镜像加速

```shell
# 在/etc/docker/daemon.json添加"registry-mirrors": ["https://jjwt39jg.mirror.aliyuncs.com"]
# 下面为当前最终版本
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "registry-mirrors": ["https://jjwt39jg.mirror.aliyuncs.com"]
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

#### 验证containerd是否安装成功

```shell
containerd -v
```

### Kubernetes 1.26.2 集群部署

> k8s官方于 2020 年 12 月宣布弃用 dockershim，此后k8s从 1.2.0 到 1.2.3 版本如果使用 Docker 作为容器运行时会在 kubelet 启动时会打印一个弃用的警告日志，而最终k8s官方在 2022 年 4 月 的 Kubernetes 1.24 版本中完全移除了 dockershim（[弃用dockershim相关问题官方说明](https://link.zhihu.com/?target=https%3A//kubernetes.io/zh-cn/blog/2022/02/17/dockershim-faq/)）因此本次要搭建的 Kubernetes 1.26.2 版本将采用官方推荐的 containerd 作为容器运行时。对于 k8s+containerd 和 k8s+docker 的两种方案网上也有网友进行了性能测试对比，前者的运行速度、效率都要比后者高，且各大公有云厂商也都往 containerd 切换，因此 k8s+containerd 的组合就成了目前最合适的方案了

#### kubeadm、kubelet、kubectl安装

|          | kubeadm                | kubelet                                       | kubectl                |
| -------- | ---------------------- | --------------------------------------------- | ---------------------- |
| 版本     | 1.26.2                 | 1.26.2                                        | 1.26.2                 |
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
yum install -y kubelet-1.26.2 kubeadm-1.26.2 kubectl-1.26.2
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
kubeadm config images list --kubernetes-version=v1.26.2

#返回如下
W0615 07:46:40.126110  102911 images.go:80] could not find officially supported version of etcd for Kubernetes v1.26.2, falling back to the nearest etcd version (3.5.7-0)
registry.k8s.io/kube-apiserver:v1.26.2
registry.k8s.io/kube-controller-manager:v1.26.2
registry.k8s.io/kube-scheduler:v1.26.2
registry.k8s.io/kube-proxy:v1.26.2
registry.k8s.io/pause:3.9
registry.k8s.io/etcd:3.5.7-0
registry.k8s.io/coredns/coredns:v1.10.1
```

**脚本下载**

```shell
# vi image_download.sh

#!/bin/bash
images_list='
registry.k8s.io/kube-apiserver:v1.26.2
registry.k8s.io/kube-controller-manager:v1.26.2
registry.k8s.io/kube-scheduler:v1.26.2
registry.k8s.io/kube-proxy:v1.26.2
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
kubeadm init --kubernetes-version=v1.26.2 --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=192.168.100.11
```

##### 方式二：使用阿里云镜像

```shell
kubeadm init --kubernetes-version=v1.26.2 --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=192.168.100.11 --image-repository=registry.aliyuncs.com/google_containers
```

此时会生成从节点加入主节点的链接

```shell
ge-repository=registry.aliyuncs.com/google_containers
[init] Using Kubernetes version: v1.26.2
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
[apiclient] All control plane components are healthy after 3.501651 seconds
[upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config" in namespace kube-system with the configuration for the kubelets in the cluster
[upload-certs] Skipping phase. Please see --upload-certs
[mark-control-plane] Marking the node master01 as control-plane by adding the labels: [node-role.kubernetes.io/control-plane node.kubernetes.io/exclude-from-external-load-balancers]
[mark-control-plane] Marking the node master01 as control-plane by adding the taints [node-role.kubernetes.io/control-plane:NoSchedule]
[bootstrap-token] Using token: jr42jp.h6n7yzqo0gra5j5q
[bootstrap-token] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles
[bootstrap-token] Configured RBAC rules to allow Node Bootstrap tokens to get nodes
[bootstrap-token] Configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstrap-token] Configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstrap-token] Configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
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

kubeadm join 192.168.100.11:6443 --token jr42jp.h6n7yzqo0gra5j5q \
        --discovery-token-ca-cert-hash sha256:29d79ae863c683314a6f5f8ee4338e4bc5b78885e4e1888dbbad132737720ff2 
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
kubeadm join 192.168.100.11:6443 --token jr42jp.h6n7yzqo0gra5j5q \
        --discovery-token-ca-cert-hash sha256:29d79ae863c683314a6f5f8ee4338e4bc5b78885e4e1888dbbad132737720ff2 
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

#### 第一种：基于operator安装calico

###### 下载operator资源清单文件

如果不能直接应用（网络原因 可以先找个下载下来再使用apply应用）


```shell
# 网络原因，宿主机(需要具备访问的条件)手动访问下面地址，将里面的内容放到tigera-operator.yaml中
https://raw.githubusercontent.com/projectcalico/calico/v3.26.0/manifests/tigera-operator.yaml
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
https://raw.githubusercontent.com/projectcalico/calico/v3.26.0/manifests/custom-resources.yaml
#打开 custom-resources.yaml文件将cidr 改为上面 kubeadm 初始化的时候设置的 --pod-network-cidr的配置信息
cidr: 192.168.0.0/16  改为      cidr: 10.244.0.0/16 

# 执行是需要保证上面tigera-operator.yaml 已经执行成功
kubectl apply -f custom-resources.yaml
```

##### 第二种:基于calico.yml安装

###### 下载calico配置文件

> 这里3.25或者3.26版本都行

```shell
wget  https://docs.tigera.io/archive/v3.25/manifests/calico.yaml
```

这里下载不下来就本地下载后传入服务器

###### 修改配置文件

```text
将
# - name: CALICO_IPV4POOL_CIDR
#   value: "192.168.0.0/16"
修改为
- name: CALICO_IPV4POOL_CIDR
  value: "10.100.0.0/16"
```

###### 修改calico 文件

去掉前缀，目的是可以从镜像中下载，需要提前配置docker镜像

```shell
cat calico.yaml |grep 'image:'
sed -i 's#docker.io/##g' calico.yaml
```

###### 执行创建calico.yaml创建网络

```shell
kubectl apply -f calico.yaml
```

##### 验证网络情况

```shell
kubectl get pods --all-namespaces

#监视calico-sysem命名空间中pod运行情况
watch kubectl get pods -n calico-system

#查看kube-system命名空间中coredns状态，处于Running状态表明联网成功
kubectl get pods -n calico-system

```

#### 删除重装相关

##### 停止kubelet服务

```shell
systemctl stop kubelet
systemctl disable kubelet
```

##### 使用 kubeadm 重置

```
kubeadm reset
```

##### 卸载相关应用

```shell
sudo yum remove -y kubeadm kubectl kubelet kubernetes-cni kube*   
sudo yum autoremove -y
```

##### 配置清理

```shell
rm -rf /etc/systemd/system/kubelet.service
rm -rf /etc/systemd/system/kube*
```

##### 手动清理kubernetes

```shell
sudo rm -rf ~/.kube
sudo rm -rf /etc/kubernetes/
sudo rm -rf /var/lib/kube*
```

此时删除失败会有占用 可执行 `umount $(df -HT | grep '/var/lib/kubelet/pods' | awk '{print $7}')` 后再清理

