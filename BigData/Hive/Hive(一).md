# Hive

## 简介

```
1.数据仓库
2.解释器、编译器、优化器
3.hive运行时，元数据存储在关系型数据库中
```



Hive的产生：非java编程者对hdfs的mapreduce操作

Hive的本质是MapReduce，但是由driver封装



oltp、olap



## 概览

![image-20201210151622525](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20201210151622525.png)

CLI

command line，命令行接口



JDBC/ODBC

数据库连接



WEBGUI

UI页面，2.2之后淘汰。HUE



ThriftServer

rpc远程调用接口



Driver

hive的核心组件。编译器解释器优化器。



Metastore

元数据存储



![image-20201210152724478](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20201210152724478.png)