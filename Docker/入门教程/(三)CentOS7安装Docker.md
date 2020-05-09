# CentOS7安装Docker

Docker是个啥前面已经讲过了，接下来直接开始安装吧

## 一、查看内核版本

```
uname -a
```



## 二、安装依赖

```
yum install -y yum-utils device-mapper-persistent-data lvm2
```

如果提示安装失败，可以更新yum源



## 三、设置docker yum源

```
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
```



## 四、查看docker版本并安装

```
yum list docker-ce --showduplicates | sort -r
```

上面的命令会显示很多版本，找到合适内核的版本安装即可，比如：

```
yum install docker-ce-17.12.1.ce
```



## 五、启动Docker

```
systemctl start docker
#设置开机启动
systemctl enable docker
```



## 六、验证安装

```
docker version 
```

有client和service两部分表示docker安装启动都成功了



## 七、总结

到这里Docker安装就结束了，可以快乐地使用容器化技术来迅速部署和开发了~