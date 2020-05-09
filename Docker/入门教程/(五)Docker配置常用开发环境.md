# Docker配置常用开发环境

## 一、Jdk8_Maven_Tomcat

Docker配置环境的优点在于可以拉下别的开发者已经为你继承好的一整套环境，比如jdk8+Maven+Tomcat，当然可以选择一个个地安装，这里就一起装掉了

```
#查找镜像
docker search jdk
#拉取镜像
docker pull codenvy/jdk8_maven3_tomcat8
#创建容器
docker run -di --name=jdk_maven_tomcat
```

安装完毕以后测试

```
docker exec -it jdk_maven_tomcat /bin/bash
#检查jdk版本，其他就略过了
java -version
```



## 二、MySQL

```
#拉取镜像
docker pull centos/mysql-57-centos7
#创建容器
docker run -di --name=mysql5.7 -p 3306:3306 -e MYSQL_ROOT_PASSWORD=123456 centos/mysql-57-centos7
#之后就可以远程登陆mysql了
```





