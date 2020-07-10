# Docker镜像部署阿里云

## 1.编写dockerfile

```
FROM ascdc/jdk8
RUN echo test jdk8
MAINTAINER Kyle
WORKDIR /usr/local/webapps
ADD docker-web ./docker-web
```



## 2.创建镜像

编写完成后，在当前目录构建imge

```
docker build -t kylsky/jdk8:latest .
```



### 3.在阿里云创建镜像仓库

https://cr.console.aliyun.com/cn-hangzhou/namespaces

创建仓库完成后，根据仓库中的阿里云指南即可完成镜像的远程部署



### 4.其他

docker build之后发现镜像文件居然有500M之大，感觉不太利于之后的项目开发，这里贴一个最小化构建的教程https://blog.csdn.net/xyb0926/article/details/103474964

