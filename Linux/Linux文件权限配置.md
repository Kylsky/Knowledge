## 1.使用者与群组

linux账号信息:

/etc/passwd

个人密码：

/etc/shadow

群组名称：

/etc/group



![image-20200826143025858](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20200826143025858.png)

![image-20200826143102194](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20200826143102194.png)

第 一 栏 代 表 这 个 文 件 的 类 型 与 权 限

第一个字符代表这个文件是否为目录，若是d，则表示为目录

接 下 来 的 字 符 中， 以 三 个 为 一 组， 且 均 为“ rwx” 的 三 个 参 数 的 组 合。 其 中，[ r ]代 表 可 读（ read）、[ w ]代 表 可 写（ write）、[ x ]代 表 可 执 行（ execute）。 要 注 意 的 是， 这 三 个 权 限 的 位 置 不 会 改 变， 如 果 没 有 权 限， 就 会 出 现 减 号[ - ]而 已。

第 二 栏 表 示 有 多 少 文 件 名 链 接 到 此 节 点（ i-node）



## 2.改变文件属性与权限

![image-20200826150205714](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20200826150205714.png)

### 2.1 chgrp

改 变 文 件 所 属 群 组 

### 2.2 chown

改 变 文 件 拥 有 者 ，也能改变所属群组

### 2.3 chmod

改 变 文 件 的 权 限, SUID, SGID, SBIT 等 等 的 特 性

![image-20200826145926236](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20200826145926236.png)



## 3.文件种类与扩展名

### 3.1 文件种类

#### 3.1.1 正规文件

**1.纯文本文件**

略

**2.二进制档**

可执行文件等

**3.数据格式文件**

数 据 格 式 文 件（ data）： 有 些 程 序 在 运 行 的 过 程 当 中 会 读 取 某 些 特 定 格 式 的 文 件， 那 些 特 定 格 式 的 文 件 可 以 被 称 为 数 据 文 件 （data file）。 举 例 来 说， 我 们 的 Linux 在 使 用 者 登 陆 时， 都 会 将 登 录 的 数 据 记 录 在 /var/ log/ wtmp 那 个 文 件 内， 该 文 件 是 一 个 data file， 他 能 够 通 过 last 这 个 指 令 读 出 来！ 但 是 使 用 cat 时， 会 读 出 乱 码 ～ 因 为 他 是 属 于 一 种 特 殊 格 式 的 文 件。



#### 3.1.2 目录(directory)

略



#### 3.1.3 链接(link)文件

就 是 类 似 Windows 系 统 下 面 的 快 捷 方 式 ， 第 一 个 属 性 为 [ l ]（ 英 文 L 的 小 写）， 例 如 [lrwxrwxrwx] ；



#### 3.1.4 设备(device)与设备文件

**1.区块(block)设备文件**

一 些 储 存 数 据， 以 提 供 系 统 随 机 存 取 的 周 边 设 备， 举 例 来 说， 硬 盘 与 软 盘 等 就 是 。 你 可 以 随 机 的 在 硬 盘 的 不 同 区 块 读 写， 这 种 设 备 就 是 区 块 设 备 。查 一 下/ dev/ sda ， 会 发 现 第 一 个 属 性 为[ b ]

**2.字符(character)设备文件**

一 些 序 列 埠 的 周 边 设 备， 例 如 键 盘、 鼠 标 等 等。这 些 设 备 的 特 色 就 是“ 一 次 性 读 取” 的， 不 能 够 截 断 输 出。 举 例 来 说， 你 不 可 能 让 鼠 标“ 跳 到” 另 一 个 画 面， 而 是“ 连 续 性 滑 动” 到 另 一 个 地 方 啊！ 第 一 个 属 性 为 [ c ]。



#### 3.1.5 数据接口文件(sockets)

既 然 被 称 为 数 据 接 口 文 件， 想 当 然 尔， 这 种 类 型 的 文 件 通 常 被 用 在 网 络 上 的 数 据 承 接 了。 我 们 可 以 启 动 一 个 程 序 来 监 听 用 户 端 的 要 求， 而 用 户 端 就 可 以 通 过 这 个 socket 来 进 行 数 据 的 沟 通 了。 第 一 个 属 性 为 [ s ]， 最 常 在/ run 或/ tmp 这 些 个 目 录 中 看 到 这 种 文 件 类 型 了。



#### 3.1.6 数据输送档(FIFO,pipe)

用 于 解 决 多 个 程 序 同 时 存 取 一 个 文 件 所 造 成 的 错 误 问 题。 FIFO 是 first-in-first-out 的 缩 写。 第 一 个 属 性 为[ p] 。



### 3.2 扩展名

略



### 3.3 文件长度限制

在 Linux 下 面， 使 用 传 统 的 Ext2/ Ext3/ Ext4 文 件 系 统 以 及 近 来 被 CentOS 7 当 作 默 认 文 件 系 统 的 xfs 而 言， 针 对 文 件 的 文 件 名 长 度 限 制 为：

```
单 一 文 件 或 目 录 的 最 大 容 许 文 件 名 为 255Bytes， 以 一 个 ASCII 英 文 占 用 一 个 Bytes 来 说， 则 大 约 可 达 255 个 字 符 长 度。 若 是 以 每 个 中 文 字 占 用 2Bytes 来 说， 最 大 文 件 名 就 是 大 约 在 128 个 中 文 字 
```



## 4.Linux目录配置

### 4.1 FHS

FileSystem Hierarchy Standard

<img src="http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20200826152325888.png" alt="image-20200826152325888" style="zoom:67%;" />



### 4.2 根目录的意义与内容

<img src="http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20200826153325499.png" alt="image-20200826153315567" style="zoom:67%;" />

<img src="http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20200826153325499.png" alt="image-20200826153325499" style="zoom:67%;" />

<img src="http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20200826153342399.png" alt="image-20200826153342399" style="zoom:67%;" />

<img src="http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20200826153354803.png" alt="image-20200826153354803" style="zoom:67%;" />

<img src="http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20200826153406033.png" alt="image-20200826153406033" style="zoom:67%;" />

<img src="http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20200826153415420.png" alt="image-20200826153415420" style="zoom:67%;" />

<img src="http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20200826153426425.png" alt="image-20200826153426425" style="zoom:67%;" />

事 实 上 FHS 针 对 根 目 录 所 定 义 的 标 准 就 仅 有 上 面 的 内容， 不 过 Linux 下 面 还 有 许 多 目 录 也 需 要 了 解 一 下 。 下 面 是 几 个 在 Linux 当 中 也 是 非 常 重 要 的 目 录 ：

<img src="http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20200826153534906.png" alt="image-20200826153534906" style="zoom:67%;" />

<img src="http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20200826153550289.png" alt="image-20200826153550289" style="zoom:67%;" />

早 期 Linux 在 设 计 的 时 候， 若 发 生 问 题 时， 救 援 模 式 通 常 仅 挂 载 根 目 录 而 已， 因 此 有 五 个 重 要 的 目 录 被 要 求 一 定 要 与 根 目 录 放 置 在 一 起， 那 就 是 /etc, /bin, /dev, /lib, /sbin 这 五 个 重 要 目 录。 现 在 许 多 的 Linux distributions 由 于 已 经 将 许 多 非 必 要 的 文 件 移 出 /usr 之 外 了， 所 以 /usr 也 是 越 来 越 精 简， 同 时 因 为 /usr 被 建 议 为“ 即 使 挂 载 成 为 只 读， 系 统 还 是 可 以 正 常 运 行” 的 模 样， 所 以 救 援 模 式 也 能 同 时 挂 载 /usr。 例 如 CentOS 7. x 版 本 在 救 援 模 式 的 情 况 下 就 是 这 样。 因 此 那 个 五 大 目 录 的 限 制 已 经 被 打 破 了。例 如 CentOS 7. x 就 已 经 将 /sbin, /bin, /lib 通 通 移 动 到 /usr 下 面 了 
