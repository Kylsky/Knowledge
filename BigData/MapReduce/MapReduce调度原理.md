## MapReduce调度原理

### 1.mapreduce作为一个计算框架，需要数据来进行驱动，在与hdfs的搭配使用中，通过计算向数据移动实现大数据计算时，需要考虑文件资源的管理（使用hdfs的哪些块）和任务的调度，mapreduce中使用JobTracker和TaskTracker概念来处理。

![image-20201209162201269](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20201209162201269.png)



**JobTracker：**

负责资源管理和任务调度

**TaskTracker：**

负责任务的管理，如启动、监控

资源汇报，如与JobTracker的心跳汇报



### 2.客户端

#### 2.1

对于一个客户端来说，当客户端需要计算部分文件的数据时，首先会通过namenode获取文件block元数据，并以此计算出split切片清单，以此控制map数量。此时，客户端可以依据split清单分发mapreduce任务



#### 2.2

客户端依据计算任务计算程序未来运行时的配置文件



#### 2.3

客户端对计算任务的移动需要相对可靠，因此客户端会将jar包，split清单以及配置文件上传到hdfs中，副本数默认为10.因为map任务可能分散在多个datanode中，若副本数较少，会对datanode产生IO消耗



2.4

客户端调用JobTracker，通知启动一个计算程序，并告知文件位置。



### 3.JobTracker

jobstracker负责资源的管理和任务的调度，其工作流程如下

1.从hdfs中取回split清单

2.根据taskTracker返回的信息，确定每一个split对应的map任务需要去到哪一个tasktracker

3.task Tracker在与jobTracker心跳时获取分配给自己的任务信息



#### 4.TaskTracker

1.从心跳中取到任务

2.从hdfs中下载任务，包括jar包及配置文件

3.最终启动任务的MapTask/ReduceTask。‘



#### 5.存在问题

1.jobtracker单点

2.压力过大

3.集成了资源管理任务调度，两者耦合。未来新的计算框架不能复用，需要各自实现资源管理。多客户端情况下出现的多资源调度（jobtracker）无法互相感知，且由于资源不隔离，可能产生资源争抢。



#### 6.Haddop2.x解决上述问题

通过yarn架构将jobtracker中的资源管理独立出来。

![image-20201210102636197](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20201210102636197.png)

client通过resource manager获取split清单并上传至hdfs，此时resource manager获取空闲节点创建ApplicationMaster，appMaster从resource manager获取split清单，并向resource manager申请在node manager中生成container。container创建成功后会注册到Application Master。最终由app master把任务提交给container，container根据反射等操作进行任务执行。

namenode会有线程监控container的资源情况，如果资源超额，namenode会将container直接kill。



#### 7.MapReduce on yarn 流程

![image-20201210105717254](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20201210105717254.png)