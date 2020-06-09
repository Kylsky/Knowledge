# Dockerfile

## 一、 .dockerignore

.**dockerignore**文件会通知docker哪些内容不需要被打包到img中，如

```
.git
app.log
```





## 二、 dockerfile

在项目根目录下新建文本，命名为Dockerfile，内容如下：

```
FROM jdk1.8:latest
COPY . /opt/homestay
WORKDIR /opt/homestay
EXPOSE 1040
```

上面的代码一共四行，第一行表示当前需要的image继承自jdk1.8的镜像,冒号后面的内容表示版本号。

第二行表示将当前项目路径拷贝到镜像中的指定路径。

第三行表示指定镜像的工作目录。

第四行表示暴露1040断供提供访问



## 三、创建image文件

使用以下命令：

```
docker image build -t homestay .
or
docker image build -t homestay:0.0.1 .
CMD echo '1'
```

-t参数用来指定image的名字，：后面用于指定标签，若不指定，则默认为latest。最后的点表示创建image需要用到的dockerfile文件地址。CMD表示容器创建后自动执行后面的代码。注意，指定了CMD命令以后，docker container run命令就不能附加命令了，如前面的/bin/bash

创建完成后，即可用如下命令查看:

```
docker image ls
```

