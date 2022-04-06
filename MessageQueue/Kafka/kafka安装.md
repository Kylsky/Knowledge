本文档使用脚本形式启动kafka，使用单机集群形式，版本为2.13-3.1.0，安装包可以通过以下链接下载。另外，本文使用zk的搭建流程不在此展开，请移步到zk相关文档。

```
https://www.apache.org/dyn/closer.cgi?path=/kafka/3.1.0/kafka_2.13-3.1.0.tgz
```



## 1.启动kafka server

```
bin/kafka-server-start.sh config/server.properties
```



## 2.修改connect配置文件

```
# 配置kafka server地址
bootstrap.servers=localhost:9092
# 配置rest端口
rest.port=8083
rest.advertised.port=8083
#配置读取插件的目录
plugin.path=/Users/kyle/MyTask/kafka/pakages/plugins
```



## 3.启动kafka-connect

```
bin/connect-distributed.sh config/connect-distributed.properties
```



## 4.部署kafka-connect-ui

这里使用docker的方式进行部署

```
docker run --rm -it -p 8000:8000 -e "CONNECT_URL=http://localhost:8083" landoop/kafka-connect-ui
```

部署后通过访问localhost:8000查看是否成功