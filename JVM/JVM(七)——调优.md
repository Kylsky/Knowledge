# JVM(七)——调优

## 一、Hotspot参数分类

#### 标准：”-“开头，所有的hotspot都支持

#### 非标准：”-X“开头，特定版本的hotspot支持特定的命令

#### 不稳定：”-XX“开头，下个版本可能取消



## 二、常见垃圾回收器组合参数设定

### -XX:+UseSerialGC

使用serial new + serial old。小型程序使用，默认情况下不jvm不会设置成该选项

### -XX:+UseParNewGC

使用parnew+serial old。这个组合已经很少使用

### -XX:+UseConc(urrent)MarkSweepGC

使用parnew + cms + serial old

### -XX:+UseParallelGC

使用parallel scavenge + parallel old

### -XX:+UseParallelOldGC

使用parallel scavenge + parallel old

### -XX:+UseG1GC

使用G1



## 三、GC日志解读

### 日志主体部分

![img](http://kylescloud.top/site/pic/GCLog.png)

### Heap Dump部分

![img](http://kylescloud.top/site/pic/HeapDump.jpg)

划红线的3条16进制数分别表示**内存起始地址、使用空间结束地址，整体空间结束地址**



## 四、调优

### 1.调优的基础概念

所谓调优，追求的主要是以下两个指标：

#### 吞吐量

用户代码执行时间/(用户代码执行时间+垃圾回收时间)。科学计算、数据挖掘等任务需要吞吐量优先，一般使用Parallel Scavenge+Parallel Old

#### 响应时间

stw(stop-the-world)越短，响应时间越好。网站、GUI、API等任务需要响应时间优先，一般使用G1



### 2.什么是调优

①根据需求进行JVM规划核预调优

②优化运行JVM运行环境

③解决JVM运行过程中出现的各种问题



### 3.调优规划

①结合业务场景

②垃圾回收器组合

③计算内存需求，选定CPU

④设定年代大小、升级年龄

⑤设定日志参数，如指定如下参入：

```
-Xloggc:/opt/xxx/logs/xxx-xxx-gc-%t.log -XX:+UseGCLogFileRotation -XX:NumberOfGCLogFiles=5 -XX:GCLogFileSize=20M -XX:+PrintGCDetails -XX:+PrintGCDateStamps -XX:+PrintGCCause
```

⑥观察日志情况



### 4.调优举例

①有一个50万PV的资料类网站（从磁盘提取文档到内存）原服务器32位，1.5G 的堆，用户反馈网站比较缓慢，因此公司决定升级，新的服务器为64位，16G 的堆内存，结果用户反馈卡顿十分严重，反而比以前效率更低了

1. 为什么原网站慢? 很多用户浏览数据，很多数据load到内存，内存不足，频繁GC，STW长，响应时间变慢
2. 为什么会更卡顿？ 内存越大，FGC时间越长
3. 咋办？ PS -> PN + CMS 或者 G1

②系统CPU经常100%，如何调优？(面试高频) CPU100%那么一定有线程在占用系统资源，

1. 找出哪个进程cpu高（top）
2. 该进程中的哪个线程cpu高（top -Hp）
3. 导出该线程的堆栈 (jstack)
4. 查找哪个方法（栈帧）消耗时间 (jstack)
5. 工作线程占比高 | 垃圾回收线程占比高

③系统内存飙高，如何查找问题？（面试高频）

1. 导出堆内存 (jmap)
2. 分析 (jhat jvisualvm mat jprofiler ... )

④如何监控JVM

1. jstat jvisualvm jprofiler arthas top...



## 五、关于TPS和QPS

### TPS

TPS：Transactions Per Second（每秒传输的事物处理个数），即服务器每秒处理的事务数。TPS包括一条消息入和一条消息出，加上一次用户数据库访问。

### QPS

QPS：全名 Queries Per Second，意思是“每秒查询率”，是一台服务器每秒能够响应的查询次数，是对一个特定的查询服务器在规定时间内所处理流量多少的衡量标准。简单的说，QPS = req/sec = 请求数/秒。它代表的是服务器的机器的性能最大吞吐能力。

### 怎么得到一个事务会消耗多少内存？

弄台机器，看能承受多少TPS？是不是达到目标？扩容或调优，让它达到理想处理能力



## 六、JVM常用参数

- -Xmn -Xms -Xmx -Xss 年轻代 最小堆 最大堆 栈空间
- -XX:+UseTLAB 使用TLAB，默认打开
- -XX:+PrintTLAB 打印TLAB的使用情况
- -XX:TLABSize 设置TLAB大小
- -XX:+DisableExplictGC System.gc()不管用 ，FGC
- -XX:+PrintGC
- -XX:+PrintGCDetails
- -XX:+PrintHeapAtGC
- -XX:+PrintGCTimeStamps
- -XX:+PrintGCApplicationConcurrentTime (低) 打印应用程序时间
- -XX:+PrintGCApplicationStoppedTime （低） 打印暂停时长
- -XX:+PrintReferenceGC （重要性低） 记录回收了多少种不同引用类型的引用
- -verbose:class 类加载详细过程
- -XX:+PrintVMOptions
- -XX:+PrintFlagsFinal -XX:+PrintFlagsInitial 必须会用
- -Xloggc:opt/log/gc.log
- -XX:MaxTenuringThreshold 升代年龄，最大值15
- 锁自旋次数 -XX:PreBlockSpin 热点代码检测参数-XX:CompileThreshold 逃逸分析 标量替换 ... 这些不建议设置