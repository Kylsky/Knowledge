# Docker核心实现技术

作 为 一 种 容 器 虚 拟 化 技 术， Docker 深 度 应 用 了 操 作 系 统 的 多 项 底 层 支 持 技 术。 早 期 版 本 的 Docker 是 基 于 已 经 成 熟 的 Linux Container（ LXC） 技 术 实 现 的。 自 Docker 0.9 版 本 起， Docker 逐 渐 从 LXC 转 移 到 新 的 libcontainer（ https:// github.com/ docker/ libcontainer ）上， 并 且 积 极 推 动 开 放 容 器 规 范 runc， 试 图 打 造 更 通 用 的 底 层 容 器 虚 拟 化 库。 

从 操 作 系 统 功 能 上 看， 目 前 Docker 底 层 依 赖 的 核 心 技 术 主 要 包 括 Linux 操 作 系 统 的 **命 名 空 间**（ Namespace）、 **控 制 组**（ Control Group）、 **联 合 文 件 系 统**（ Union File System） 和 **Linux 网 络 虚 拟 化** 支 持。 本 章 将 介 绍 这 些 核 心 技 术 的 实 现。



## 1.基本架构

Docker 目 前 采 用 了 标 准 的 C/ S 架 构， 如 图 17-1 所 示。 客 户 端 和 服 务 端 既 可 以 运 行 在 一 个 机 器 上， 也 可 运 行 在 不 同 机 器 上 通 过 socket 或 者 RESTful API 来 进 行 通 信。

### 1.1 服务端

Docker 服 务 端 默 认 监 听 本 地 的 unix:/// var/ run/ docker.sock 套 接 字， 只 允 许 本 地 的 root 用 户 或 docker 用 户 组 成 员 访 问。 可 以 通 过-H 选 项 来 修 改 监 听 的 方 式。

例 如， 让 服 务 端 监 听 本 地 的 TCP 连 接 1234 端 口， 如 下 所 示：

```
docker daemon -H 0.0.0.0: 1234
```

### 1.2 客户端

同 样， 客 户 端 默 认 通 过 本 地 的 unix:/// var/ run/ docker.sock 套 接 字 向 服 务 端 发 送 命 令。 如 果 服 务 端 没 有 监 听 在 默 认 的 地 址， 则 需 要 客 户 端 在 执 行 命 令 的 时 候 显 式 指 定 服 务 端 地 址。 例 如， 假 定 服 务 端 监 听 在 本 地 的 TCP 连 接 1234 端 口 tcp:// 127.0.0.1： 1234， 只 有 通 过-H 参 数 指 定 了 正 确 的 地 址 信 息 才 能 连 接 到 服 务 端， 如 下 所 示：

```
docker -H tcp:// 127.0.0.1: 1234 version
```



### 1.3 新的架构

使 用 Docker 时， 必 须 要 启 动 并 保 持 Docker Daemon 的 正 常 运 行， 它 既 要 管 理 容 器 的 运 行 时， 又 要 负 责 提 供 对 外 部 API 的 响 应。 而 一 旦 Docker Daemon 服 务 不 正 常， 则 已 经 运 行 在 Docker 主 机 上 的 容 器 也 往 往 无 法 继 续 使 用。 Docker 团 队 已 经 意 识 到 了 这 个 问 题， 在 较 新 的 版 本（ 1.11.0 +） 中， 开 始 将 维 护 容 器 运 行 的 任 务 放 到 一 个 单 独 的 组 件 containerd 中 来 管 理， 并 且 支 持 OCI 的 runc 规 范。 原 先 的 对 客 户 端 API 的 支 持 则 仍 然 放 在 Docker Daemon， 通 过 解 耦， 大 大 减 少 了 对 Docker Daemon 的 依 赖。



## 2.命名空间



## 3.控制组



## 4.联合文件系统



## 5.Linux网络虚拟化

