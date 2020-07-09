# Docker常用命令

### docker create [image]

创建指定镜像的容器

### docker ps -a

查看所有容器

### docker start [容器id]

启动指定容器

### docker pause [容器id]

暂停指定容器服务

### docker unpause [容器id]

使容器继续服务

### docker stop [容器id]

退出容器，但进程会进入休眠状态而不会关闭

### docker rm -f [容器id]

强制删除容器

### docker logs [容器id]

查看指定容器的日志

### docker logs -f [容器name]

查看指定容器的日志

### docker exec -it [容器id] /bin/bash

启动shell进程打开容器，按ctrl+P+Q退出



### docker run [OPTIONS] IMAGE [COMMAND] [ARG...]

#### 1.OPTIONS说明

- **-a stdin:** 指定标准输入输出内容类型，可选 STDIN/STDOUT/STDERR 三项；
- **-d:** 后台运行容器，并返回容器ID；
- **-i:** 以交互模式运行容器，通常与 -t 同时使用；
- **-P:** 随机端口映射，容器内部端口**随机**映射到主机的端口
- **-p:** 指定端口映射，格式为：**主机(宿主)端口:容器端口**
- **-t:** 为容器重新分配一个伪输入终端，通常与 -i 同时使用；
- **--name="nginx-lb":** 为容器指定一个名称；
- **--dns 8.8.8.8:** 指定容器使用的DNS服务器，默认和宿主一致；
- **--dns-search example.com:** 指定容器DNS搜索域名，默认和宿主一致；
- **-h "mars":** 指定容器的hostname；
- **-e username="ritchie":** 设置环境变量；
- **--env-file=[]:** 从指定文件读入环境变量；
- **--cpuset="0-2" or --cpuset="0,1,2":** 绑定容器到指定CPU运行；
- **-m :**设置容器使用内存最大值；
- **--net="bridge":** 指定容器的网络连接类型，支持 bridge/host/none/container: 四种类型；
- **--link=[]:** 添加链接到另一个容器；
- **--expose=[]:** 开放一个端口或一组端口；
- **--volume , -v:** 绑定一个卷



#### 2.举例

```
docker run --name mynginx -v /opt/data:/data -p 80:80 -it nginx:latest /bin/bash
```

