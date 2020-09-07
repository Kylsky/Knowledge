## 1.关于变量

### 1.1 设置变量

name=kyle

### 1.2 输出变量

echo $name

### 1.3 自定变量转环境变量

export name

### 1.4 删除变量

unset name

### 1.5 查看环境变量

env

![image-20200901092836593](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20200901092836593.png)

### 1.6 查看所有变量

set

### 1.7 查看语系

locale

### 1.8 读取键盘输入作为变量

read 

```
read [-pt] 变量名
```

-p ：后 面 可 以 接 提 示 字 符

-t ：后 面 可 以 接 等 待 的“ 秒 数！” 

```
read -p "Please enter a key" name
```

### 1.9 声明并设定变量类型

declare / typeset

```
declare [-aixr] 变量名
```

**-a** ：将 后 面 名 为 variable 的 变 量 定 义 成 为 阵 列 （array） 类 型 

**-i** ：将 后 面 名 为 variable 的 变 量 定 义 成 为 整 数 数 字 （integer） 类 型 

**-x** ：用 法 与 export 一 样， 就 是 将 后 面 的 variable 变 成 环 境 变 量； 

**-r** ：将 变 量 设 置 成 为 readonly 类 型， 该 变 量 不 可 被 更 改 内 容， 也 不 能 unset

```
declare -i sum=100+300+50
```

### 1.10 文件系统与程序限制

ulimit

```
ulimit [-SHacdfltu] [配 额]
```

-H ：hard limit ，严 格 的 设 置， 必 定 不 能 超 过 这 个 设 置 的 数 值； 

-S ：soft limit ，警 告 的 设 置， 可 以 超 过 这 个 设 置 值， 但 是 若 超 过 则 有 警 告 讯 息。 在 设 置 上， 通 常 soft 会 比 hard 小， 举 例 来 说， soft 可 设 置 为 80 而 hard 设 置 为 100， 那 么 你 可 以 使 用 到 90 （因 为 没 有 超 过 100）， 但 介 于 80 ~ 100 之 间 时， 系 统 会 有 警 告 讯 息 通 知 

-a ：后 面 不 接 任 何 选 项 与 参 数， 可 列 出 所 有 的 限 制 额 度； 

-c ：当 某 些 程 序 发 生 错 误 时， 系 统 可 能 会 将 该 程 序 在 内 存 中 的 信 息 写 成 文 件（ 除 错 用）， 这 种 文 件 就 被 称 为 核 心 文 件（ core file）。 此 为 限 制 每 个 核 心 文 件 的 最 大 容 量。 

-f ：此 shell 可 以 创 建 的 最 大 文 件 大 小（ 一 般 可 能 设 置 为 2GB） 单 位 为 KBytes 

-d ：程 序 可 使 用 的 最 大 断 裂 内 存（ segment） 容 量； 

-l ：可 用 于 锁 定 （lock） 的 内 存 量 

-t ：可 使 用 的 最 大 CPU 时 间 （单 位 为 秒） 

-u ：单 一 使 用 者 可 以 使 用 的 最 大 程 序（ process） 数 量

### 1.11 变量内容的删除

echo ${PATH#/*:}

删除匹配的最短信息



echo ${PATH##/*}

删除匹配的最长信息



echo ${PATH%:*bin}

删除最后出现 **:*bin** 的内容



echo ${PATH%%:*bin}

删除符合标准的最长内容



### 1.12 历史命令

history

```
history [n] 
history [-c] 
history [-raw] histfiles
```

**n** ：数 字， 意 思 是“ 要 列 出 最 近 的 n 笔 命 令 列 表” 的 意 思！ 

**-c** ：将 目 前 的 shell 中 的 所 有 history 内 容 全 部 消 除 

**-a** ：将 目 前 新 增 的 history 指 令 新 增 入 histfiles 中， 若 没 有 加 histfiles ， 则 默 认 写 入 ~/. bash_history 

**-r** ：将 histfiles 的 内 容 读 到 目 前 这 个 shell 的 history 记 忆 中； 

**-w** ：将 目 前 的 history 记 忆 内 容 写 入 histfiles 中



## 2.Bash环境配置文件

### 2.1 login与non-login shell

为 什 么 要 介 绍 login, non-login shell 呢？ 这 是 因 为 这 两 个 取 得 bash 的 情 况 中， 读 取 的 配 置 文 件 数 据 并 不 一 样 所 致。

**login shell**： 取 得 bash 时 需 要 完 整 的 登 陆 流 程 的， 就 称 为 login shell。 举 例 来 说， 你 要 由 tty1 ~ tty6 登 陆， 需 要 输 入 使 用 者 的 帐 号 与 密 码， 此 时 取 得 的 bash 就 称 为“ login shell ”啰； login shell 其 实 只 会 读 取 这 两 个 配 置 文 件：

1./etc/profile；

2.~/. bash_profile 或 ~/. bash_login 或 ~/. profile



**non-login shell**： 取 得 bash 接 口 的 方 法 不 需 要 重 复 登 陆 的 举 动， 举 例 来 说，（ 1） 你 以 X window 登 陆 Linux 后， 再 以 X 的 图 形 化 接 口 启 动 终 端 机， 此 时 那 个 终 端 接 口 并 没 有 需 要 再 次 的 输 入 帐 号 与 密 码， 那 个 bash 的 环 境 就 称 为 non-login shell 了。（ 2） 你 在 原 本 的 bash 环 境 下 再 次 下 达 bash 这 个 指 令， 同 样 的 也 没 有 输 入 帐 号 密 码， 那 第 二 个 bash （子 程 序） 也 是 non-login shell



### 2.2 /etc/ profile （login shell 才 会 读）

这 个 文 件 设 置 的 变 量 主 要 有：

**PATH**： 会 依 据 UID 决 定 PATH 变 量 要 不 要 含 有 sbin 的 系 统 指 令 目 录； 

**MAIL**： 依 据 帐 号 设 置 好 使 用 者 的 mailbox 到 /var/ spool/ mail/ 帐 号 名； 

**USER**： 根 据 使 用 者 的 帐 号 设 置 此 一 变 量 内 容； 

**HOSTNAME**： 依 据 主 机 的 hostname 指 令 决 定 此 一 变 量 内 容； 

**HISTSIZE**： 历 史 命 令 记 录 笔 数。 CentOS 7. x 设 置 为 1000 ； 

**umask**： 包 括 root 默 认 为 022 而 一 般 用 户 为 002 等

bash 的 login shell 情 况 下 所 读 取 的 整 体 环 境 配 置 文 件 其 实 只 有 /etc/ profile， 但 是 /etc/ profile 还 会 调 用 出 其 他 的 配 置 文 件， 所 以 让 我 们 的 bash 操 作 接 口 变 的 非 常 的 友 善



### 2.3 ~/. bash_profile

bash 在 读 完 了 整 体 环 境 设 置 的 /etc/ profile 并 借 此 调 用 其 他 配 置 文 件 后， 接 下 来 则 是 会 读 取 使 用 者 的 个 人 配 置 文 件。 在 login shell 的 bash 环 境 中， 所 读 取 的 个 人 偏 好 配 置 文 件 其 实 主 要 有 三 个， 依 序 分 别 是：

~/. bash_profile 

~/. bash_login 

~/. profile

只有才前一个文件不存在时，才回去读取下一个文件

![image-20200901160953700](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20200901160953700.png)

这 个 文 件 内 有 设 置 PATH 这 个 变 量 喔！ 而 且 还 使 用 了 export 将 PATH 变 成 环 境 变 量 。 由 于 PATH 在 /etc/ profile 当 中 已 经 设 置 过， 所 以 在 这 里 就 以 累 加 的 方 式 增 加 使 用 者 主 文 件 夹 下 的 ~/ bin/ 为 额 外 的 可 执 行 文 件 放 置 目 录。 这 也 就 是 说， **你 可 以 将 自 己 创 建 的 可 执 行 文 件 放 置 到 你 自 己 主 文 件 夹 下 的 ~/ bin/ 目 录** ， 那 就 可 以 直 接 执 行 该 可 执 行 文 件 而 不 需 要 使 用 绝 对/ 相 对 路 径 来 执 行 该 文 件。

![image-20200901161535433](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20200901161535433.png)



### 2.4 source ：读 入 环 境 配 置 文 件 的 指 令

利 用 source 或 小 数 点 （.） 都 可 以 将 配 置 文 件 的 内 容 读 进 来 目 前 的 shell 环 境 中。 举 例 来 说， 我 修 改 了 ~/. bashrc ，那 么 不 需 要 登 出， 立 即 以 source ~/. bashrc 就 可 以 将 刚 刚 最 新 设 置 的 内 容 读 进 来 目 前 的 环 境 中！ 很 不 错 吧！ 还 有， 包 括 ~/ bash_profile 以 及 /etc/ profile 的 设 置 中， 很 多 时 候 也 都 是 利 用 到 这 个 source （或 小 数 点） 的 功 能 。



### 2.5 ~/. bashrc （non-login shell 会 读）

当 你 取 得 non-login shell 时， 该 bash 配 置 文 件 仅 会 读 取 ~/. bashrc 而 已，以下是bashrc的内容

![image-20200901162437329](../../../AppData/Roaming/Typora/typora-user-images/image-20200901162437329.png)

为 什 么 需 要 调 用 /etc/ bashrc 呢？ 因 为 /etc/ bashrc 帮 我 们 的 bash 定 义 出 下 面 的 数 据：

1.依 据 不 同 的 UID 规 范 出 umask 的 值

2.依 据 不 同 的 UID 规 范 出 提 示 字 符 （就 是 PS1 变 量）

3.调 用 /etc/ profile.d/*. sh 的 设 置



## 3.组合键

![image-20200907143044820](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20200907143044820.png)

![image-20200907143057143](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20200907143057143.png)



## 4.万用字符和特殊符号

### 4.1 万用字符

![image-20200907143249436](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20200907143249436.png)

![image-20200907143306276](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20200907143306276.png)



### 4.2 特殊符号

![image-20200907143354273](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20200907143354273.png)

![image-20200907143409914](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20200907143409914.png)



### 5.数据流重导向

标 准 输 入 　 　（ stdin） ：代 码 为 0 ，使 用 < 或 < < 

标 准 输 出 　 　（ stdout）： 代 码 为 1 ，使 用 > 或 > > 

标 准 错 误 输 出（ stderr）： 代 码 为 2 ，使 用 2 > 或 2 > > 

#### 5.1 标准输入

```
#将正确的数据写入文件
find /home --name .bashrc >right.txt
#将错误的数据写入文件
find /home --name .bashrc 2>error.txt
#同时写入错误和正确的数据
find /home --name .bashrc >right.txt 2>error.txt
#正确和错误的数据写入同一个文件
find /home --name .bashrc &>right.txt
```



#### 5.2 标准输出

```
#用 cat 直 接 将 输 入 的 讯 息 输 出 到 catfile 中， 且 当 由 键 盘 输 入 eof 时， 该 次 输 入 就 结 束”
cat > catfile < < "eof"

#用 stdin 取 代 键 盘 的 输 入 以 创 建 新 文 件 的 简 单 流 程
cat > catfile < ~/. bashrc
```



#### 5.3 命令执行判断依据

在 某 些 情 况 下， 很 多 指 令 我 想 要 一 次 输 入 去 执 行， 而 不 想 要 分 次 执 行 时，  基 本 上 有 两 个 选 择， 一 个 是 通 过 shell script 撰 写 脚 本 去 执 行， 一 种 则 是 通 过 下 面 的 介 绍 来 一 次 输 入 多 重 指 令、

##### ；

在 指 令 与 指 令 中 间 利 用 分 号 （;） 来 隔 开， 这 样 一 来， 分 号 前 的 指 令 执 行 完 后 就 会 立 刻 接 着 执 行 后 面 的 指 令 了。

##### &&和||

在 指 令 与 指 令 中 间 利 用 && 或||来 隔 开， 这 样 一 来， 分 号 前 的 指 令 执 行 完 后 就 会 根 据 前 面 执 行 代 码 是 否 正 确 来 判 断 是 不 是 需 要 执 行 后 续 的 命 令。



#### 5.4 管线命令

处理示意图：

![image-20200907161545248](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20200907161545248.png)

##### cut

cut 主 要 的 用 途 在 于 将“ 同 一 行 里 面 的 数 据 进 行 分 解” 最 常 使 用 在 分 析 一 些 数 据 或 文 字 数 据 的 时 候，这 是 因 为 有 时 候 我 们 会 以 某 些 字 符 当 作 分 区 的 参 数， 然 后 来 将 数 据 加 以 切 割， 以 取 得 我 们 所 需 要 的 数 据。

```
#用 于 有 特 定 分 隔 字 符 
cut -d' 分 隔 字 符' -f fields 

#用 于 排 列 整 齐 的 讯 息
cut -c 字 符 区 间 
```

**-d** ：后 面 接 分 隔 字 符。 与 -f 一 起 使 用； 

**-f** ：依 据 -d 的 分 隔 字 符 将 一 段 讯 息 分 区 成 为 数 段， 用 -f 取 出 第 几 段 的 意 思； 

**-c** ：以 字 符 （characters） 的 单 位 取 出 固 定 字 符 区 间



示例：

```
echo ${PATH} | cut -d ':' -f 5
export | cut -c 12-
```



##### grep

cut 是 将 一 行 讯 息 当 中， 取 出 某 部 分 我 们 想 要 的， 而 grep 则 是 分 析 一 行 讯 息， 若 当 中 有 我 们 所 需 要 的 信 息，就 将 该 行 拿 出 来

```
grep [-acinv] [--color = auto] '搜 寻 字 串' filename 
```

**-a** ：将 binary 文 件 以 text 文 件 的 方 式 搜 寻 数 据 

**-c** ：计 算 找 到 '搜 寻 字 串' 的 次 数 

**-i** ：忽 略 大 小 写 的 不 同， 所 以 大 小 写 视 为 相 同 

**-n** ：顺 便 输 出 行 号 

**-v** ：反 向 选 择， 亦 即 显 示 出 没 有 '搜 寻 字 串' 内 容 的 那 一 行