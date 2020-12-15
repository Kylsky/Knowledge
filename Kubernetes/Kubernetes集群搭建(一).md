# Kubernetes集群搭建(一)

## 1.前期准备

资料下载地址（自个儿翻译下）：

```
链接：https://盘.百度.com/s/1liDttY0WZbItt7-fNjMivw    提取码：xdji
```



3台虚拟机，内存2g，一主二备，ip分别如下

```
192.168.10.10 master
192.168.10.11 node1
192.168.10.12 node2
```



在每台服务器运行如下代码

```
#设置时区
timedatectl set-timezone Asia/Shanghai  
#各自设置每个服务器的hostname
#10设置为master
hostnamectl set-hostname master   
#11设置为node1
hostnamectl set-hostname node1
#12设置为node2
hostnamectl set-hostname node2

#修改host文件
vim /etc/hosts
192.168.10.10 master
192.168.10.11 node1
192.168.10.12 node2

#关闭防火墙，正式环境可以跳过这一步
sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config
setenforce 0
systemctl disable firewalld
systemctl stop firewalld
```



## 2.安装docker

在每个节点安装docker，命令如下：

```
cd kubernetes-1.14/
tar -zxvf docker-ce-18.09.tar.gz
cd docker 
yum localinstall -y *.rpm
systemctl start docker
systemctl enable docker
```



## 3.确保从cgroups均在同一个从groupfs

#cgroups是control groups的简称，它为Linux内核提供了一种任务聚集和划分的机制，通过一组参数集合将一些任务组织成一个或多个子系统。   
#cgroups是实现IaaS虚拟化(kvm、lxc等)，PaaS容器沙箱(Docker等)的资源管理控制部分的底层基础。
#子系统是根据cgroup对任务的划分功能将任务按照一种指定的属性划分成的一个组，主要用来实现资源的控制。
#在cgroup中，划分成的任务组以层次结构的形式组织，多个子系统形成一个数据结构中类似多根树的结构。cgroup包含了多个孤立的子系统，每一个子系统代表单一的资源

```
docker info | grep cgroup 

#如果不是groupfs,执行下列语句

cat << EOF > /etc/docker/daemon.json
{
  "exec-opts": ["native.cgroupdriver=cgroupfs"]
}
EOF
systemctl daemon-reload && systemctl restart docker
```



## 4.安装kubeadmin

```
cd /usr/local/k8s-install/kubernetes-1.14
tar -zxvf kube114-rpm.tar.gz
cd kube114-rpm
yum localinstall -y *.rpm
```



## 5.关闭swap分区

```
swapoff -a
vi /etc/fstab
#将以下内容注释掉
/dev/mapper/centos-swap swap                    swap    defaults        0 0
```



## 6.配置网桥

```
#提高网络传输安全性
cat <<EOF >  /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl --system
```



## 7.通过镜像安装k8s

```
cd /usr/local/k8s-install/kubernetes-1.14
docker load -i k8s-114-images.tar.gz
docker load -i flannel-dashboard.tar.gz
```

