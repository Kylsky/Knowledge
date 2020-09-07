## 1.vi与vim

### 1.1 区别

简 单 的 来 说， vi 是 老 式 的 文 书 处 理 器， 不 过 功 能 已 经 很 齐 全 了， 但 是 还 是 有 可 以 进 步 的 地 方。 vim 则 可 以 说 是 程 序 开 发 者 的 一 项 很 好 用 的 工 具， 就 连 vim 的 官 方 网 站 （http:// www.vim.org） 自 己 也 说 vim 是 一 个“ 程 序 开 发 工 具” 而 不 是 文 书 处 理 软 件。 因 为 vim 里 面 加 入 了 很 多 额 外 的 功 能， 例 如 支 持 正 则 表 达 式 的 搜 寻 架 构、 多 文 件 编 辑、 区 块 复 制 等 等。



### 1.2 vi的使用

基 本 上 vi 共 分 为 三 种 模 式， 分 别 是“ 一 般 指 令 模 式”、“ 编 辑 模 式” 与“ 命 令 行 命 令 模 式”。 这 三 种 模 式 的 作 用 分 别 是：

#### 1.2.1 一般指令模式

以 vi 打 开 一 个 文 件 就 直 接 进 入 一 般 指 令 模 式 了（ 这 是 默 认 的 模 式， 也 简 称 为 一 般 模 式）。 在 这 个 模 式 中， 你 可 以 使 用“ 上 下 左 右” 按 键 来 移 动 光 标， 你 可 以 使 用“ 删 除 字 符” 或“ 删 除 整 列” 来 处 理 文 件 内 容， 也 可 以 使 用“ 复 制、 贴 上” 来 处 理 你 的 文 件 数 据。



#### 1.2.2 编辑模式

在 一 般 指 令 模 式 中 可 以 进 行 删 除、 复 制、 贴 上 等 等 的 动 作， 但 是 却 无 法 编 辑 文 件 内 容 的！ 要 等 到 你 按 下“ i, I, o, O, a, A, r, R” 等 任 何 一 个 字 母 之 后 才 会 进 入 编 辑 模 式。 注 意 了！ 通 常 在 Linux 中， 按 下 这 些 按 键 时， 在 画 面 的 左 下 方 会 出 现“ INSERT 或 REPLACE ”的 字 样， 此 时 才 可 以 进 行 编 辑。 而 如 果 要 回 到 一 般 指 令 模 式 时， 则 必 须 要 按 下“ Esc” 这 个 按 键 即 可 退 出 编 辑 模 式。



#### 1.2.3 命令行命令模式

在 一 般 模 式 当 中， 输 入“ : / ? ”三 个 中 的 任 何 一 个 按 钮， 就 可 以 将 光 标 移 动 到 最 下 面 那 一 列。 在 这 个 模 式 当 中， 可 以 提 供 你“ 搜 寻 数 据” 的 动 作， 而 读 取、 存 盘、 大 量 取 代 字 符、 离 开 vi 、显 示 行 号 等 等 的 动 作 则 是 在 此 模 式 中 达 成 的。

![image-20200831142745294](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20200831142745294.png)



## 2.按键说明

vi 的 三 种 模 式 只 有 一 般 指 令 模 式 可 以 与 编 辑、 命 令 行 界 面 切 换， 编 辑 模 式 与 命 令 行 界 面 之 间 并 不 能 切 换 的。

### 2.1 一般指令模式可用的按键说明

![image-20200831143042639](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20200831143042639.png)

![image-20200831143056704](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20200831143056704.png)

![image-20200831143119355](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20200831143119355.png)

![image-20200831143136370](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20200831143136370.png)

![image-20200831143152962](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20200831143152962.png)



### 2.2 一 般 指 令 模 式 切 换 到 编 辑 模 式 的 可 用 的 按 钮 说 明

![image-20200831143257806](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20200831143257806.png)



### 2.3 一 般 指 令 模 式 切 换 到 命 令 行 界 面 的 可 用 按 钮 说 明

![image-20200831143333416](../../../AppData/Roaming/Typora/typora-user-images/image-20200831143333416.png)

![image-20200831143344650](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20200831143344650.png)

![image-20200831143355857](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20200831143355857.png)