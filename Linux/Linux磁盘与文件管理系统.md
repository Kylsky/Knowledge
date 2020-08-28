## 1.Linux ext2文件系统(inode)

![image-20200827151013514](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20200827151013514.png)

### 1.1 data block

data block 是 用 来 放 置 文 件 内 容 数 据 地 方， 在 Ext2 文 件 系 统 中 所 支 持 的 block 大 小 有 1K, 2K 及 4K 三 种 而 已。 在 格 式 化 时 block 的 大 小 就 固 定 了， 且 每 个 block 都 有 编 号， 以 方 便 inode 的 记 录 啦。 不 过 要 注 意 的 是， 由 于 block 大 小 的 差 异， 会 导 致 该 文 件 系 统 能 够 支 持 的 最 大 磁 盘 容 量 与 最 大 单 一 文 件 大 小 并 不 相 同。 因 为 block 大 小 而 产 生 的 Ext2 文 件 系 统 限 制 如 下

<img src="http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20200827151840030.png" alt="image-20200827151840030" style="zoom:67%;" />



### 1.2 inode

<img src="http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20200827152305732.png" alt="image-20200827152305732" style="zoom:67%;" />



### 1.3 superblock

superblock记录了以下信息：

1.block 与 inode 的 总 量； 

2.未 使 用 与 已 使 用 的 inode / block 数 量； 

3.block 与 inode 的 大 小 （block 为 1, 2, 4K， inode 为 128Bytes 或 256Bytes）； 

4.filesystem 的 挂 载 时 间、 最 近 一 次 写 入 数 据 的 时 间、 最 近 一 次 检 验 磁 盘 （fsck） 的 时 间 等 文 件 系 统 的 相 关 信 息； 

5.valid bit 数 值，若 此 文 件 系 统 已 被 挂 载， 则 valid bit 为 0 ，若 未 被 挂 载， 则 valid bit 为 1 。



### 1.4 Filesystem Description

这 个 区 段 可 以 描 述 每 个 block group 的 开 始 与 结 束 的 block 号 码， 以 及 说 明 每 个 区 段 （superblock, bitmap, inodemap, data block） 分 别 介 于 哪 一 个 block 号 码 之 间。 这 部 份 也 能 够 用 dumpe2fs 来 观 察 的。



### 1.5 block bitmap

区块对照表。如 果 想 要 新 增 文 件 时 会 用 到 block， 要 使 用 哪 个 block 来 记 录 呢？ 当 然 是 选 择“ 空 的 block ”来 记 录 新 文 件 的 数 据 啰。 那 怎 么 知 道 哪 个 block 是 空 的？ 这 就 得 要 通 过 block bitmap 的 辅 助 了。 从 block bitmap 当 中 可 以 知 道 哪 些 block 是 空 的， 因 此 我 们 的 系 统 就 能 够 很 快 速 的 找 到 可 使 用 的 空 间 来 处 置 文 件



### 1.6 inode bitmap

inode对照表。这 个 其 实 与 block bitmap 是 类 似 的 功 能， 只 是 block bitmap 记 录 的 是 使 用 与 未 使 用 的 block 号 码， 至 于 inode bitmap 则 是 记 录 使 用 与 未 使 用 的 inode 号 码 

## 2.文件系统与目录树关系

### 2.1 目录

当 我 们 在 Linux 下 的 文 件 系 统 创 建 一 个 目 录 时， 文 件 系 统 会 分 配 一 个 inode 与 至 少 一 块 block 给 该 目 录。 其 中， inode 记 录 该 目 录 的 相 关 权 限 与 属 性， 并 可 记 录 分 配 到 的 那 块 block 号 码； 而 block 则 是 记 录 在 这 个 目 录 下 的 文 件 名 与 该 文 件 名 占 用 的 inode 号 码 数 据。 也 就 是 说 目 录 所 占 用 的 block 内 容 在 记 录 如 下 的 信 息

![image-20200827153819299](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20200827153819299.png)

### 2.2 文件

当 我 们 在 Linux 下 的 ext2 创 建 一 个 一 般 文 件 时， ext2 会 分 配 一 个 inode 与 相 对 于 该 文 件 大 小 的 block 数 量 给 该 文 件。 例 如： 假 设 我 的 一 个 block 为 4 KBytes ，而 我 要 创 建 一 个 100 KBytes 的 文 件， 那 么 linux 将 分 配 一 个 inode 与 25 个 block 来 储 存 该 文 件！ 但 同 时 请 注 意， 由 于 inode 仅 有 12 个 直 接 指 向， 因 此 还 要 多 一 个 block 来 作 为 区 块 号 码 的 记 录，因 此 总 共 需 要 26 个 block 块。



### 3.日 志 式 文 件 系 统 的 功 能

在 一 般 正 常 的 情 况 下， 上 述 的 新 增 动 作 当 然 可 以 顺 利 的 完 成。 但 是 如 果 有 个 万 一 怎 么 办？ 例 如 你 的 文 件 在 写 入 文 件 系 统 时， 因 为 不 知 名 原 因 导 致 系 统 中 断（ 例 如 突 然 的 停 电 啊、 系 统 核 心 发 生 错 误 啊 ～ 等 等 的 怪 事 发 生 时）， 所 以 写 入 的 数 据 仅 有 inode table 及 data block 而 已， 最 后 一 个 同 步 更 新 中 介 数 据 的 步 骤 并 没 有 做 完， 此 时 就 会 发 生 metadata 的 内 容 与 实 际 数 据 存 放 区 产 生 不 一 致 （Inconsistent） 的 情 况 了。 

既 然 有 不 一 致 当 然 就 得 要 克 服！ 在 早 期 的 Ext2 文 件 系 统 中， 如 果 发 生 这 个 问 题， 那 么 系 统 在 重 新 开 机 的 时 候， 就 会 借 由 Superblock 当 中 记 录 的 valid bit （是 否 有 挂 载） 与 filesystem state （clean 与 否） 等 状 态 来 判 断 是 否 强 制 进 行 数 据 一 致 性 的 检 查！ 若 有 需 要 检 查 时 则 以 e2fsck 这 支 程 序 来 进 行 的。

不 过， 这 样 的 检 查 真 的 是 很 费 时 ～ 因 为 要 针 对 metadata 区 域 与 实 际 数 据 存 放 区 来 进 行 比 对， 呵 呵 ～ 得 要 搜 寻 整 个 filesystem 呢 ～ 如 果 你 的 文 件 系 统 有 100GB 以 上， 而 且 里 面 的 文 件 数 量 又 多 时， 哇！ 系 统 真 忙 碌 ～ 而 且 在 对 Internet 提 供 服 务 的 服 务 器 主 机 上 面， 这 样 的 检 查 真 的 会 造 成 主 机 复 原 时 间 的 拉 长 ～ 真 是 麻 烦 ～ 这 也 就 造 成 后 来 所 谓 日 志 式 文 件 系 统 的 兴 起 了。

### 日志式文件系统

为 了 避 免 上 述 提 到 的 文 件 系 统 不 一 致 的 情 况 发 生， 因 此 我 们 的 前 辈 们 想 到 一 个 方 式， 如 果 在 我 们 的 filesystem 当 中 规 划 出 一 个 区 块， 该 区 块 专 门 在 记 录 写 入 或 修 订 文 件 时 的 步 骤， 那 不 就 可 以 简 化 一 致 性 检 查 的 步 骤 了？



## 3.XFS文件系统简介

Ext 文 件 系 统 家 族 对 于 文 件 格 式 化 的 处 理 方 面， 采 用 的 是 预 先 规 划 出 所 有 的 inode/ block/ meta data 等 数 据， 未 来 系 统 可 以 直 接 取 用， 不 需 要 再 进 行 动 态 配 置 的 作 法。 这 个 作 法 在 早 期 磁 盘 容 量 还 不 大 的 时 候 还 算 OK 没 啥 问 题， 但 时 至 今 日， 磁 盘 容 量 越 来 越 大， 连 传 统 的 MBR 都 已 经 被 GPT 所 取 代， 连 我 们 这 些 老 人 家 以 前 听 到 的 超 大 TB 容 量 也 已 经 不 够 看 了！ 现 在 都 已 经 说 到 PB 或 EB 以 上 容 量 了 呢！ 那 你 可 以 想 像 得 到， 当 你 的 TB 以 上 等 级 的 传 统 ext 家 族 文 件 系 统 在 格 式 化 的 时 候， 光 是 系 统 要 预 先 分 配 inode 与 block 就 消 耗 很 多 时 间 
