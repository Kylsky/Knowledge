# Dockerfile构建镜像

Dockerfile是一个包含用于组合镜像的命令的文本文档，Docker通过读取dockerfile中的指令按部读取自动生成镜像，执行命令：

```
docker build -t 机构/镜像名<:tags> Dockerfile路径
```

举例：

```
FROM tomcat:latest
MAINTAINER Kyle
RUN echo test && echo test2
WORKDIR /usr/local/tomcat/webapps
ADD docker-web ./docker-web
```

dockerfile编写完成后，通过开始的命令创建镜像文件

```
docker build -t god/Kyle:latest .
```

之后通过docker images即可查看创建的镜像