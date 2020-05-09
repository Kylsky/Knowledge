# Docker基本概念

Docker是提供应用打包，部署与应用的容器化平台

![image-20200509111314998](C:\Users\Kyle\AppData\Roaming\Typora\typora-user-images\image-20200509111314998.png)

docker引擎包含三个部分：

1.docker daemon服务层

2.rest api通信层

3.client客户端层



## 镜像image

只读文件，提供了运行程序完整的软硬件资源，可以说是应用程序的集装箱。



## 容器container

是镜像的实例，容器之间是彼此隔离的



## 注册中心Registry

远程仓库