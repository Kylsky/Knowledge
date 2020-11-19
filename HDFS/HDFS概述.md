# HDFS

### 1.核心

1.分治思想

2.并行计算

*3.计算向数据移动

4.数据本地化读取



### 2.Hadoop相关项目

#### 2.1 MapReduce

分布式计算

#### 2.2 YARN

分布式资源管理

#### 2.3 Common

公共工具类

#### 2.4 Hadoop Distributed File System（HDFS）

分布式存储

#### 2.5 批计算—>流计算

Spark属于伪实时计算

Flink属于实时计算

### 3.大数据生态

![image-20200731165422903](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20200731165422903.png)



### 4.理论知识点

#### 4.1 存储模型

- 文件线性按字节切割成块（block），具有offset，id属性
- 文件的block大小不需要是特定的
- 一个文件除了最后一个block，其他block大小是一致的
- block的大小依据硬件的I/O特性调整
- block被分散存放在集群的节点中，具有location属性
- block具有副本，没有主从概念，副本不能出现在同一个节点
- 副本是满足可靠性和性能的关键
- 文件上传可以指定block大小和副本数，上传后只能修改副本数
- 一次写入多次读取，不支持修改
- 支持追加数据



#### 4.2 架构设计

- HDFS是一个主从架构

- 由一个NameNode和一些DataNode组成

- 面向文件包含：文件数据和文件元数据

- NameNode负责存储和管理文件元数据，并维护一个层次型的文件目录树

- DataBide负责存储文件数据（block块），并提供block的读写

- DataNode与NameNode维持心跳，并汇报自己持有的block信息

- Client和NameNode交互文件元数据，和DataNode交互文件block数据

  ![img](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/04a1596e-3866-4436-8ad4-b4f233db393f-2429517.jpg)



#### 6.3 角色功能

##### 6.3.1 NameNode

- 完全基于内存存储文件元数据、目录结构、文件bolck的映射
- 需要持久化方案保证数据可靠性
- 提供副本放置策略



##### 6.3.2 DataNode

- 基于本地磁盘存储文件block（文件的形式）
- 保存block的校验和数据，保证block的可靠性
- 与NameNode保持心跳，汇报blcok列表状态



#### 6.4 元数据持久化

- 任何对文件系统元数据产生修改的操作，NameNode都会使用一种称为EditLog的事物日志记录下来
- 使用FsImage存储内存所有的元数据状态
- 使用本地磁盘保存EditLog和FsImage
- EditLog具有完整性，数据丢失少，但恢复速度慢，并有体积膨胀风险
- FsImage具有恢复速度快，体积与内存数据相当，但不能实时保存，容易产生数据丢失
- NameNode使用了FsImage+EditLog整合的方案
- 滚动将增量的EditLog更新到FsImage，以保证更近时点的FsImage和更小的EditLog体积



#### 6.5 安全策略

1.HDFS搭建时会格式化分布式文件系统，格式化操作会产生一个空的FsImage

2.当NameNode启动时，从硬盘中读取EditLog和FsImage

3.将所有EditLog中的事务作用在内存中的FsImage上

4.将新版本的FsImage从内存中保存到本地磁盘上

5.删除旧的EditLog，因为旧的EditLog的事务都已经作用在FsImage上了

6.NameNode启动后会进入一个成为安全模式的特殊状态

7.处于安全模式的NameNode是不会进行数据块的复制的

8.NameNode从所有的DataNode接收心跳信号和块状态报告

9.每当NameNode检测确认某个数据块的副本数目达到这个最小值，那么该数据块就会被认为是副本安全的（safely replicated）

10.在一定百分比的数据块被NameNode检测确认是安全之后（加上一个额外的30秒等待时间），NameNode将退出安全模式

11.接下来NameNode会确定还有哪些数据块的副本没有达到指定数目，并将这些数据块复制到其他DataNode上

12.HDFS中的SecondaryNameNode（SNN）

```
在非HA（高可用）模式下，SNN一般是独立的节点，周期完成对NN的EditLog向FsImage合并，减少EditLog大小，减少NN启动时间

根据配置文件设置的时间间隔fs.checkpoint.period默认是3600秒

根据配置文件设置EditLog大小fs.checkpoint.size规定edits文件的最大值默认是64MB
```



<img src="http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/dac38b28-7cd6-4cc1-b445-2c204c2caab6-2429517.jpg" alt="img" style="zoom: 67%;" />



#### 6.6 副本放置策略

第一个副本：放置在上传文件的DataNode；如果是集群外提交，则随机挑选一台磁盘不太慢，CPU不太忙的节点

第二个副本：放置在与第一个副本不同的机架的节点上

第三个副本：与第二个副本相同机架的节点上

更多副本：随机节点



#### 6.7 读写流程

- HDFS写流程

  ![img](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/fdb06867-bc59-4072-bb55-62258e29322a-2429517.jpg)

  - Client和NameNode连接创建文件元数据
  - NameNode判定源数据是否有效
  - NameNode触发副本放置策略，返回一个有序的DataNode列表
  - Client和DataNode建立pipeline连接（流式传输）
  - Client将块切分成packet（64kb),并使用chunk（512b）+checksum(校验和，4b)填充
  - Client将packet放入发送队列dataqueue中，并向第一个DataNode发送
  - 第一个DataNode收到packet后本地保存并发送第二个DataNode
  - 第二个DataNode收到packet后本地保存并发送第三个DataNode
  - 这一个过程中，上游节点同时发送下一个packet
  - 生活中类比工厂的流水线：结论：流式其实也是变种的并行计算
  - HDFS使用这种传输方式，副本数对于client是透明的
  - 当block传输完成，DataNode各自向NameNode汇报，同时Client继续传输下一个block
  - 所以，client的传输和block的汇报也是并行的



- HDFS读流程

  ![img](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/ba751708-6f08-4cf5-be7b-ec3853e83e01-2429517.jpg)

  - 为了降低整体的带宽消耗和读取延时，HDFS会尽量让读取程序读取离它最近的副本
  - 如果在读取程序的同一个机架上，那么客户端也将首先读取本地数据中心的副本
  - 语义：下载一个文件：
    - Clienthe NameNode交互文件元数据获取fileBlockLocation
    - NameNode尝试下载block并校验数据完整性
  - 语义：下载一个文件其实是获取文件的所有block元数据，那么子集获取某些block应该成立
    - HDFS支持client给出文件的offset（偏移量）自定义连接哪些block的DataNode，自定义获取数据
      - 这个是支持计算分层的分治、并行计算的核心，也是为什么分布式文件系统这么多，还是要开发出HDFS