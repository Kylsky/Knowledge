## 前言

Class 文 件 格 式 采 用 一 种 类 似 于 C 语 言 结 构 体 的 伪 结 构 来 存 储 数 据， 这 种 伪 结 构 中 只 有 两 种 数 据 类 型：“ 无 符 号 数” 和“ 表”。 后 面 的 解 析 都 要 以 这 两 种 数 据 类 型 为 基 础：

**无 符 号 数** 属 于 基 本 的 数 据 类 型， 以 u1、 u2、 u4、 u8 来 分 别 代 表 1 个 字 节、 2 个 字 节、 4 个 字 节 和 8 个 字 节 的 无 符 号 数， 无 符 号 数 可 以 用 来 描 述 数 字、 索 引 引 用、 数 量 值 或 者 按 照 UTF-8 编 码 构 成 字 符 串 值。 

**表** 是 由 多 个 无 符 号 数 或 者 其 他 表 作 为 数 据 项 构 成 的 复 合 数 据 类 型， 为 了 便 于 区 分， 所 有 **表 的 命 名 都 习 惯 性 地 以“_info” 结 尾**。 表 用 于 描 述 有 层 次 关 系 的 复 合 结 构 的 数 据， 整 个 Class 文 件 本 质 上 也 可 以 视 作 是 一 张 表， 这 张 表 由 下图 所 示 的 数 据 项 按 严 格 顺 序 排 列 构 成。

<img src="http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20201103154845759.png" alt="image-20201103154845759" style="zoom:67%;" />



## 1.魔数

每 个 Class 文 件 的 头 4 个 字 节 被 称 为 魔 数（ Magic Number）， 它 的 唯 一 作 用 是 确 定 这 个 文 件 是 否 为 一 个 能 被 虚 拟 机 接 受 的 Class 文 件。

Class 文 件 的 魔 数 取 得 很 有“ 浪 漫 气 息”， 值 为 0xCAFEBABE（ 咖 啡 宝 贝？）



## 2.版本号

紧 接 着 魔 数 的 4 个 字 节 存 储 的 是 Class 文 件 的 版 本 号： 第 5 和 第 6 个 字 节 是 次 版 本 号（ Minor Version）， 第 7 和 第 8 个 字 节 是 主 版 本 号（ Major Version）。 Java 的 版 本 号 是 从 45 开 始 的， JDK 1.1 之 后 的 每 个 JDK 大 版 本 发 布 主 版 本 号 向 上 加 1（ JDK 1.0 ～ 1.1 使 用 了 45.0 ～ 45.3 的 版 本 号）， 高 版 本 的 JDK 能 向 下 兼 容 以 前 版 本 的 Class 文 件， 但 不 能 运 行 以 后 版 本 的 Class 文 件， 因 为《 Java 虚 拟 机 规 范》 在 Class 文 件 校 验 部 分 明 确 要 求 了 即 使 文 件 格 式 并 未 发 生 任 何 变 化， 虚 拟 机 也 必 须 拒 绝 执 行 超 过 其 版 本 号 的 Class 文 件。



## 3.常量池

常 量 池 是 在 Class 文 件 中 第 一 个 出 现 的 表 类 型 数 据 项 目。

由 于 常 量 池 中 常 量 的 数 量 是 不 固 定 的， 所 以 在 常 量 池 的 入 口 需 要 放 置 一 项 u2 类 型 的 数 据， 代 表 常 量 池 容 量 计 数 值（ constant_pool_count）。

在 Class 文 件 格 式 规 范 制 定 之 时， 设 计 者 将 第 0 项 常 量 空 出 来 是 有 特 殊 考 虑 的， 这 样 做 的 目 的 在 于， 如 果 后 面 某 些 指 向 常 量 池 的 索 引 值 的 数 据 在 特 定 情 况 下 需 要 表 达“ 不 引 用 任 何 一 个 常 量 池 项 目” 的 含 义， 可 以 把 索 引 值 设 置 为 0 来 表 示。 Class 文 件 结 构 中 只 有 常 量 池 的 容 量 计 数 是 从 1 开 始。

常 量 池 中 主 要 存 放 两 大 类 常 量： 字 面 量（ Literal） 和 符 号 引 用（ Symbolic References）。 

**字 面 量** 比 较 接 近 于 Java 语 言 层 面 的 常 量 概 念， 如 文 本 字 符 串、 被 声 明 为 final 的 常 量 值 等。

**符 号 引 用** 则 属 于 编 译 原 理 方 面 的 概 念， 主 要 包 括 下 面 几 类 常 量：

```
1.被 模 块 导 出 或 者 开 放 的 包（ Package）
2.类 和 接 口 的 全 限 定 名（ Fully Qualified Name）
3.字 段 的 名 称 和 描 述 符（ Descriptor）
4.方 法 的 名 称 和 描 述 符
5.方 法 句 柄 和 方 法 类 型（ Method Handle、 Method Type、 Invoke Dynamic）
6.动 态 调 用 点 和 动 态 常 量（ Dynamically-Computed Call Site、 Dynamically-Computed Constant）
```



常 量 池 中 每 一 项 常 量 都 是 一 个 表， 最 初 常 量 表 中 共 有 11 种 结 构 各 不 相 同 的 表 结 构 数。后 来 为 了 更 好 地 支 持 动 态 语 言 调 用， 额 外 增 加 了 4 种 动 态 语 言 相 关 的 常 量 [1] ，为 了 支 持 Java 模 块 化 系 统（ Jigsaw）， 又 加 入 了 CONSTANT_Module_info 和 CONSTANT_Package_info 两 个 常 量， 所 以 截 至 JDK 13， 常 量 表 中 分 别 有 17 种 不 同 类 型 的 常 量。 这 17 类 表 都 有 一 个 共 同 的 特 点， 表 结 构 起 始 的 第 一 位 是 个 u1 类 型 的 标 志 位（ tag）， 代 表 着 当 前 常 量 属 于 哪 种 常 量 类 型。 

<img src="http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20201103163337389.png" alt="image-20201103163337389" style="zoom:67%;" />

<img src="http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20201103163354741.png" alt="image-20201103163354741" style="zoom:67%;" />

以下是常量池中的17种数据类型的结构总表：

<img src="http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20201104100337413.png" alt="image-20201104100337413" style="zoom:67%;" />

![image-20201104100349817](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20201104100349817.png)



## 4.标志位

在 常 量 池 结 束 之 后， 紧 接 着 的 2 个 字 节 代 表 访 问 标 志（ access_flags）， 这 个 标 志 用 于 识 别 一 些 类 或 者 接 口 层 次 的 访 问 信 息， 包 括： 这 个 Class 是 类 还 是 接 口； 是 否 定 义 为 public 类 型； 是 否 定 义 为 abstract 类 型； 如 果 是 类 的 话， 是 否 被 声 明 为 final等 等。 

![image-20201104095450404](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20201104095450404.png)



## 5.类索引

u2 类 型 的 数 据。类 索 引 用 于 确 定 这 个 类 的 全 限 定 名。类 索 引 和 父 类 索 引 用 两 个 u2 类 型 的 索 引 值 表 示， 它 们各自指 向一个类型为CONSTANT_Class_info的 类 描 述 符 常 量， 通 过CONSTANT_Class_info类 型 的 常 量 中 的 索 引 值 可 以 找 到 定 义 在CONSTANT_Utf8_info类 型 的 常 量 中 的 全 限 定 名 字 符 串。



## 6.父类索引

u2 类 型 的 数 据。父 类 索 引 用 于 确 定 这 个 类 的 父 类 的 全 限 定 名。



## 7.接口索引集合

是 一 组 u2 类 型 的 数 据 的 集 合， Class 文 件 中 由 这 **类索引**、**父类索引**、**接口索引集合**来 确 定 该 类 型 的 继 承 关 系。第 一 项 u2 类 型 的 数 据 为 接 口 计 数 器（ interfaces_count）， 表 示 索 引 表 的 容 量。 如 果 该 类 没 有 实 现 任 何 接 口， 则 该 计 数 器 值 为 0， 后 面 接 口 的 索 引 表 不 再 占 用 任 何 字 节。



## 8.字段表集合

字 段 表（ field_info） 用 于 描 述 接 口 或 者 类 中 声 明 的 变 量。 

Java 语 言 中 的“ 字 段”（ Field） **包 括** 类 级 变 量 以 及 实 例 级 变 量， 但 **不 包 括** 在 方 法 内 部 声 明 的 局 部 变 量。 

字 段 可 以 包 括 的 修 饰 符 有 字 段 的 **作 用 域**（ public、 private、 protected 修 饰 符）、 是 **实 例 变 量** 还 是 **类 变 量**（ static 修 饰 符）、 **可 变 性**（ final）、 **并 发 可 见 性**（ volatile 修 饰 符， 是 否 强 制 从 主 内 存 读 写）、 **可 否 被 序 列 化**（ transient 修 饰 符）、 **字 段 数 据 类 型**（ 基 本 类 型、 对 象、 数 组）、 **字 段 名 称**。 上 述 这 些 信 息 中， 各 个 修 饰 符 都 是 布 尔 值， 要 么 有 某 个 修 饰 符， 要 么 没 有， 很 适 合 使 用 标 志 位 来 表 示。 而 字 段 叫 做 什 么 名 字、 字 段 被 定 义 为 什 么 数 据 类 型， 这 些 都 是 无 法 固 定 的， 只 能 引 用 常 量 池 中 的 常 量 来 描 述。



字段表的结构：

<img src="http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20201104102827891.png" alt="image-20201104102827891" style="zoom:67%;" />



### 8.1 access_flags

字 段 修 饰 符 放 在 access_flags 项 目 中， 它 与 类 中 的 access_flags 项 目 是 非 常 类 似 的， 都 是 一 个 u2 的 数 据 类 型

<img src="http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20201104103034013.png" alt="image-20201104103034013" style="zoom:67%;" />



### 8.2 name_index 和 descriptor_index

都 是 对 常 量 池 项 的 引 用， 分 别 代 表 着 字 段 的 简 单 名 称 以 及 字 段 和 方 法 的 描 述 符。

**简 单 名 称** 是 指 没 有 类 型 和 参 数 修 饰 的 方 法 或 者 字 段 名 称， 如 inc() 方 法 和 m 字 段 的 简 单 名 称 分 别 就 是“ inc” 和“ m”。相 比 于 简 单 名 称， 方 法 和 字 段 的 描 述 符 就 要 复 杂 一 些。

**描 述 符** 的 作 用 是 用 来 描 述 字 段 的 数 据 类 型、 方 法 的 参 数 列 表（ 包 括 数 量、 类 型 以 及 顺 序） 和 返 回 值。 根 据 描 述 符 规 则， 基 本 数 据 类 型（ byte、 char、 double、 float、 int、 long、 short、 boolean） 以 及 代 表 无 返 回 值 的 void 类 型 都 用 一 个 大 写 字 符 来 表 示， 而 对 象 类 型 则 用 字 符 L 加 对 象 的 全 限 定 名 来 表 示。描述符标志字段的含义如下：

<img src="http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20201104104317606.png" alt="image-20201104104317606" style="zoom:67%;" />



### 8.3 属性表集合

属 性 表 集 合， 用 于 存 储 一 些 额 外 的 信 息， 字 段 表 可 以 在 属 性 表 中 附 加 描 述 零 至 多 项 的 额 外 信 息。 如 一 个 字 段 m， 它 的 属 性 表 计 数 器 为 0， 也 就 是 没 有 需 要 额 外 描 述 的 信 息， 但 是， 如 果 将 字 段 m 的 声 明 改 为“ final static int m = 123；”， 那 就 可 能 会 存 在 一 项 名 称 为 ConstantValue 的 属 性， 其 值 指 向 常 量 123。



### 8.4 补充

字 段 表 集 合 中 不 会 列 出 从 父 类 或 者 父 接 口 中 继 承 而 来 的 字 段， 但 有 可 能 出 现 原 本 Java 代 码 之 中 不 存 在 的 字 段， 譬 如 在 内 部 类 中 为 了 保 持 对 外 部 类 的 访 问 性， 编 译 器 就 会 自 动 添 加 指 向 外 部 类 实 例 的 字 段。 另 外， 在 Java 语 言 中 字 段 是 无 法 重 载 的， 两 个 字 段 的 数 据 类 型、 修 饰 符 不 管 是 否 相 同， 都 必 须 使 用 不 一 样 的 名 称， 但 是 对 于 Class 文 件 格 式 来 讲， 只 要 两 个 字 段 的 描 述 符 不 是 完 全 相 同， 那 字 段 重 名 就 是 合 法 的。



## 9.方法表集合

Class 文 件 存 储 格 式 中 对 方 法 的 描 述 与 对 字 段 的 描 述 采 用 了 几 乎 完 全 一 致 的 方 式， 方 法 表 的 结 构 如 同 字 段 表 一 样， 依 次 包 括 访 问 标 志（ access_flags）、 名 称 索 引（ name_index）、 描 述 符 索 引（ descriptor_index）、 属 性 表 集 合（ attributes） 几 项。

这 些 数 据 项 目 的 含 义 也 与 字 段 表 中 的 非 常 类 似， 仅 在 访 问 标 志 和 属 性 表 集 合 的 可 选 项 中 有 所 区 别。

<img src="http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20201104110133464.png" alt="image-20201104110133464" style="zoom:67%;" />



### 9.1 方法访问标志

![image-20201104110254078](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20201104110254078.png)



### 9.2 属性表

方 法 里 的 Java 代 码， 经 过 Javac 编 译 器 编 译 成 字 节 码 指 令 之 后， 存 放 在 方 法 属 性 表 集 合 中 一 个 名 为“ Code” 的 属 性 里 面。



## 10.再谈属性表

属 性 表（ attribute_info） 在 前 面 的 讲 解 之 中 已 经 出 现 过 数 次， Class 文 件、 字 段 表、 方 法 表 都 可 以 携 带 自 己 的 属 性 表 集 合， 以 描 述 某 些 场 景 专 有 的 信 息。 与 Class 文 件 中 其 他 的 数 据 项 目 要 求 严 格 的 顺 序、 长 度 和 内 容 不 同， 属 性 表 集 合 的 限 制 稍 微 宽 松 一 些， 不 再 要 求 各 个 属 性 表 具 有 严 格 顺 序， 并 且《 Java 虚 拟 机 规 范》 允 许 只 要 不 与 已 有 属 性 名 重 复， 任 何 人 实 现 的 编 译 器 都 可 以 向 属 性 表 中 写 入 自 己 定 义 的 属 性 信 息， Java 虚 拟 机 运 行 时 会 忽 略 掉 它 不 认 识 的 属 性。 为 了 能 正 确 解 析 Class 文 件，《 Java 虚 拟 机 规 范》 最 初 只 预 定 义 了 9 项 所 有 Java 虚 拟 机 实 现 都 应 当 能 识 别 的 属 性， 而 在 最 新 的《 Java 虚 拟 机 规 范》 的 Java SE 12 版 本 中， 预 定 义 属 性 已 经 增 加 到 29 项。

<img src="http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20201104110613773.png" alt="image-20201104110613773" style="zoom:67%;" />

<img src="http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20201104110636561.png" alt="image-20201104110636561" style="zoom:67%;" />

对 于 每 一 个 属 性， 它 的 名 称 都 要 从 常 量 池 中 引 用 一 个 CONSTANT_Utf8_info 类 型 的 常 量 来 表 示， 而 属 性 值 的 结 构 则 是 完 全 自 定 义 的， 只 需 要 通 过 一 个 u4 的 长 度 属 性 去 说 明 属 性 值 所 占 用 的 位 数 即 可。 一 个 符 合 规 则 的 属 性 表 应 该 满 足 下 表：

<img src="http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20201104110758127.png" alt="image-20201104110758127" style="zoom:67%;" />