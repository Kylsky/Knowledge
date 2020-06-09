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

### docker log -f [容器name]

查看指定容器的日志

### docker exec -it [容器id] /bin/bash

启动shell进程打开容器，按ctrl+P+Q退出