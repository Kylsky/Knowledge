# Zookeeper配置文件解析

以下是zookeeper安装的默认配置，更多配置可以参考文章：https://www.cnblogs.com/likui360/p/5985588.html

```
# The number of milliseconds of each tick
tickTime=2000
# The number of ticks that the initial 
# synchronization phase can take
initLimit=10
# The number of ticks that can pass between 
# sending a request and getting an acknowledgement
syncLimit=5
# the directory where the snapshot is stored.
# do not use /tmp for storage, /tmp here is just 
# example sakes.
dataDir=/var/bigdata/hadoop/zk
# the port at which the clients will connect
clientPort=2181
# the maximum number of client connections.
# increase this if you need to handle more clients
#maxClientCnxns=60
#
# Be sure to read the maintenance section of the 
# administrator guide before turning on autopurge.
#
# http://zookeeper.apache.org/doc/current/zookeeperAdmin.html#sc_maintenance
#
# The number of snapshots to retain in dataDir
#autopurge.snapRetainCount=3
# Purge task interval in hours
# Set to "0" to disable auto purge feature
#autopurge.purgeInterval=1
server.1=node02:2888:3888
server.2=node03:2888:3888
server.3=node04:2888:3888

```

### tickTime

ZK中的一个时间单元。ZK中所有时间都是以这个时间单元为基础，进行整数倍配置的。

### initLimit

集群中的follower服务器(F)与leader服务器(L)之间**初始连接时**能容忍的最多心跳数（tickTime的数量）。Follower在启动过程中，会从Leader同步所有最新数据，然后确定自己能够对外服务的起始状态。Leader允许F在 **initLimit** 时间内完成这个工作。通常情况下，我们不用太在意这个参数的设置。如果ZK集群的数据量确实很大了，F在启动的时候，从Leader上同步数据的时间也会相应变长，因此在这种情况下，有必要适当调大这个参数了。

### syncLimit

集群中flower服务器（F）跟leader（L）服务器之间的**请求和响应**最多能容忍的心跳数。  在运行过程中，Leader负责与ZK集群中所有机器进行通信，例如通过一些心跳检测机制，来检测机器的存活状态。如果L发出心跳包在syncLimit之后，还没有从F那里收到响应，那么就认为这个F已经不在线了。注意：不要把这个参数设置得过大，否则可能会掩盖一些问题。

### dataDir

该属性对应的目录。用来存放myid信息跟一些版本，日志，跟服务器唯一的ID信息等。需要注意的是不要使用默认的/tmp来存储快照的信息，会导致抛出异常

### clientPort

客户端连接的端口，默认为2181

## maxClientCnxns

单个客户端与单台服务器之间的连接数的限制，是ip级别的，默认是60，如果设置为0，那么表明不作任何限制。请注意这个限制的使用范围，仅仅是单台客户端机器与单台ZK服务器之间的连接数限制，不是针对指定客户端IP，也不是ZK集群的连接数限制，也不是单台ZK对所有客户端的连接数限制。

### autopurge.purgeInterval

3.4.0及之后版本，ZK提供了自动清理事务日志和快照文件的功能，这个参数指定了清理频率，单位是小时，需要配置一个1或更大的整数，默认是0，表示不开启自动清理功能。

### autopurge.snapRetainCount

这个参数和上面的参数搭配使用，这个参数指定了需要保留的文件数目。默认是保留3个。

### 集群信息配置

```
server.1=node02:2888:3888
server.2=node03:2888:3888
server.3=node04:2888:3888
```

配置集群信息存在一定的格式：service.N =YYY： A：B

N：代表服务器编号（也就是myid里面的值）

YYY：服务器地址

A：表示 Flower 跟 Leader的通信端口，简称服务端内部通信的端口（默认2888）

B：表示 是选举端口（默认是3888）