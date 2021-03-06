# Zookeeper常用命令和节点信息详解

## 连接Zookeeper

```
zkCli.sh -server 127.0.0.1:2181
```

## 常用命令

### help

输出zk支持的所有命令

```
ZooKeeper -server host:port cmd args
                       stat path [watch]
                       set path data [version]
                       ls path [watch]
                       delquota [-n|-b] path
                       ls2 path [watch]
                       setAcl path acl
                       setquota -n|-b val path
                       history 
                       redo cmdno
                       printwatches on|off
                       delete path [version]
                       sync path
                       listquota path
                       rmr path
                       get path [watch]
                       create [-s] [-e] path data acl
                       addauth scheme auth
                       quit 
                       getAcl path
                       close 
                       connect host:port

```



### 1.stat path [watch]

查看节点状态

**path**：znode的路径，ZooKeeper中没有相对路径，所有路径都必须以'/'开头。

**watch**：有watch则表示监听这个节点，节点有变化会立即通知

### 2.set path data [version]

设置节点数据。

**data**：znode携带的数据。

**version**：对数据加上版本号，可以保证数据的一致性

### 3.ls path [watch]

列出path的节点

### 4.delquota [-n|-b] path

删除节点限制。

**-n**：限制子节点最大节点数

**-b**：限制数据值的最大长度

### 5.ls2 path [watch]

列出path节点的详细信息，如：

```
[zk: localhost:2181(CONNECTED) 12] ls2 /
[dubbo, zookeeper, ooxx, yarn-leader-election, hadoop-ha]
cZxid = 0x0
ctime = Thu Jan 01 08:00:00 CST 1970
mZxid = 0x0
mtime = Thu Jan 01 08:00:00 CST 1970
pZxid = 0xd00000006
cversion = 5
dataVersion = 0
aclVersion = 0
ephemeralOwner = 0x0
dataLength = 0
numChildren = 5
```

### *6.setAcl path acl

为节点授权，zookeeper支持5种权限：CREATE(c)、READ(r)、WRITE(w)、DELETE(d)、ADMIN(a)

**acl**：acl由三部分构成：1为scheme，2为user，3为permission，一般情况下表示为scheme:id:permissions。因此在授权之前，需要先使用addAuth指令为节点添加一个user

### 7.setquota -n|-b val path

设置节点权限

**-n**：子节点最大个数

**-b**：子节点最大数据长度

### 8.history

查看输入指令的历史

### 9.redo cmdno

再次执行cmdno所对应的指令

### 10.printwatches on|off

打开或关闭监听日志

### 11.delete path [version]

删除指定节点(的指定版本)的

### 12.sync path

与leader同步数据，在获取数据前，应该先执行sync,保证获取到最新数据，sync是异步的，无需等待，zookeeper能保证所有后续操作在sync完成后执行

### 13.listquota path

列出节点配额

### 14.rmr path

删除节点

### 15.get path [watch]

获取节点上的数据

### 16.create [-s] [-e] path data acl

创建一个节点

**-s**：创建的是带序列号的节点，序列号用0填充节点路径。
**-e**：创建的是临时节点。
**acl**：这个节点的ACL。

### *17.addauth scheme auth

addauth命令用于节点认证，使用方式：如addauth digest username:password

### 18.quit

推出客户端

### 19.getAcl path

获取节点的授权信息

### 20.close

close命令用于关闭与服务端的链接

### 21.connect host:port

连接到指定的zkServer



## 节点信息详解

先使用get path获取节点信息，如get /kyle

```
"aa"
cZxid = 0xd00000016
ctime = Fri Aug 16 02:10:33 CST 2019
mZxid = 0xd00000016
mtime = Fri Aug 16 02:10:33 CST 2019
pZxid = 0xd00000016
cversion = 0
dataVersion = 0
aclVersion = 0
ephemeralOwner = 0x0
dataLength = 4
numChildren = 0
```

### cZxid

zookeeper的事务是按照顺序执行的，cZid共有64个位，c表示create，末32位用来表示节点创建的事务id，前32位是用来标识leader关系是否改变

### cTime

节点创建的时间戳

### mZxid

组成与cZxid相似，末32位用来表示节点最近修改的事务id

### mTime

节点最近修改的时间戳

### pZxid

子节点最近修改的事务id

### cversion

版本号

### dataVersion

数据版本号

### aclVersion

授权版本号

### ehpemeralOwner

临时归属者。若节点属于持久节点，则位0x0，表示没有持有者。若属于临时节点则表示当前节点的持有者的sessionid