# Dockerfile

## 1.基本结构

一般而言，Dockerfile可分为四部分：基础镜像信息、维护者信息、镜像操作指令和容器启动时执行指令，如：

```
# This Dockerfile uses the ubuntu image 
# VERSION 2 - EDITION 1 
# Author: docker_user 
# Command format: Instruction [arguments / command] .. 
# Base image to use, this must be set as the first line FROM ubuntu 
# Maintainer: docker_user < docker_user at email.com > (@ docker_user) MAINTAINER docker_user docker_user@ email.com 
# Commands to update the image 
RUN echo "deb http:// archive.ubuntu.com/ ubuntu/ raring main universe" > > /etc/ apt/ sources.list 
RUN apt-get update && apt-get install -y nginx RUN echo "\ ndaemon off;" > > /etc/ nginx/ nginx.conf 
# Commands when creating a new container CMD /usr/ sbin/ nginx
```



## 2.指令说明

<img src="http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20200825112356931.png" alt="image-20200825112356931" style="zoom:67%;" />

### 2.1 FROM

指 定 所 创 建 镜 像 的 基 础 镜 像， 如 果 本 地 不 存 在， 则 默 认 会 去 Docker Hub 下 载 指 定 镜 像。 格 式 为 FROM < image >， 或 FROM < image >： < tag >， 或 FROM < image >@ < digest >。 任 何 Dockerfile 中 的 第 一 条 指 令 必 须 为 FROM 指 令。 并 且， 如 果 在 同 一 个 Dockerfile 中 创 建 多 个 镜 像， 可 以 使 用 多 个 FROM 指 令（ 每 个 镜 像 一 次）。



### 2.2 MAINTAINER

指 定 维 护 者 信 息， 格 式 为 MAINTAINER < name >。 例 如：

```
MAITAINER kyle@98.com
```



### 2.3 RUN

格 式 为 RUN < command > 或 RUN[" executable"，" param1"，" param2"]。 注 意， 后 一 个 指 令 会 被 解 析 为 Json 数 组， 因 此 必 须 用 双 引 号。 前 者 默 认 将 在 shell 终 端 中 运 行 命 令， 即/ bin/ sh-c； 后 者 则 使 用 exec 执 行， 不 会 启 动 shell 环 境。 指 定 使 用 其 他 终 端 类 型 可 以 通 过 第 二 种 方 式 实 现， 例 如 RUN["/ bin/ bash"，"-c"，" echo hello"]。 每 条 RUN 指 令 将 在 当 前 镜 像 的 基 础 上 执 行 指 定 命 令， 并 提 交 为 新 的 镜 像。 当 命 令 较 长 时 可 以 使 用\ 来 换 行。 例 如：

```
RUN apt-get update \ 
&& apt-get install -y libsnappy-dev zlib1g-dev libbz2-dev \ 
&& rm -rf /var/ cache/ apt
```



### 2.4 CMD

CMD 指 令 用 来 指 定 启 动 容 器 时 默 认 执 行 的 命 令。 它 支 持 三 种 格 式：

1.CMD[" executable"，" param1"，" param2"] 使 用 exec 执 行， 是 推 荐 使 用 的 方 式； 

2.CMD command param1 param2 在/ bin/ sh 中 执 行， 提 供 给 需 要 交 互 的 应 用； 

3.CMD[" param1"，" param2"] 提 供 给 ENTRYPOINT 的 默 认 参 数。 

每 个 Dockerfile 只 能 有 一 条 CMD 命 令。 如 果 指 定 了 多 条 命 令， 只 有 最 后 一 条 会 被 执 行。 如 果 用 户 启 动 容 器 时 手 动 指 定 了 运 行 的 命 令（ 作 为 run 的 参 数）， 则 会 覆 盖 掉 CMD 指 定 的 命 令。



### 2.5 LABEL

LABEL 指 令 用 来 指 定 生 成 镜 像 的 元 数 据 标 签 信 息。 格 式 为 LABEL < key > = < value > < key > = < value > < key > = < value >...。 例 如：

```
LABEL version =" 1.0" LABEL description =" This text illustrates \ 
that label-values can span multiple lines."
```



### 2.6 EXPOSE

声 明 镜 像 内 服 务 所 监 听 的 端 口。 格 式 为 EXPOSE < port >[ < port >...]。 例 如：

```
EXPOSE 22 80 8443
```

注 意， 该 指 令 只 是 起 到 声 明 作 用， 并 不 会 自 动 完 成 端 口 映 射。 在 启 动 容 器 时 需 要 使 用-P， Docker 主 机 会 自 动 分 配 一 个 宿 主 机 的 临 时 端 口 转 发 到 指 定 的 端 口； 使 用-p， 则 可 以 具 体 指 定 哪 个 宿 主 机 的 本 地 端 口 会 映 射 过 来。



### 2.7 ENV

指 定 环 境 变 量， 在 镜 像 生 成 过 程 中 会 被 后 续 RUN 指 令 使 用， 在 镜 像 启 动 的 容 器 中 也 会 存 在。 格 式 为 ENV < key > < value > 或 ENV < key > = < value >...。 例 如：

```
ENV PG_MAJOR 9.3 ENV PG_VERSION 9.3.4 RUN curl -SL http:// example.com/ postgres-$ PG_VERSION.tar.xz | tar -xJC /usr/ src/ postgress && …
```

指 令 指 定 的 环 境 变 量 在 运 行 时 可 以 被 覆 盖 掉， 如 docker run--env < key > = < value > built_image。



### 2.8 ADD

该 命 令 将 复 制 指 定 的 < src > 路 径 下 的 内 容 到 容 器 中 的 < dest > 路 径 下。 格 式 为 ADD < src > < dest >。 其 中 < src > 可 以 是 Dockerfile 所 在 目 录 的 一 个 相 对 路 径（ 文 件 或 目 录）， 也 可 以 是 一 个 URL， 还 可 以 是 一 个 tar 文 件（ 如 果 为 tar 文 件， 会 自 动 解 压 到 < dest > 路 径 下）。 < dest > 可 以 是 镜 像 内 的 绝 对 路 径， 或 者 相 对 于 工 作 目 录（ WORKDIR） 的 相 对 路 径。 路 径 支 持 正 则 格 式， 例 如：

```
ADD *. c /code/
```



### 2.9 COPY

格 式 为 COPY < src > < dest >。 复 制 本 地 主 机 的 < src >（ 为 Dockerfile 所 在 目 录 的 相 对 路 径、 文 件 或 目 录） 下 的 内 容 到 镜 像 中 的 < dest > 下。 目 标 路 径 不 存 在 时， 会 自 动 创 建。 路 径 同 样 支 持 正 则 格 式。 当 使 用 本 地 目 录 为 源 目 录 时， 推 荐 使 用 COPY。



### 2.10 ENTRYPOINT

指 定 镜 像 的 默 认 入 口 命 令， 该 入 口 命 令 会 在 启 动 容 器 时 作 为 根 命 令 执 行， 所 有 传 入 值 作 为 该 命 令 的 参 数。 支 持 两 种 格 式：

```
ENTRYPOINT ["executable","param1"，"param2"]
ENTRYPOINT command param1 param2
```

每 个 Dockerfile 中 只 能 有 一 个 ENTRYPOINT， 当 指 定 多 个 时， 只 有 最 后 一 个 有 效。 在 运 行 时， 可 以 被--entrypoint 参 数 覆 盖 掉， 如 docker run--entrypoint。



### 2.11 VOLUME

创 建 一 个 数 据 卷 挂 载 点。 格 式 为 VOLUME["/ data"]。 可 以 从 本 地 主 机 或 其 他 容 器 挂 载 数 据 卷， 一 般 用 来 存 放 数 据 库 和 需 要 保 存 的 数 据 等。与docker run创建的挂载点不同，通过 VOLUME 指令创建的挂载点，无法指定主机上对应的目录，是自动生成的。



### 2.12 USER

指 定 运 行 容 器 时 的 用 户 名 或 UID， 后 续 的 RUN 等 指 令 也 会 使 用 指 定 的 用 户 身 份。 格 式 为 USER daemon。 当 服 务 不 需 要 管 理 员 权 限 时， 可 以 通 过 该 命 令 指 定 运 行 用 户， 并 且 可 以 在 之 前 创 建 所 需 要 的 用 户。 例 如：

```
RUN groupadd -r postgres && useradd -r -g postgres postgres
```



### 2.13 WORKDIR

为 后 续 的 RUN、 CMD 和 ENTRYPOINT 指 令 配 置 工 作 目 录。 格 式 为 WORKDIR/ path/ to/ workdir。

可 以 使 用 多 个 WORKDIR 指 令， 后 续 命 令 如 果 参 数 是 相 对 路 径， 则 会 基 于 之 前 命 令 指 定 的 路 径。 例 如：

```
WORKDIR /a 
WORKDIR b 
WORKDIR c 
RUN pwd
```

则 最 终 路 径 为/ a/ b/ c



### 2.14 ARG

指 定 一 些 镜 像 内 使 用 的 参 数（ 例 如 版 本 号 信 息 等）， 这 些 参 数 在 执 行 docker build 命 令 时 才 以--build-arg < varname > = < value > 格 式 传 入。 

格 式 为 ARG < name >[ = < default value >]。 

 可 以 用 docker build --build-arg < name > = < value >来 指 定 参 数 值。



### 2.15 ONBUILD

配 置 当 所 创 建 的 镜 像 作 为 其 他 镜 像 的 基 础 镜 像 时， 所 执 行 的 创 建 操 作 指 令。 格 式 为 ONBUILD[ INSTRUCTION]。 例 如， Dockerfile 使 用 如 下 的 内 容 创 建 了 镜 像 image-A：

```
[...] 
ONBUILD ADD . /app/ src 
ONBUILD RUN /usr/ local/ bin/ python-build --dir /app/ src [...]
```

如 果 基 于 image-A 创 建 新 的 镜 像 时， 新 的 Dockerfile 中 使 用 FROM image-A 指 定 基 础 镜 像， 会 自 动 执 行 ONBUILD 指 令 的 内 容， 等 价 于 在 后 面 添 加 了 两 条 指 令：

```
FROM image-A 
#Automatically run the following 
ADD . /app/ src 
RUN /usr/ local/ bin/ python-build --dir /app/ src
```



### 2.16 STOPSIGNAL

指 定 所 创 建 镜 像 启 动 的 容 器 接 收 退 出 的 信 号 值。 例 如：

```
STOPSIGNAL signal
```



### 2.17 HEALTHCHECK

配 置 所 启 动 容 器 如 何 进 行 健 康 检 查（ 如 何 判 断 健 康 与 否）， 自 Docker 1.12 开 始 支 持。 格 式 有 两 种：



### 2.18 SHELL

指定其他命令使用shell时的默认shell类型，默认为["/bin/sh","-c"]



## 3.创建镜像

编 写 完 成 Dockerfile 之 后， 可 以 通 过 docker build 命 令 来 创 建 镜 像。 基 本 的 格 式 为 docker build[ 选 项] 内 容 路 径， 该 命 令 将 读 取 指 定 路 径 下（ 包 括 子 目 录） 的 Dockerfile， 并 将 该 路 径 下 的 所 有 内 容 发 送 给 Docker 服 务 端， 由 服 务 端 来 创 建 镜 像。 因 此 除 非 生 成 镜 像 需 要， 否 则 一 般 建 议 放 置 Dockerfile 的 目 录 为 空 目 录。 有 两 点 经 验： 

1.如 果 使 用 非 内 容 路 径 下 的 Dockerfile， 可 以 通 过**-f** 选 项 来 指 定 其 路 径。 

2.要 指 定 生 成 镜 像 的 标 签 信 息， 可 以 使 用**-t** 选 项。 例 如， 指 定 Dockerfile 所 在 路 径 为/ tmp/ docker_builder/， 并 且 希 望 生 成 镜 像 标 签 为 build_repo/ first_image， 可 以 使 用 下 面 的 命 令：



## 4.dockerignore文件

可 以 通 过. dockerignore 文 件（ 每 一 行 添 加 一 条 匹 配 模 式） 来 让 Docker 忽 略 匹 配 模 式 路 径 下 的 目 录 和 文 件。 例 如：

```
# comment 
*/ temp* 
*/*/ temp* 
tmp? 
~*
```

