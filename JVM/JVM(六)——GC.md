# JVM(六)——GC

## 一、什么是垃圾

当没有单独一个引用指向的一个对象或一堆对象被称为垃圾



## 二、如何找到垃圾

判断对象是否为垃圾通常有两种算法

### Reference Count

引用计数，在对象头上记录指向该对象的引用数量，当对象头记录的引用数量为0时，则说明需要回收。Reference Count的问题在于无法识别循环引用的对象，此时需要使用Root Searching。

### Root Searching

根搜索算法。当对象不再与GC Roots相关联时，该对象会被清理。GC Roots主要为以下内容：

1.在 虚 拟 机 栈（ 栈 帧 中 的 本 地 变 量 表） 中 引 用 的 对 象， 譬 如 各 个 线 程 被 调 用 的 方 法 堆 栈 中 使 用 到 的 参 数、 局 部 变 量、 临 时 变 量 等。

2.本 地 方 法 栈 中 JNI（ 即 通 常 所 说 的 Native 方 法） 引 用 的 对 象。

3.·在 方 法 区 中 常 量 引 用 的 对 象， 譬 如 字 符 串 常 量 池（ String Table） 里 的 引 用。

4.在 方 法 区 中 类 静 态 属 性 引 用 的 对 象， 譬 如 Java 类 的 引 用 类 型 静 态 变 量。

5.所 有 被 同 步 锁（ synchronized 关 键 字） 持 有 的 对 象。

6.Java 虚 拟 机 内 部 的 引 用， 如 基 本 数 据 类 型 对 应 的 Class 对 象， 一 些 常 驻 的 异 常 对 象（ 比 如 NullPointExcepiton、 OutOfMemoryError） 等， 还 有 系 统 类 加 载 器。

7.反 映 Java 虚 拟 机 内 部 情 况 的 JM Bean、 JVMTI 中 注 册 的 回 调、 本 地 代 码 缓 存 等。



## 三、GC 算法

### Mark-Sweep(标记清除)

标记可以回收的对象并回收。算法相对简单，**在存活对象比较多的情况下效率较高**。需要经过两次扫描，所以**效率较之其他算法偏低，且容易产生碎片**



### Copying(拷贝)

将堆内存一分为二，第一部分用于存储对象，当第一部分需要GC时，jvm将不需要GC的对象复制到第二部分中，然后清除第一部分的内存信息。适用于存活对象较少的情况，只扫描一次，**效率提高**了，且**没有碎片**。但是这**造成了空间的浪费**，并且**复制移动对象需要调整对象引用**。



### Mark-Compact(标记压缩)

经过一次扫描确认不可回收的对象，随后将不可回收的对象整理成连续的排列，并清除需要GC的对象。mark compact **不会产生碎片**，但是需要**扫描两遍**内存，并且**需要复制移动**，因此**效率偏低**



## 四、堆内存逻辑分区

### 部分垃圾回收器使用模型

除Epsilon ZGC Shenandoah之外的GC都是使用逻辑分代模型，G1是逻辑分代，物理不分代。除此之外不仅逻辑分代，而且物理分代。



### 分区

![heapPartition](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/heapPartition.jpg)

堆内存一般被分为**新生代**和**老年代**。

**新生代**分为eden区和survivor区，新生代的存活对象较少，死去对象较多，因此相对使用**copying算法**效率较高，新生代中存活的对象的复制年龄超过限制时(**超过jvm参数-XX:MaxTenuringThreshold指定次数**，默认15)，就会进入老年代。



### GC概念

![img](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/GCConcept.jpg)

MinorGC/YGC:年轻代空间耗尽时触发

MajorGC/FullGC：在老年代无法继续分配空间时触发，新生代老年代同时进行回收



### 对象内存分配问题

对象在创建之初会尽量往栈上存储，而不是eden区，这里介绍一下细节

#### 1.栈上分配

**①线程私有小对象**

容量小

**②无逃逸**

只在当前代码或上下文中有意义。逃逸分析的JVM参数为-XX:DoEscapeAnalysis

**③支持标量替换**

使用普通类型作为标量来代替整个对象。标量替换的JVM参数为-XX:EliminateAllocations

#### 2.线程本地分配(TLAB,Thread Local Allocation Buffer)

若无法在栈上分配，则分配到eden区。由于线程会抢占资源导致效率下降，因此在eden区中为每一个线程分配默认为eden区容量1%的大小。TLAB的JVM参数为-XX:UseTLAB

**①占用eden，默认1%**

**②多线程的时候不用竞争eden就可以申请空间，提高效率**

**③小对象**

#### 3.老年代

大对象



## 五、常见的垃圾回收器

![GarbageCollectors](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/GarbageCollectors.jpg)

### Serial

**新生代**收集器，采用复制算法。a stop-the-world(stw),copying collector which uses a single GC thread。工作线程在一个安全点上(safe point)停止工作，serial开始GC，现在GC使用极少。

### Parallel Scavenge

**新生代**收集器，采用复制算法。a stop-the-world(stw),copying collector which uses multiple GC thread。多线程清理垃圾

### ParNew(Parallel New)

**新生代**收集器，采用复制算法a stop-the-world(stw),copying collector which uses multiple GC thread。Parallel Scavenge的新版本，用来配合CMS使用，默认线程数为cpu的核数。

### Serial Old

**老年代**收集器，采用标记整理算法。a stop-the-world(stw),mark-sweep-compact collector that uses a single GC thread。与serial相似，只是gc算法不一样。现在基本不用

### Parallel Old

**老年代**收集器，采用标记整理算法。a stop-the-world(stw),compacting collector which uses multiple GC thread。与Parallel相似

### CMS

**老年代**收集器，采用标记清除算法1.4之后诞生。Concurrent Mark Sweep,CMS是里程碑式的GC。现代服务器的内存越来越大，因此回收线程的工作时长会很大，因此stw无法被忍受。CMS会在下面详细介绍。

### G1

在jdk1.7中使用，这里不详细介绍

#### ZGC

在jdk11中使用，这里不详细介绍



### 常用组合

#### 1.Serial+Serial Old

#### 2.ParNew+CMS

#### 3.Parallel Scavenge+Parallel Old



### 垃圾回收器的选择

#### Serial——几十兆、上百兆的内存大小

#### Parallel Scavenge——几个Gb的内存大小

#### CMS——20Gb及以上

#### G1——上百Gb及以上

#### ZGC——4T~16T(JDK13)



## 六、CMS

### 从线程角度分析工作阶段

![CMS1](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/CMS1.jpg)

#### 初始标记

stop-the-world，标记GC Roots

#### 并发标记

占用了CMS80%的GC时间，在这段时间与应用程序同时运行，应用程序产生垃圾的同时，CMS标记新的垃圾

#### 重新标记

stop-the-world，标记并发标记间由于一些情况产生的未被标记的新垃圾

#### 并发清理

开始清理标记阶段清理出的垃圾，并发清理阶段产生的新的垃圾称为浮动垃圾，等待下一阶段CMS的GC



### CMS的缺点

#### 1.Memory Fragmentation

内存碎片化，CMS使用的是Mark Sweep算法，会导致碎片化，因此当老年代中的内存趋于碎片化时，新生代的对象可能无法存储到老年代中，该情况下会导致CMS转变为Serial Old来进行GC，单线程的GC会导致效率急剧下降

#### 2.Floating Garbage

在并发清理阶段产生的浮动垃圾需要等待下一次的GC才能被清理，但是这部分延迟未清理的空间可能导致新生代的对象无法被正常放入老年代，与此同时会在日志中记录下Concurrent Mode Failure，并切换到Serial Old。可以通过降低-XX:CMSInitiatingOccupancyFraction (92%)这个值，让老年代保持足够的空间。