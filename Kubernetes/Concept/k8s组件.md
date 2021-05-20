# K8S组件

当你部署一个k8s应用，其实相当于部署了一个集群。k8s包含一组工作主机，称其为**nodes**，nodes用来运行容器化应用。每一个集群至少包含一个工作node。

node中包含了**pods**，pods用来作为应用载体。**control plane**则用来管理nodes和pods。生产环境中，一般control plane通常运行在多台机器上，一个集群也会有多个nodes，以此提供容错性和高可用性。

![Components of Kubernetes](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/components-of-kubernetes.svg)



## 1.Control Plane 组件

control plane组件负责集群的全局控制，检测，集群事件响应等。control plane的组件如下。

### 1.1 kube-apiserver

api server用于提供k8s的api，api server 是control plane的前端。kube-apiserver就是k8s api server的主要实现，并且支持水平拓展，因此可以运行kube-apiserver的多个实例，并在这些实例之间平衡流量。



### 1.2 etcd

一致和高可用的键值存储用作Kubernetes的所有集群数据的备份存储。

如果Kubernetes集群使用etcd作为它的备份存储，请确保有一个备份这些数据的计划。



### 1.3 kube-scheduler

用于监控新的pods并调度给node。

调度决策考虑的因素包括：个别和集体的资源需求、硬件/软件/策略约束、关联和反关联规范、数据局部性、工作负载间干扰和截止日期。



### 1.4 kube-controller-manager

运行**controller**程序的组件。controller是一个控制循环，它通过apiserver监视集群的共享状态，并进行更改，试图将当前状态移动到所需的状态。

逻辑上，每一个controller应该是一个独立的进程，但是为了减少复杂度，它们被编译成二进制文件并运行在同一个进程中。

controller包括了：

**Node Controller**：

用于控制nodes发现与下线

**Replication Controller：**

负责为系统中的每个Replication Controller维护正确的pods数量

**Endpoints Controller：**

填充端点对象，如services和pods

**Service Account & Token Controllers：**

为新的名称空间创建默认帐户和API访问令牌



### 1.5 cloud-controller-management

一个Kubernetes控制平面组件，嵌入特定于云的控制逻辑。

云控制器管理器允许将k8s集群链接到云提供商的API中，并将与云平台交互的组件与仅集群交互的组件分离开。cloud-controller-management只运行特定于云提供商的控制器。如果您在自己的场地上运行Kubernetes，或者在自己的PC内的学习环境中运行，那么集群没有cloud-controller-management。

cloud-controller-management拥有独立的controller：

**Node Controller：**

用于检查云提供商，以确定一个节点在停止响应后是否已在云中删除

**Route Controller：**

用于在底层云基础设施中设置路由

**Service Controller：**

用于创建、更新和删除云提供商负载平衡器



## 2.Node 组件

node组件运行在每一个node，用来维护运行的pods，并提供k8s运行时环境。

### 2.1 kubelet

运行在每一个node上的代理，它确保容器能在pod上运行。

kubelet采用了一组通过各种机制提供的PodSpecs，并确保这些PodSpecs中描述的容器运行正常。kubelet不会管理不通过k8s创建的容器。



### 2.2 kube-proxy

kube-proxy是运行在集群中的每个节点上的网络代理，实现了Kubernetes服务概念的一部分。

kube-proxy维护节点上的网络规则。这些网络规则允许从集群内部或外部的网络会话与Pods进行网络通信。

kube-proxy使用操作系统的数据包过滤层进行流量转发(如果有的话)，否则，kube-proxy将自己转发流量。



### 2.3 container runtime

容器运行时是负责运行容器的软件。

Kubernetes支持多种容器运行时:Docker、containerd、crio



## 3.Addons

addons使用Kubernetes资源(守护进程、部署等)来实现集群特性。因为它们提供了集群级别的特性，所以addons的命名空间资源属于kube-system命名空间。



### 3.1 DNS

所有k8s集群都需要Cluster DNS，因为很多地方需要使用到它。

Cluster DNS是一个dns 服务器，除了系统环境中的其他DNS服务器之外，它还为Kubernetes服务提供DNS记录。



### 3.2 Web UI（Dashboard）

web ui是用于Kubernetes集群的通用的、基于web的UI。它允许用户管理并排除集群中运行的应用程序以及集群本身的故障。



### 3.3 Container Resource Monitoring

在一个中央数据库中记录关于容器的通用时间序列指标，并提供一个UI用于浏览该数据。



### 3.4 Cluster-level-logging

负责将容器日志保存到具有搜索/浏览界面的中央日志存储中。