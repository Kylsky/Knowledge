# k8s集群架构

Kubernetes通过将容器放入Pods中以在*Nodes*上运行来运行工作负载，每个节点都包含了运行pods所必须的服务，这些服务由control plane控制。

通常，一个集群中有几个节点。在学习或资源有限的环境中，可能只有一个。

节点上的组件包括 **kubelet**、一个**container runtime**以及**kube-proxy**



### 1.节点管理

有两种方式将node加到API SERVER上

```json
1.kubelet的自注册
2.用户手动添加，以下是JSON添加Node的例子：
{
  "kind": "Node",
  "apiVersion": "v1",
  "metadata": {
    "name": "10.240.79.157",
    "labels": {
      "name": "my-first-k8s-node"
    }
  }
}
```

在用户创建一个Node对象，或者kubelet自注册一个Node对象后，无论node对象是否可用，control plane都会对其进行健康检查，因此如果一个node不被使用或者已经废弃，需要及时手动删除它，以免浪费服务器资源。



### 2.自注册

当kubelet标志`--register-node`为true（默认设置）时，kubelet将尝试向API服务器注册自身。这是大多数发行版使用的首选模式。

对于自注册，kubelet使用以下选项启动：

- `--kubeconfig`  用于向API服务器进行身份验证的凭据的路径。
- `--cloud-provider`  如何与云提供商读取有关自身的元数据。
- `--register-node`  自动向API服务器注册。
- `--register-with-taints`  使用给定的列表注册节点污点（以逗号分隔`<key>=<value>:<effect>`），用来拒绝pod的调度。
- `--node-ip`  节点的IP地址。
- `--node-labels`  标签在集群中注册节点时添加。
- `--node-status-update-frequency`  指定kubelet多久将一次节点状态发布到主节点。

当node authorization mode和Node restriction admission plugin启用，kubelets仅被授权创建/修改自己的节点资源。



### 3.人工节点管理

使用kubectl命令行工具创建或修改node对象。

当您想手动创建节点对象时，设置kubelet标志--register-node=false

将节点标记为unschedulable可以防止调度器将新pods放到该节点上，但不会影响该节点上的现有pods。作为节点重新启动或其他维护之前的准备步骤，这非常有用：

```
kubectl cordon $NODENAME
```



### 4.Node状态

一个node status包含以下内容

```
Addresses
Conditions
Capacity and Allocatable
Info
```

可以通过`kubectl describe node <insert-node-name-here>`命令查看

#### 4.1 Addresses

HostName：主机名

ExternalIP：通常是外部可路由的节点的IP地址(从集群外部可用)。

InternalIP：通常是仅在集群内可路由的节点的IP地址



#### 4.2 Conditions

描述了所有运行节点的状态。

| 状态               | 说明                                                         |
| ------------------ | ------------------------------------------------------------ |
| Ready              | 如果节点运行正常并准备接受pods，则为True;如果节点运行不正常且不接受pods，则为False;如果节点控制器在最后一个节点-monitor-grace-period(缺省值为40秒)中没有听到该节点的消息，则为Unknown。 |
| DiskPressure       | 如果磁盘存在压力，即磁盘容量较低，则为True;否则false         |
| MemoryPressure     | 如果节点内存存在压力，即节点内存较低，则为True;否则false     |
| PIDPressure        | 如果进程存在压力(即节点上进程太多)，则为True;否则false       |
| NetworkUnavailable | 如果节点的网络没有正确配置，则为True，否则为False            |



### 4.3 容量与分配

描述节点上的可用资源：CPU、内存和可以调度到节点上的 Pod 的个数上限。

`capacity` 块中的字段标示节点拥有的资源总量。 

`allocatable` 块指示节点上可供普通 Pod 消耗的资源量。



### 4.4 信息

关于节点的一般性信息，例如内核版本、Kubernetes 版本（`kubelet` 和 `kube-proxy` 版本）、 Docker 版本（如果使用了）和操作系统名称。这些信息由 `kubelet` 从节点上搜集而来。

