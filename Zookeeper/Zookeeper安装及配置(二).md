# Zookeeper安装及配置

zookeeper是一个分布式协调框架，本文主要介绍一下zookeeper在伪分布式情况下的配置，同时也适用于分布式应用

## 一、下载zookeeper

wget http://mirror.bit.edu.cn/apache/zookeeper/zookeeper-3.5.6/apache-zookeeper-3.5.6.tar.gz



## 二、解压缩

tar -zxvf apache-zookeeper-3.5.6.tar.gz



### 三、编辑配置文件

```
cd apahce-zookeeper-3.5.6
cd conf
cp zoo_sample.cfg zoo.cfg
vi zoo.cfg
在末尾加上：
server1=node01:2888:3888
server2=node02:2888:3888
server3=node03:2888:3888
集群拷贝
scp -r <需要拷贝的内容> <目标ip地址或host>:'pwd'
如：scp -r zoo.cfg node02:'pwd'
```



## 四、修改/etc/profile

```
vi /etc/profile
ZOOKEEPER_HOME=/opt/apache-zookeeper-3.5.6
PATH=$PATH:$ZOOKEEPER_HOME/bin
esc + :wq
source /etc/profile
集群拷贝：
scp -r <需要拷贝的内容> <目标ip地址或host>:'pwd'
如：scp -r /etc/profile node02:'/etc/prfile'


```



## 五、启动集群

zkServer.sh start或者zsServer.sh start-foreground



## 六、检查启动

在3台主机都启动zk后，执行

**zkServer.status**

需要注意的是,zookeeper集群启动时会进行选主，因此可以看到不同的zookeeper实例可能是follower或leader。此时zookeeper集群已经安全完成并启动了