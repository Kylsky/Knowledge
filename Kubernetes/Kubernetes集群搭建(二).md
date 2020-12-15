# Kubernetes集群搭建(二)

## 1.构建master节点

```
kubeadm init --kubernetes-version=v1.14.1 --pod-network-cidr=10.244.0.0/16

mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

kubectl get nodes
#查看存在问题的pod
kubectl get pod --all-namespaces
#设置全局变量
#安装flannel网络组件
kubectl create -f kube-flannel.yml
#安装完flannel之后coredns还是出现了crashLoopBackOff的状态，这里找了很久原因，后来发现是防火墙导致的
iptables -P INPUT ACCEPT
iptables -P FORWARD ACCEPT
iptables -P OUTPUT ACCEPT
iptables -F

#开机启动kuberlet
systemctl enable kubelet.service
```



## 2.构建node节点

```
#在master 上执行kubeadm token list 查看 ，在node上运行
kubeadm join 192.168.10.10:6443 --skip-phases=preflight --token 0tscwe.mqxhxii12izho0o0 --discovery-token-unsafe-skip-ca-verification

kubectl get nodes
```

![image-20201211171657309](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20201211171657309.png)



## 3.启用web ui dashboard

```
kubectl apply -f kubernetes-dashboard.yaml
kubectl apply -f admin-role.yaml
kubectl apply -f kubernetes-dashboard-admin.rbac.yaml
#查看dashboard的NAME
kubectl get pods -n kube-system
#查看dashboard对应的具体内容，找到指定ip地址
kubectl describe pod <dashboard名称>
```



## 4.一些需要用到的指令

### kubectl get nodes

获取kubernetes集群当前节点

### kubectl get pod --all-namespaces

获取当前node下的所有pods

### kubeadm token list

查看token信息

### kubectl describe pod <pod1> <pod2> ... -n kube-system

查看pod的具体信息

### kubectl logs <pod> -n kube-system

查看pod日志

### kubectl create -f <filename>

创建部署

### kubectl apply -f <filename>

更新部署配置

### kubectl get pod [-o wide] 

查看已部署pod