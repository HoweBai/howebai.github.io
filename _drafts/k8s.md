---
layout: post
title: kubernetes
tags: 运维
---
## 一. k8s基本概念

### 1.1 k8s架构

k8s为主从架构，主要包含master/node两部分

#### 1.1.1 master

Master 节点是 k8s 集群的控制节点，负责整个集群的管理和控制，一般有3个master用作高可用。Master 节点上主要包含以下组件：

- `kube-apiserver`：集群控制的入口，提供 http服务，为restful接口
- `kube-controller-manager`：k8s 集群中所有资源对象的自动化控制中心
- `kube-scheduler`：负责 Pod 的调度

#### 1.1.2 node

Node是 k8s 集群中的工作节点，Node 上的工作负载由 Master 节点分配，工作负载主要是运行容器应用。Node 节点上包含以下组件：

- `kubelet`：负责 Pod 的创建、启动、监控、重启、销毁等工作，同时与 Master 节点协作，实现集群管理的基本功能。
- `kube-proxy`：实现 k8s Service 的通信和负载均衡
- `容器引擎`：运行容器化(Pod)应用，一般为docker

### 1.2 其他名词

#### 1.2.1 Pod

Pod 是 k8s 最小的部署调度单元。每个 Pod 可运行一个或多个业务容器。若运行多个容器，常为一个主容器+其他辅助容器的形式，多个容器共享NET、UTS、IPC名称空间，还共享存储卷

#### 1.2.2 Label和Label Selector

Label是键值对形式的meta数据，用于区分k8s中各种对象（不同类型对象，或者同类型对象的不同个体），使用Label Selector可用于筛选满足条件的对象

#### 1.2.3 Pod控制器

用于控制pod数量的控制器，为满足不同的场景，常用有如下几种：

* `ReplicationController`  副本控制器，控制同一类pod对象的副本数量，精确符合定义的数量；还能滚动更新（更新时可临时改变副本数量）
* `ReplicaSet` 不直接使用，而是使用Deployment

* `Deployment` 只能管理无状态的应用，还支持二级控制器HPA（HorizontalPodAutoscaler），自动伸缩pod数量

* `StatefulSet`  管理有状态的应用

* `DaemonSet`  在每个node上运行一个副本，而不是随意运行

* `Job`  运行作业，有些程序执行完就可以结束了，这种不需要一直运行的，就可以把它们运行为Job；但是如果是意外中断的，就需要恢复运行

* `Cronjob`  周期运行作业

#### 1.2.4 Service

pod删除后，会通过pod控制器重新拉起来，但是新pod的ip、主机名等信息都会变化，如果以配置的方式访问pod中的服务，服务是不可用的，因此需要有服务发现的功能。k8s为每一组提供同类服务的pod和它的客户端之间，添加了一个中间层，这个中间层叫做Service。

Service只要不删除，其名称和地址都是固定的。service通过`Label Selector`识别并关联pod，并动态探测其IP，以后客户端访问pod的服务，都通过service做代理

service实际是iptables或者ipvs规则，其ip实际是不存在的，因此直接ping不通

#### 1.2.5 dns

客户端通过名称访问service，需要有dns解析，因此需要dns服务器，一般由dns pod提供（这种被称为集群的AddOns（附件））

#### 1.2.6 etcd

etcd为master存储集群信息的数据库，一般也会有多份以支持高可用，只有通过ApiServer可以操作

#### 1.2.7 namespace

不同于linux上资源隔离的namespace，k8s的namespace貌似只是对其对象分组，方便管理

### 1.3 总结

## 二. 搭建k8s集群

k8s有两种部署方式，一种直接在可各node上部署服务，但是比较麻烦，还需要生成很多对证书；另一种则是以pod方式部署，使用部署工具如`kubeadm`

这里介绍的第二种方式，`kubeadm`托管在github上[Kubernetes · GitHub](https://github.com/orgs/kubernetes/repositories?page=2)



**前提**

1. 各node能通过`/etc/hosts`文件互相解析

2. 各node有时间服务器同步时间

3. `iptables`和`firewall`不启动，因为k8s会有操作（可设置为开机不启动）

   ```shell
   systemctl status firewall.service
   systemctl stop firewall.service
   ```

   

### 2.1 安装k8s相关组件

（如果以第一种方式部署，在github上`kubernetes`仓库的`kubernetes`项目下，找到`release`（此处看到的是源码包），找到`CHANGELOG`，有对应平台的安装包，master和node节点都是安装`Server`包，`client`包是用于和server交互的）

#### 2.1.1 

第二种方式应该安装`kubeadm`，`kubelet`，`docker`等，使用国内镜像会更快安装[(阿里云开源镜像站资源目录)](https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/repodata/)。

```shell
# 配置yum源
# docker-ce
cd /etc/yum.repos.d/
wget https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo # repo文件是镜像网站上找的
# kubernetes 
# 添加repo文件内容如下
[kubernetes]
name=Kubernetes Repo
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/ # 路径也是镜像网站上的，repodata的上一层路径
gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg  # key也是网站上的，是rpm-package-key这个
enabled=1

yum repolist
yum install docker-ce kubelet kubeadm kubectl
```

#### 2.2.2

环境准备，前3步所有节点都需要执行

```shell
# 禁用防火墙
systemctl stop firewalld
systemctl disable firewalld

# 禁用SELINUX
setenforce 0
cat /etc/selinux/config # 修改 SELINUX=disabled

# 若docker的cgroupdriver为cgroups，需要修改为systemd
# 查看
docker info | grep cgroup
# 修改，在 /etc/docker/daemon.json中添加
{
	"exec-opts":[
		"native.cgroupdriver=systemd"
	]
}
# 生效
systemctl daemon-reload
systemctl restart docker

# 创建 
# 因为随后docker会操作iptables以生成一些规则，为了方便后续操作
vim /etc/sysctl.d/k8s.conf # 写入
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
# 生效
modprobe br_netfilter
sysctl -p /etc/sysctl.d/k8s.conf

```

#### 2.2.3 

外网不可访问，提前下载镜像

```shell
# 1. 把之前设置的代理去掉
# 2. 查看应该安装的镜像版本
[root@localhost ~]# kubeadm config images list
k8s.gcr.io/kube-apiserver:v1.22.4
k8s.gcr.io/kube-controller-manager:v1.22.4
k8s.gcr.io/kube-scheduler:v1.22.4
k8s.gcr.io/kube-proxy:v1.22.4
k8s.gcr.io/pause:3.5
k8s.gcr.io/etcd:3.5.0-0
k8s.gcr.io/coredns/coredns:v1.8.4
# 3. 去阿里云那儿pull镜像，把 k8s.gcr.io 替换为 registry.aliyuncs.com/google_containers
# 4. 打tag，把阿里的tag换成 k8s.gcr.io

```

或者使用代理下载

```shell
vim /usr/lib/systemd/system/docker.service
#[Service]下添加以下内容
Environment="HTTPS_PROXY=http://www.ik8s.io:10080"
Environment="NO_PROXY=127.0.0.0/8,172.20.0.0/16"

# 退出后重启服务
systemctl daemon-reload
systemctl restart docker
docker info # 可查看刚添加的变量
```



```shell
systemctl enable kubelet
systemctl enable docker

# systemctl启动服务失败，可以这样看日志
# 1. systemctl status {service}
# 2. tail /var/log/messages
```

#### 2.2.4 

接下来应该初始化kubeadm，使用`kubeadm init --help`可查看其选项

* `--apiserver-advertise-address`: 要监听的地址，默认0.0.0.0
* `--apiserver-bind-port`：要监听的端口，默认6443
* `--cert-dir`：加载证书的目录，默认 /etc/kubernetes/pki
* `--config`：自己的配置文件
* `--ignore-preflight-errors`：预检查时如果遇到错误可以忽略
* `--kubernetes-version`：k8s的版本
* `--pod-network-cidr`：pod使用的网络
* `--service-cidr`：service的网络地址

```shell
# 执行初始化命令
kubeadm init --kubernetes-version=v1.99.1 --pod-network-cidr=10.244.0.0/16 --service-cidr=10.96.0.0/12
```

若出现ERROR Swap错误，这样解决

```shell
# 错误
 [ERROR Swap]: running with swap on is not supported. Please disable swap
 
 # 解决方式
 # step1. kubelet配置文件增加参数，指明额外的初始化选项
 vim /etc/sysconfig/kubelet
 KUBELET_EXTRA_ARGS="--fail-swap-on=false"  # 如果如果swap on，不让它出错
 # step2. 执行命令添加参数  --ignore-preflight-errors=Swap
 kubeadm init --kubernetes-version=v1.99.1 --pod-network-cidr=10.244.0.0/16 --service-cidr=10.96.0.0/12 --ignore-preflight-errors=Swap
```



```shell
# 检查master服务是否健康
kubectl get cs 

# 若出现  scheduler            Unhealthy
# 可修改两个配置文件，注视掉 port=0的一行
/etc/kubernetes/manifests/kube-controller-manager.yaml
/etc/kubernetes/manifests/kube-scheduler.yaml

# 然后重启kubelet服务
```



此时执行`kubectl get nodes`，master节点还是Noteady的状态，因为还未部署网络插件，这里我们使用`flannel`插件[https://github.com/flannel-io/flannel](https://github.com/flannel-io/flannel)

找到 `Deploying flannel manually`可看到手动部署的命令

```shell
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

等待一会儿，直到`docker image ls`可查看到flannel的镜像，并且`kubectl get pods`查看不到内容，因为默认查看的是`default`名称空间，这些系统级的pod已经被挪到`kube-system`名称空间了

```shell
kubectl get pods -n kube-system
```

##### 2.2.5 

加入其他node

```shell
kubeadm join 192.168.153.131:6443 --token 4v5prq.to8l8xk7i8w86tyx --discovery-token-ca-cert-hash sha256:ee27de95a6c643fc6314a29c6f472a7fcfe821f1b4293198d091f17f5dcfc2d3
```



## 三. 快速入门

* 创建deployment，并运行指定副本数

  ```shell
  kubectl create deployment --name=nginx-deploy --image=nginx:1.21.4-alpine --port=80 --replicas=1
  ```

* 扩缩容

  ```shell
  kubectl scale --replicas=3 deployment nginx-deploy
  ```

* 给deployment创建service

  ```shell
  kubectl expose deployment nginx-deploy --name=nginx-service --port=80 --target-port=80 --protocol=TCP
  ```

  没指定类型的service默认为`ClusterIP`，只能在集群内访问。指定成`NodePort`会对外映射一个端口，则可以在集群外访问

* 更新镜像

  ```shell
  kubectl set image deploymeny nginx-deploy {container_name}={new_image}
  ```

  

* 查看更新和回滚

  ```shell
  # 查看更新
  kubectl rollout status deploymeny nginx-deploy 
  
  # 回滚
  kubectl rollout undo deploymeny nginx-deploy --to-revision={version} # 不指定版本默认到上一个版本
  ```

## 四. k8s资源

### 4.1 资源分类

k8s中，各种操作对象都是当做资源来管理的，资源实例化后就是一个对象，核心资源大概有以下类：

namespace级资源

* workload资源，工作负载类资源对象，包括pod和pod控制器（deployment、Replicaset、Statefulset、DaemonSet、Job、Cronjob）
* 服务发现与均衡类资源，包括Service和Ingress
* 配置与存储类资源，包括各种volume，CSI（容器存储接口，用以扩种第三方存储卷），块存储或云存储、分布式存储或网络存储、节点级存储、临时存储
  * 两种特殊的存储卷，ConfigMap（当做配置中心使用的资源类型）、Secret（与ConfigMap一样，用以保存敏感数据）
  * DownwardAPI 用于把外部的数据输入给容器

集群级的资源

* namespace 、Node、Role、ClusterRole、RoleBinding、ClusterRoleBinding

元数据级资源

* 用来提供元数据，如HPA、PodTemplate、LimitRange

### 4.2 配置清单创建pod

#### 4.2.1 获取pod信息

* `kubectl get pod {pod_name} -o yaml` # 以yaml格式输出pod信息

  * `apiVersion`: 该资源所属集群的api组名称和api版本，省略名称默认为core组
  
    可通过`kubectl api-versions`查看所有的类别。pod是最核心的资源，所以属于core群组；pod控制器属于apps群组中
  
    version常有alpha(内测，有些后续可能没有了)、beta(公测，接口大部分稳定)、stabel

  * `kind`: 资源类别
  
  * `metadata`: 元数
  
    * name
    * namespace
    * lables
    * annotations
  
  * `spec`: specification，该资源具有的规格，用户定义的资源的期望状态。可写
  
  * `status`: 当前资源的当前状态。只读
  
  k8s即是是资源的当前状态无限接近预期状态，以满足用户的期望
  
  

apiserver仅接受json格式的资源定义，以yaml格式提供配置清单，apiserver会自动将其转换为json。

`kubectl explain pod[.metadata]`,查看资源类型的yaml定义字段，大部分资源都可由上述5种顶级字段定义



