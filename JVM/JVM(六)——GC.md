# JVM(六)——GC

## 1.什么是垃圾

当没有单独一个引用指向的一个对象或一堆对象被称为垃圾



## 2.如何找到垃圾

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



## 3.GC 算法

### Mark-Sweep(标记清除)

标记可以回收的对象并回收。算法相对简单，**在存活对象比较多的情况下效率较高**。需要经过两次扫描，所以**效率较之其他算法偏低，且容易产生碎片**



### Copying(拷贝)

将堆内存一分为二，第一部分用于存储对象，当第一部分需要GC时，jvm将不需要GC的对象复制到第二部分中，然后清除第一部分的内存信息。适用于存活对象较少的情况，只扫描一次，**效率提高**了，且**没有碎片**。但是这**造成了空间的浪费**，并且**复制移动对象需要调整对象引用**。



### Mark-Compact(标记压缩)

经过一次扫描确认不可回收的对象，随后将不可回收的对象整理成连续的排列，并清除需要GC的对象。mark compact **不会产生碎片**，但是需要**扫描两遍**内存，并且**需要复制移动**，因此**效率偏低**



## 4.堆内存逻辑分区

### 部分垃圾回收器使用模型

除Epsilon ZGC Shenandoah之外的GC都是使用逻辑分代模型，G1是逻辑分代，物理不分代。除此之外不仅逻辑分代，而且物理分代。



### 分区

![heapPartition](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/heapPartition.jpg)

堆内存一般被分为**新生代**和**老年代**。

**新生代**分为eden区和survivor区，新生代的存活对象较少，死去对象较多，因此相对使用**copying算法**效率较高，新生代中存活的对象的复制年龄超过限制时(**超过jvm参数-XX:MaxTenuringThreshold指定次数**，默认15)，就会进入老年代。



### GC概念

![img](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/GCConcept.jpg)

MinorGC/YGC:年轻代空间耗尽时触发

MajorGC/FullGC：在老年代无法继续分配空间时触发，新生代老年代同时进行回收。(MajorGC在其他文章处也有说是只清除老年代的)



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



## 5.常见的垃圾回收器

jdk6-jdk14的垃圾收集器组合：

https://cloud.tencent.com/developer/article/1620318



![GarbageCollectors](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/GarbageCollectors.jpg)

![image-20201023142159372](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20201023142159372.png)

上 图 展 示 了 七 种 作 用 于 不 同 分 代 的 收 集 器， 如 果 两 个 收 集 器 之 间 存 在 连 线， 就 说 明 它 们 可 以 搭 配 使 用 [3] ，图 中 收 集 器 所 处 的 区 域， 则 表 示 它 是 属 于 新 生 代 收 集 器 抑 或 是 老 年 代 收 集 器。

### Serial

![image-20201023142353143](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20201023142353143.png)

**新生代**收集器，采用复制算法。a stop-the-world(stw),copying collector which uses a single GC thread。工作线程在一个安全点上(safe point)停止工作，serial开始GC，现在GC使用极少。



### Parallel Scavenge

**新生代**收集器，采用复制算法。a stop-the-world(stw),copying collector which uses multiple GC thread。多线程清理垃圾。Parallel Scavenge和ParNew两者都是复制算法，都是并行处理，但是不同的是，paralel scavenge 可以设置最大gc停顿时间（-XX:MaxGCPauseMills）以及gc时间占比(-XX:GCTimeRatio)。

Parallel Scavenge 收 集 器 的 特 点 是 它 的 关 注 点 与 其 他 收 集 器 不 同， CMS 等 收 集 器 的 关 注 点 是 尽 可 能 地 缩 短 垃 圾 收 集 时 用 户 线 程 的 停 顿 时 间， 而 Parallel Scavenge 收 集 器 的 目 标 则 是 达 到 一 个 可 控 制 的 吞 吐 量（ Throughput）。 所 谓 吞 吐 量 就 是 处 理 器 用 于 运 行 用 户 代 码 的 时 间 与 处 理 器 总 消 耗 时 间 的 比 值， 即：

<img src="http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20201023151310053.png" alt="image-20201023151310053" style="zoom:67%;" />





### ParNew(Parallel New)

<img src="http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20201023143108124.png" alt="image-20201023143108124" style="zoom:67%;" />

**新生代**收集器，实际上是Serial的并行版本。采用复制算法a stop-the-world(stw),copying collector which uses multiple GC thread。Parallel Scavenge的新版本，用来配合CMS使用，默认线程数为cpu的核数。

ParNew 收 集 器 除 了 支 持 多 线 程 并 行 收 集 之 外， 其 他 与 Serial 收 集 器 相 比 并 没 有 太 多 创 新 之 处， 但 它 却 是 不 少 运 行 在 服 务 端 模 式 下 的 HotSpot 虚 拟 机， 尤 其 是 JDK 7 之 前 的 遗 留 系 统 中 首 选 的 新 生 代 收 集 器， 其 中 有 一 个 与 功 能、 性 能 无 关 但 其 实 很 重 要 的 原 因 是： 除 了 Serial 收 集 器 外， **目 前 只 有** 它 能 与 CMS 收 集 器 配 合 工 作。

ParNew 收 集 器 是 激 活 CMS 后（ 使 用-XX： + UseConcMarkSweepGC 选 项） 的 默 认 新 生 代 收 集 器， 也 可 以 使 用-XX： +/-UseParNewGC 选 项 来 强 制 指 定 或 者 禁 用 它。



### Serial Old

**老年代**收集器，采用标记整理算法。a stop-the-world(stw),mark-sweep-compact collector that uses a single GC thread。

Serial Old 是 Serial 收 集 器 的 老 年 代 版 本， 它 同 样 是 一 个 单 线 程 收 集 器， 使 用 标 记-整 理 算 法。 这 个 收 集 器 的 主 要 意 义 也 是 供 客 户 端 模 式 下 的 HotSpot 虚 拟 机 使 用。 如 果 在 服 务 端 模 式 下， 它 也 可 能 有 两 种 用 途： 一 种 是 在 JDK 5 以 及 之 前 的 版 本 中 与 Parallel Scavenge 收 集 器 搭 配 使 用 [1] ，另 外 一 种 就 是 作 为 CMS 收 集 器 发 生 失 败 时 的 后 备 预 案， 在 并 发 收 集 发 生 Concurrent Mode Failure 时 使 用。

<img src="http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20201023152733624.png" alt="image-20201023152733624" style="zoom:67%;" />



### Parallel Old

**老年代**收集器，采用标记整理算法。a stop-the-world(stw),compacting collector which uses multiple GC thread。与Parallel相似

<img src="http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20201023152850422.png" alt="image-20201023152850422" style="zoom:67%;" />



### CMS

**老年代**收集器，Concurrent Mark Sweep,CMS是里程碑式的GC。采用标记清除算法，1.4之后诞生，这 款 收 集 器 是 HotSpot 虚 拟 机 中 第 一 款 真 正 意 义 上 支 持 并 发 的 垃 圾 收 集 器， 它 首 次 实 现 了 让 垃 圾 收 集 线 程 与 用 户 线 程（ 基 本 上） 同 时 工 作。现代服务器的内存越来越大，因此回收线程的工作时长会很大，因此stw无法被忍受。CMS会在下面详细介绍。

![image-20201023155615681](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20201023155615681.png)



### G1

在jdk1.7中使用，这里不详细介绍



### ZGC

在jdk11中使用，这里不详细介绍



### G1

<img src="http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20201026094412579.png" alt="image-20201026094412579" style="zoom:67%;" />

![image-20201026094707395](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20201026094707395.png)

Garbage First，12年发布。G1 开 创 的 基 于 Region 的 堆 内 存 布 局 是 它 能 够 实 现 这 个 目 标 的 关 键。 虽 然 G1 也 仍 是 遵 循 分 代 收 集 理 论 设 计 的， 但 其 堆 内 存 的 布 局 与 其 他 收 集 器 有 非 常 明 显 的 差 异： G1 不 再 坚 持 固 定 大 小 以 及 固 定 数 量 的 分 代 区 域 划 分， 而 是 把 连 续 的 Java 堆 划 分 为 多 个 大 小 相 等 的 独 立 区 域（ Region）， 每 一 个 Region 都 可 以 根 据 需 要， 扮 演 新 生 代 的 Eden 空 间、 Survivor 空 间， 或 者 老 年 代 空 间。 收 集 器 能 够 对 扮 演 不 同 角 色 的 Region 采 用 不 同 的 策 略 去 处 理， 这 样 无 论 是 新 创 建 的 对 象 还 是 已 经 存 活 了 一 段 时 间、 熬 过 多 次 收 集 的 旧 对 象 都 能 获 取 很 好 的 收 集 效 果。

Region 中 还 有 一 类 特 殊 的 Humongous 区 域， 专 门 用 来 存 储 大 对 象。 G1 认 为 只 要 大 小 超 过 了 一 个 Region 容 量 一 半 的 对 象 即 可 判 定 为 大 对 象。 每 个 Region 的 大 小 可 以 通 过 参 数-XX：G1HeapRegionSize 设 定， 取 值 范 围 为 1MB ～ 32MB， 且 应 为 2 的 N 次 幂。 而 对 于 那 些 超 过 了 整 个 Region 容 量 的 超 级 大 对 象， 将 会 被 存 放 在 N 个 连 续 的 HumongousRegion 之 中， G1 的 大 多 数 行 为 都 把 Humongous Region 作 为 老 年 代 的 一 部 分 来 进 行 看 待。

G1作为垃圾处理器主要面对以下问题：

1.使 用 记 忆 集 避 免 全 堆 作 为 GC Roots 扫 描， 但 在 G1 收 集 器 上 记 忆 集 的 应 用 其 实 要 复 杂 很 多， 它 的 每 个 Region 都 维 护 有 自 己 的 记 忆 集， 这 些 记 忆 集 会 记 录 下 别 的 Region 指 向 自 己 的 指 针， 并 标 记 这 些 指 针 分 别 在 哪 些 卡 页 的 范 围 之 内。 G1 的 记 忆 集 在 存 储 结 构 的 本 质 上 是 一 种 哈 希 表， Key 是 别 的 Region 的 起 始 地 址， Value 是 一 个 集 合， 里 面 存 储 的 元 素 是 卡 表 的 索 引 号。 这 种“ 双 向” 的 卡 表 结 构（ 卡 表 是“ 我 指 向 谁”， 这 种 结 构 还 记 录 了“ 谁 指 向 我”） 比 原 来 的 卡 表 实 现 起 来 更 复 杂， 同 时 由 于 Region 数 量 比 传 统 收 集 器 的 分 代 数 量 明 显 要 多 得 多， 因 此 G1 收 集 器 要 比 其 他 的 传 统 垃 圾 收 集 器 有 着 更 高 的 内 存 占 用 负 担。 根 据 经 验， G1 至 少 要 耗 费 大 约 相 当 于 Java 堆 容 量 10% 至 20% 的 额 外 内 存 来 维 持 收 集 器 工 作。

2.在 并 发 标 记 阶 段 如 何 保 证 收 集 线 程 与 用 户 线 程 互 不 干 扰 地 运 行？ 这 里 首 先 要 解 决 的 是 用 户 线 程 改 变 对 象 引 用 关 系 时， 必 须 保 证 其 不 能 打 破 原 本 的 对 象 图 结 构， 导 致 标 记 结 果 出 现 错 误，G1 收 集 器 通 过 原 始 快 照（ SATB） 算 法 来 解 决 这 个 问 题。 此 外， 垃 圾 收 集 对 用 户 线 程 的 影 响 还 体 现 在 回 收 过 程 中 新 创 建 对 象 的 内 存 分 配 上， 程 序 要 继 续 运 行 就 肯 定 会 持 续 有 新 对 象 被 创 建， G1 为 每 一 个 Region 设 计 了 两 个 名 为 TAMS（ Top at Mark Start） 的 指 针， 把 Region 中 的 一 部 分 空 间 划 分 出 来 用 于 并 发 回 收 过 程 中 的 新 对 象 分 配， 并 发 回 收 时 新 分 配 的 对 象 地 址 都 必 须 要 在 这 两 个 指 针 位 置 以 上。 G1 收 集 器 默 认 在 这 个 地 址 以 上 的 对 象 是 被 隐 式 标 记 过 的， 即 默 认 它 们 是 存 活 的， 不 纳 入 回 收 范 围。 与 CMS 中 的“ Concurrent Mode Failure” 失 败 会 导 致 Full GC 类 似， 如 果 内 存 回 收 的 速 度 赶 不 上 内 存 分 配 的 速 度， G1 收 集 器 也 要 被 迫 冻 结 用 户 线 程 执 行， 导 致 Full GC 而 产 生 长 时 间“ Stop The World”。

3.怎 样 建 立 起 可 靠 的 停 顿 预 测 模 型？ 用 户 通 过-XX：MaxGCPauseMillis 参 数 指 定 的 停 顿 时 间 只 意 味 着 垃 圾 收 集 发 生 之 前 的 期 望 值， 但 G1 收 集 器 要 怎 么 做 才 能 满 足 用 户 的 期 望 呢？ G1 收 集 器 的 停 顿 预 测 模 型 是 以 衰 减 均 值（ Decaying Average） 为 理 论 基 础 来 实 现 的， 在 垃 圾 收 集 过 程 中， G1 收 集 器 会 记 录 每 个 Region 的 回 收 耗 时、 每 个 Region 记 忆 集 里 的 脏 卡 数 量 等 各 个 可 测 量 的 步 骤 花 费 的 成 本， 并 分 析 得 出 平 均 值、 标 准 偏 差、 置 信 度 等 统 计 信 息。这 里 强 调 的“ 衰 减 平 均 值” 是 指 它 会 比 普 通 的 平 均 值 更 容 易 受 到 新 数 据 的 影 响， 平 均 值 代 表 整 体 平 均 状 态， 但 衰 减 平 均 值 更 准 确 地 代 表“ 最 近 的” 平 均 状 态。 换 句 话 说， Region 的 统 计 状 态 越 新 越 能 决 定 其 回 收 的 价 值。 然 后 通 过 这 些 信 息 预 测 现 在 开 始 回 收 的 话， 由 哪 些 Region 组 成 回 收 集 才 可 以 在 不 超 过 期 望 停 顿 时 间 的 约 束 下 获 得 最 高 的 收 益。



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



## 6.CMS

### 从线程角度分析工作阶段

![CMS1](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/CMS1.jpg)

#### 初始标记

stop-the-world，标记GC Roots

#### 并发标记

占用了CMS80%的GC时间，从 GC Roots 的 直 接 关 联 对 象 开 始 遍 历 整 个 对 象 图 的 过 程， 这 个 过 程 耗 时 较 长 但 是 不 需 要 停 顿 用 户 线 程， 可 以 与 垃 圾 收 集 线 程 一 起 并 发 运 行。

#### 重新标记

stop-the-world，重 新 标 记 阶 段 则 是 为 了 修 正 并 发 标 记 期 间， 因 用 户 程 序 继 续 运 作 而 导 致 标 记 产 生 变 动 的 那 一 部 分 对 象 的 标 记 记 录， 这 个 阶 段 的 停 顿 时 间 通 常 会 比 初 始 标 记 阶 段 稍 长 一 些， 但 也 远 比 并 发 标 记 阶 段 的 时 间 短

#### 并发清理

开始清理标记阶段清理出的垃圾，并发清理阶段产生的新的垃圾称为浮动垃圾，等待下一阶段CMS的GC



### CMS的缺点

#### 1.Memory Fragmentation

内存碎片化，CMS使用的是Mark Sweep算法，会导致碎片化，因此当老年代中的内存趋于碎片化时，新生代的对象可能无法存储到老年代中，该情况下会导致CMS转变为Serial Old来进行GC，单线程的GC会导致效率急剧下降

#### 2.Floating Garbage

在并发清理阶段产生的浮动垃圾需要等待下一次的GC才能被清理，但是这部分延迟未清理的空间可能导致新生代的对象无法被正常放入老年代，与此同时会在日志中记录下Concurrent Mode Failure，并切换到Serial Old。可以通过降低-XX:CMSInitiatingOccupancyFraction (92%)这个值，让老年代保持足够的空间。



## 7.收集器的权衡

应 该 如 何 选 择 一 款 适 合 自 己 应 用 的 收 集 器 呢？ 这 个 问 题 的 答 案 主 要 受 以 下 三 个 因 素 影 响：

**1.应 用 程 序 的 主 要 关 注 点 是 什 么**

 如 果 是 数 据 分 析、 科 学 计 算 类 的 任 务， 目 标 是 能 尽 快 算 出 结 果， 那 吞 吐 量 就 是 主 要 关 注 点； 如 果 是 SLA 应 用， 那 停 顿 时 间 直 接 影 响 服 务 质 量， 严 重 的 甚 至 会 导 致 事 务 超 时， 这 样 延 迟 就 是 主 要 关 注 点； 而 如 果 是 客 户 端 应 用 或 者 嵌 入 式 应 用， 那 垃 圾 收 集 的 内 存 占 用 则 是 不 可 忽 视 的。 



**2.运 行 应 用 的 基 础 设 施 如 何**

譬 如 硬 件 规 格， 要 涉 及 的 系 统 架 构 是 x86-32/ 64、 SPARC 还 是 ARM/ Aarch64； 处 理 器 的 数 量 多 少， 分 配 内 存 的 大 小； 选 择 的 操 作 系 统 是 Linux、 Solaris 还 是 Windows 等。 



**3.使 用 JDK 的 发 行 商 是 什 么，版 本 号 是 多 少**

是 ZingJDK/ Zulu、 OracleJDK、 Open-JDK、 OpenJ9 抑 或 是 其 他 公 司 的 发 行 版？ 该 JDK 对 应 了《 Java 虚 拟 机 规 范》 的 哪 个 版 本？