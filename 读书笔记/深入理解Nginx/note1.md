# 1.前提要求

相关参考

```
http://wiki.nginx.org/Modules
http://wiki.nginx.org/3rdPartyModules
```

## 1.1 Linux内核

需要linux内核版本为2.6及以上

```
uname -a
```



## 1.2 GCC编译器

```
yum install -y gcc-c++
```



## 1.3  PCRE

主要是提供了正则表达式，如果保证用不到，可以不装，但是这种情况比较少见

```
yum install -y pcre pcre-devel
```



## 1.4 zlib

zlib 库 用 于 对 HTTP 包 的 内 容 做 gzip 格 式 的 压 缩， 如 果 我 们 在 nginx.conf 里 配 置 了 gzip on， 并 指 定 对 于 某 些 类 型（ content-type） 的 HTTP 响 应 使 用 gzip 来 进 行 压 缩 以 减 少 网 络 传 输 量， 那 么， 在 编 译 时 就 必 须 把 zlib 编 译 进 Nginx。

```
#zlib 是直接使用的库，zlib-devel是二次开发所需要的库
yum install -y zlib zlib-devel
```



## 1.5 OpenSSL

用于支持https

```
yun install -y openssl openssl-devel
```



# 2.Linux内核参数优化

由 于 默 认 的 Linux 内 核 参 数 考 虑 的 是 最 通 用 的 场 景， 不 符 合 用 于 支 持 高 并 发 访 问 的 Web 服 务 器 的 定 义， 所 以 需 要 修 改 Linux 内 核 参 数， 使 得 Nginx 可 以 拥 有 更 高 的 性 能。 

在 优 化 内 核 时，通 常 会 根 据 业 务 特 点 来 进 行 调 整， 当 Nginx 作 为 静 态 Web 内 容 服 务 器、 反 向 代 理 服 务 器 或 是 提 供 图 片 缩 略 图 功 能（ 实 时 压 缩 图 片） 的 服 务 器 时， 其 内 核 参 数 的 调 整 都 是 不 同 的。 这 里 只 针 对 最 通 用 的、 使 Nginx 支 持 更 多 并 发 请 求 的 TCP 网 络 参 数 做 简 单 说 明。

使用vi /etc/systcl.conf更改内核参数：

```
fs.file-max = 999999 							#进程能打开的最大句柄数
net.ipv4. tcp_tw_reuse = 1						#设置为1，允许讲timewait状态的socket重新用于新的tcp连接
net.ipv4. tcp_keepalive_time = 600 				 #当keepalive启用时，tcp发送keepalive消息的频率，缩小频率能够更快清理无效的连接
net.ipv4. tcp_fin_timeout = 30 					 #当服务器主动关闭连接时，socket保持在fin-wait-2的最大时间
net.ipv4. tcp_max_tw_buckets = 5000 			 #处于time-wait状态的套接字的最大值，配置过大会导致web服务器变慢
net.ipv4. ip_local_port_range = 1024 61000 		  #udp和tcp连接中本地端口的范围
net.ipv4. tcp_rmem = 4096 32768 262142 			  #tcp接收缓存的最小值、默认值、最大值
net.ipv4. tcp_wmem = 4096 32768 262142 			  #tcp发送缓存的最小值、默认值、最大值
net.core.netdev_max_backlog = 8096 				 
net.core.rmem_default = 262144 
net.core.wmem_default = 262144 
net.core.rmem_max = 2097152 
net.core.wmem_max = 2097152 
net.ipv4. tcp_syncookies = 1 
net.ipv4. tcp_max_syn.backlog = 1024
```

执行sysctl-p使上述修改生效。



# 3.Nginx命令行控制

## 3.1 默认方式启动

```
#默认读取的时/usr/local/nginx/conf/nginx.conf
/usr/local/nginx/sbin/nginx
```



## 3.2 另行指定配置文件启动

```
/usr/local/nginx/sbin/nginx -c /root/nginx.conf
```



## 3.3 另行指定安装目录的启动方式

```
/usr/local/nginx/sbin/nginx -p /usr/local/nginx/
```



## 3.4 另行指定全局配置项的启动方式

通过-g参数临时指定一些全局配置项

```
/usr/local/nginx/sbin/nginx -g "pid /var/nginx/test.pid"
```

* 注 意，另 一 个 约 束 条 件 是， 以-g 方 式 启 动 的 Nginx 服 务 执 行 其 他 命 令 行 时， 需 要 把-g 参 数 也 带 上， 否 则 可 能 出 现 配 置 项 不 匹 配 的 情 形。



## 3.5 测试配置项是否存在问题

```
/usr/local/nginx/sbin/nginx -t 
```



## 3.6 在测试配置阶段不输出信息

过滤error级别以下的信息

```
/usr/local/nginx/sbin/nginx -t -q
```



## 3.7 显示版本信息

```
/usr/local/nginx/sbin/nginx -v
```



## 3.8 显示编译阶段参数

```
/usr/local/nginx/sbin/nginx -V
```



## 3.9 快速停止服务

强制停止nginx服务，nginx程序通过nginx。pid中得到的master进程id，再向运行中的master进程发送term信号来快速地关闭nginx服务

```
/usr/local/nginx/sbin/nginx -s stop
```



## 3.10优雅停止服务

与强制退出不同，首先会关闭监听端口，接收新的连接，然后把正在处理地连接全部处理玩，再退出进程

```
/usr/local/nginx/sbin/nginx -s quit
```



## 3.11 重载配置项并生效

```
/usr/local/nginx/sbin/nginx -s reload
```



# 4.Nginx的配置

## 4.1 Nginx进程间关系

在 正 式 提 供 服 务 的 产 品 环 境 下， 部 署 Nginx 时 都 是 使 用 一 个 master 进 程 来 管 理 多 个 worker 进 程， 一 般 情 况 下， worker 进 程 的 数 量 与 服 务 器 上 的 CPU 核 心 数 相 等。 每 一 个 worker 进 程 都 是 繁 忙 的， 它 们 在 真 正 地 提 供 互 联 网 服 务， master 进 程 则 很“ 清 闲”， 只 负 责 监 控 管 理 worker 进 程。 worker 进 程 之 间 通 过 **共 享 内 存、 原 子 操 作** 等 一 些 进 程 间 通 信 机 制 来 实 现 负 载 均 衡 等 功 能。

<img src="http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20210226162712583.png" alt="image-20210226162712583" style="zoom:50%;" />



## 4.2 配置通用语法

```nginx
user nobody; 
worker_processes 8; 
error_log /var/ log/ nginx/ error.log error; 
#pid logs/ nginx.pid; 
events { 
    use epoll; 
    worker_connections 50000; 
} 
http { 
    include mime.types; 
    default_type application/ octet-stream; 
    log_format main '$ remote_addr [$ time_local] "$ request" ' 
        '$ status $ bytes_sent "$ http_referer" ' 
        '" $ http_user_agent" "$ http_x_forwarded_for"'; 
    access_log logs/ access.log main buffer = 32k;  
｝
```



## 4.3 基本配置

### 4.3.1 调试、定位问题

* 守护进程方式运行 **daemon on|off**;(默认on)
* master/worker方式工作 **master_process on|off**;（默认on）
* error日志设置 **error_log /${path}/${file} ${level}**(默认error_log logs/error.log error)
* 是否处理特殊的调试点 **debug_points[stop|abort]** (用于调试nginx，通常不使用)
* 仅对指定的客户端输出debug级别日志 **debug_connections[IP|CIDR]** (放置在events{}中)
* 限制coredump核心转储文件大小 **wordker_rlimit_core ${size}** (用于nginx非法操作的定位)
* coredump文件所放置的目录 **wordking_directory ${path}** 



### 4.3.2 正常运行

* 定义环境变量 **env VAR|VAR = VALUE**
* 嵌入其他配置文件 **include /${path}/${file}** (如：include conf.d/*.conf;)
* pid文件路径 **pid ${path}/${file}**
* worker进程运行额用户及用户组 **user username[groupname]** (默认user nobody nobody)
* 设置可以打开的最大句柄数 **worker_rlimit_nofile ${limit}**
* 限制信号队列 **worker_rlimit_sigpending ${limit}**



### 4.3.3 优化性能

* worker进程个数 **worker_process ${number}** (可以和worker_cpu_affinity 一起使用)
* 绑定nginx worker进程到指定的cpu内核 (如： **worker_cpu_affinity 1000 0100 0010 0001**)
* ssl硬件加速 **ssl_engine ${device}** (通过openssl engine -t查询)
* 时钟更新频率 **timer_resolution ${time}**
* nginx worker优先级设置 **worker_priority ${nice}** (默认0，优先级越高，分配到cpu时间片越大)



### 4.3.4 事件类

* 负载均衡锁 **accept_mutext [on|off]** (默认on，关闭后会导致负载不均衡，但是tcp连接建立时间会变短)
* 负载均衡锁文件路径 **lock_file ${path}/${file}** (默认lock_file logs/nginx.lock，锁关闭时该配置不生效，开启时若不支持原子锁，该配置生效)
* 使用accept锁后到真正建立连接之间的延迟时间 **accept_mutex_delay ${time}ms** (默认500ms)
* 批量建立新连接 **multi_accept[on|off]** (默认off，当事件模型通知有新连接，不会对所有请求都建立连接)
* 选择事件模型 **use[ kqueue | rtsig | epoll |/ dev/ poll | select | poll | eventport]**(默认自动选择)
* worker最大连接数 **worker_connections ${number}**



# 5.Nginx配置静态web服务器

所 有 的 HTTP 配 置 项 都 必 须 直 属 于 http 块、 server 块、 location 块、 upstream 块 或 if 块 等（ HTTP 配 置 项 自 然 必 须 全 部 在 http{} 块 之 内， 这 里 的“ 直 属 于” 是 指 配 置 项 直 接 所 属 的 大 括 号 对 应 的 配 置 块）， 同 时， 在 描 述 每 个 配 置 项 的 功 能 时， 会 说 明 它 可 以 在 上 述 的 哪 个 块 中 存 在， 因 为 有 些 配 置 项 可 以 任 意 地 出 现 在 某 一 个 块 中， 而 有 些 配 置 项 只 能 出 现 在 特 定 的 块 中



## 5.1 虚拟主机与请求分发

### 5.1.1 监听端口

**语法：**

```nginx
listen address:port[ default( deprecated in 0.8.21) | default_server |[ backlog = num | rcvbuf = size | sndbuf = size | accept_filter = filter | deferred | bind | ipv6only =[ on | off] | ssl]];

示例：
listen 127.0.0.1：8080;
#不加端口默认为80
listen 127.0.0.1;	
listen 8000;
listen *:8000;
listen localhost:8000;
#IPv6
listen [::]:8000;
listen [fe80::1];
listen [:::a8c9:1234]:80;
#其他参数
listen 443 default_server ssl;
listen 127.0.0.1 default_server accept_filter=dataready backlog=1024;
```

**默认：**

listen 80；

**配置块：**

server

**参数：**

* default

  将所在server作为web服务的默认server块

* default_server

  同上

* backlog=${num}

  表示tcp中backlog队列的大小

* bind 

  绑定当前端口/地址对，只有同事对一个端口监听多个地址时才会生效，即多个location

* ssl

  在当前监听的端口上建立的连接必须基于ssl协议



### 5.1.2 主机名称

**语法：**

server_name name[…]

**默认：**

server_name ""；

**配置块：**

server

**说明：**

Nginx 正 是 使 用 server_name 配 置 项 针 对 特 定 Host 域 名 的 请 求 提 供 不 同 的 服 务， 以 此 实 现 虚 拟 主 机 功 能。在开始处理一个HTTP请求时，Nginx会取出hearder头中的host，与每个server中的server_name进行匹配，一次决定到底由哪一个server块来处理。



### 5.1.3 server_names_bucket_size

**语法：**

server_names_bucket_size ${size}

**默认：**

server_names_bucket_size 32|64|128

**配置块：**

http、server、location

**说明：**

为了提高快速寻找到响应server name的能力，nginx使用散列表来存储server name。该参数设置了每个散列桶占用的内存大小



### 5.1.4 server_names_has_max_size

**语法：**

server_names_hash_max_size ${size}

**默认：**

server_names_hash_max_size 512;

**配置块：**

http、server、location

**说明：**

该参数会影响散列表的冲突率，size越大，消耗的内存越多，但散列key的冲突会降低。



### 5.1.5 重定向主机名称处理

**语法：**

server_names_in_redirect on|off

**默认：**

server_names_in_redirect on

**配置块：**

http、server、location

**说明：**

该 配 置 需 要 配 合 server_name 使 用。 在 使 用 on 打 开 时， 表 示 在 重 定 向 请 求 时 会 使 用 server_name 里 配 置 的 第 一 个 主 机 名 代 替 原 先 请 求 中 的 Host 头 部， 而 使 用 off 关 闭 时， 表 示 在 重 定 向 请 求 时 使 用 请 求 本 身 的 Host 头 部。



### 5.1.6 location

**语法：**

location[=|~|~*|^~|@] /uri/{...}

**配置块：**

server

**说明：**

location会尝试根据用户请求的uri匹配，若成功，就选择该块处理用户请求

```nginx
1) =表示把uri作为字符串，以便与参数中的uri做完全匹配
location = /{
	#只有当用户请求是/时，才会使用该location
}

2) ~ 表示匹配时对字母大小写敏感

3) ~* 表示忽略字母大小写问题

4) ^~表示匹配uri时只需要其前半部分与uri参数匹配即可
location ^~ /images/ {
	#以/images/开始的请求都会匹配上
}

5) @表示仅用于nginx服务内部请求之间的重定向，带有@的location不直接处理用户请求

6) 匹配所有http请求
location / {
	# /可以匹配所有请求
}
```



## 5.2 文件路径的定义

### 5.2.1 以root方式涉及资源路径

**语法：**

root ${path}

**默认：**

root html;

**配置块：**

http、server、location、if

**示例：**

```nginx
location /download/ {
	root /opt/web/html;
}
```



### 5.2.2 以alias方式设置资源路径

**语法：**

alias ${path}

**配置块：**

location

**说明：**

root会将完整的uri请求映射，而alias会将location后配置的路径丢弃，以下是两个等价的配置：

<img src="http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20210303085530120.png" alt="image-20210303085530120" style="zoom: 67%;" />

alias还支持正则表达式，如：

<img src="http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20210303085736827.png" alt="image-20210303085736827" style="zoom:80%;" />



### 5.2.3 访问首页

**语法：**

index ${index}

**默认：**

index index.html

**配置块：**

http、server、location

**说明：**

index后可以跟多个文件参数，nginx将会按照顺序来访问这些文件



### 5.2.4 根据http返回码重定向页面

**语法：**

error_page code[...] [=|=answer-code] uri|@named_location

**配置块：**

http、server、location、if

**示例：**

```nginx
#匹配单个
error_page 404 /404.html;
#匹配多个
error_page 502 503 504 /50x.html;
#更改错误码
error_page 404 =20 /empty.gif;
#请求重定向
location / {
	error_page 404 @fallback;
}
location @fallback {
	proxy_pass http://backend;
}
```



### 5.2.5 try_files

**语法：**

try_files path1[path2] uri

**配置块：**

server、location

**说明：**

try_files 后 要 跟 若 干 路 径， 如 path1 path2...， 而 且 最 后 必 须 要 有 uri 参 数， 意 义 如 下： 尝 试 按 照 顺 序 访 问 每 一 个 path， 如 果 可 以 有 效 地 读 取， 就 直 接 向 用 户 返 回 这 个 path 对 应 的 文 件 结 束 请 求， 否 则 继 续 向 下 访 问。 如 果 所 有 的 path 都 找 不 到 有 效 的 文 件， 就 重 定 向 到 最 后 的 参 数 uri 上。 因 此， 最 后 这 个 参 数 uri 必 须 存 在， 而 且 它 应 该 是 可 以 有 效 重 定 向 的。 例 如：

```nginx
try_files /system/ maintenance.html $ uri $ uri/ index.html $ uri.html @other; 
location @other {
	proxy_pass http:// backend; 
}
```



## 5.3 内存及磁盘资源的分配

（ps:由于这部分内容对我而言不太常用，因此在这儿只放了一点）

### 5.3.1  http包体只存储到磁盘文件中

**语法：**

client_body_in_file_only on|clean|off

**默认：**

client_body_in_file_only off;

**配置块：**

http、server、location

**说明：**

当 值 为 非 off 时， 用 户 请 求 中 的 HTTP 包 体 一 律 存 储 到 磁 盘 文 件 中， 即 使 只 有 0 字 节 也 会 存 储 为 文 件。 当 请 求 结 束 时， 如 果 配 置 为 on， 则 这 个 文 件 不 会 被 删 除（ 该 配 置 一 般 用 于 调 试、 定 位 问 题）， 但 如 果 配 置 为 clean， 则 会 删 除 该 文 件。



### 5.3.2  connection_pool_size

**语法：**

connection_pool_size ${size}

**默认：**

connection_pool_size 256;

**配置块：**

http、server

**说明：**

Nginx 对 于 每 个 建 立 成 功 的 TCP 连 接 会 预 先 分 配 一 个 内 存 池， 上 面 的 size 配 置 项 将 指 定 这 个 内 存 池 的 初 始 大 小， 用 于 减 少 内 核 对 于 小 块 内 存 的 分 配 次 数。 需 慎 重 设 置， 因 为 更 大 的 size 会 使 服 务 器 消 耗 的 内 存 增 多， 而 更 小 的 size 则 会 引 发 更 多 的 内 存 分 配 次 数



## 5.4 网络连接设置

### 5.4.1 读取http头部的超时时间

**语法：**

client_header_timeout ${time}(默认单位：秒)

**默认：**

client_header_timeout 60

**配置块：**

http、server、location

**说明：**

客 户 端 与 服 务 器 建 立 连 接 后 将 开 始 接 收 HTTP 头 部， 在 这 个 过 程 中， 如 果 在 一 个 时 间 间 隔（ 超 时 时 间） 内 没 有 读 取 到 客 户 端 发 来 的 字 节， 则 认 为 超 时， 并 向 客 户 端 返 回 408(" Request timed out") 响 应。



### 5.4.2 读取http包体的超时时间

**语法：**

client_body_timeout ${time}(默认单位：秒)

**默认：**

client_body_timeout 60

**配置块：**

http、server、location

**说明：**

与上相似



### 5.4.3 发送响应的超时时间 

**语法：** 

send_timeout time; 

**默认：**

 send_timeout 60; 

**配置块：** 

http、 server、 location



### 5.4.4 keepalive禁用

**语法：**

keepalive_diable [msie6|safari|none]

**默认：**

keepalive_disable msie6 safari;

**配置块：**

http、server、location

**说明：**

 HTTP 请 求 中 的 keepalive 功 能 是 为 了 让 多 个 请 求 复 用 一 个 HTTP 长 连 接， 这 个 功 能 对 服 务 器 的 性 能 提 高 是 很 有 帮 助 的。 但 有 些 浏 览 器， 如 IE 6 和 Safari， 它 们 对 于 使 用 keepalive 功 能 的 POST 请 求 处 理 有 功 能 性 问 题。 因 此， 针 对 IE 6 及 其 早 期 版 本、 Safari 浏 览 器 默 认 是 禁 用 keepalive 功 能 的。



### 5.4.5 keepalive超时时间

**语法：**

keepalive_timeout ${time}(默认单位：秒)

**默认：**

keepalive_timeout 75;

**配置块：**

http、server、location



### 5.4.6 一个keepalive长连接上允许承载的请求最大数

**语法：**

keepalive_request ${number}

**默认：**

keepalive_request 100;

**配置块：**

http、server、location



## 5.5 MIME类型的设置

MIME：Multipurpose Internet Mail Extensions，多用途互联网邮件扩展类型

### 5.5.1 MIME type与文件扩展映射

**语法：**

type {...}

**配置块：**

http、server、location

**示例：**

```
types {
	text/html html;
	text/html conf;
	image/gif gif;
	image/jpeg jpg;
}
```



### 5.5.2 默认MIME type

**语法：**

default_type ${type}

**默认：**

default_type text/plain

**说明：**

若找不到相应的MIME type与文件扩展名之间的映射，则使用默认的MIME typ而作为Content-Type。



## 5.6 对客户端请求的限制

### 5.6.1 limit_except method ...[...]

**配置块：**

location

**说明：**

通过limt_except后面指定的方法名来限制用户请求。方法名包括GET/HEAD/POST/PUT/DELETE/MKCOL/COPY/MOVE/OPTIONS/PROPFIND/PROPPATCH/LOCK/UNLOCK/PATCH

```
#禁止GET和HEAD方法
location / {
	limit_except GET {
		allow 192.168.1.0/32
		deny all;
	}
}
```



### 5.6.2 http请求包体最大值

**语法：**

client_max_body_size ${size}

**默认：**

client_max_body_size 1m;

**配置块：**

http、server、location

**说明：**

浏 览 器 在 发 送 含 有 较 大 HTTP 包 体 的 请 求 时， 其 头 部 会 有 一 个 Content-Length 字 段， client_max_body_size 是 用 来 限 制 Content-Length 所 示 值 的 大 小 的。受。 例 如， 用 户 试 图 上 传 一 个 10GB 的 文 件， Nginx 在 收 完 包 头 后， 发 现 Content-Length 超 过 client_max_body_size 定 义 的 值， 就 直 接 发 送 413(" Request Entity Too Large") 响 应 给 客 户 端。



### 5.6.2 对请求的限速

**语法：**

limit_rate ${speed}

**默认：**

limit_rate 0;

**配置块：**

http、server、location、if

**说明：**

限制每秒传输的字节数，0表示不限速

针 对 不 同 的 客 户 端， 可 以 用 $ limit_rate 参 数 执 行 不 同 的 限 速 策 略。 例 如：

```nginx
server {
	if($slow){
		set $limit_rate 4k;
	}
}
```



### 5.6.2 limit_rate_after

**语法：**

limit_rate_after ${time}

**默认：**

limit_rate_after 1m;

**配置块：**

http、server、location、if

**说明：**

此 配 置 表 示 Nginx 向 客 户 端 发 送 的 响 应 长 度 超 过 limit_rate_after 后 才 开 始 限 速。 



## 5.7 文件操作优化

### 5.7.1  sendfile系统调用

**语法：**

sendfile on|off

**默认：**

sendfile off;

**配置块：**

http、server、location

**说明：**

可 以 启 用 Linux 上 的 sendfile 系 统 调 用 来 发 送 文 件， 它 减 少 了 内 核 态 与 用 户 态 之 间 的 两 次 内 存 复 制， 这 样 就 会 从 磁 盘 中 读 取 文 件 后 直 接 在 内 核 态 发 送 到 网 卡 设 备， 提 高 了 发 送 文 件 的 效 率。



### 5.7.2 AIO系统调用

**语法：**

aio on|off

**默认：**

aio off;

**配置块：**

http、server、location

**说明：**

此配置项表示是否启用异步文件IO系统，该功能与sendfile互斥



## 5.8 对客户端请求的特殊处理

### 5.8.1 忽略不合法http头部

**语法：**

ignore_invalie_headers on|off

**默认：**

ignore_invalie_headers on

**配置块：**

http、server

**说明：**

如 果 将 其 设 置 为 off， 那 么 当 出 现 不 合 法 的 HTTP 头 部 时， Nginx 会 拒 绝 服 务， 并 直 接 向 用 户 发 送 400（ Bad Request） 错 误。 如 果 将 其 设 置 为 on， 则 会 忽 略 此 HTTP 头 部。



### 5.8.1 http头部是否允许下划线

**语法：**

underscores_in_headers on|off

**默认：**

underscores_in_headers off

**配置块：**

http、server

**说明：**

默 认 为 off， 表 示 HTTP 头 部 的 名 称 中 不 允 许 带“_”（ 下 划 线）。



### 5.8.2 if_modified_since处理策略

**语法：**

if_modified_since [off|exact|before]

**默认：**

if_modified_since exact;

**配置块：**

http、server、location

**说明：**

* off：表 示 忽 略 用 户 请 求 中 的 If-Modified-Since 头 部。 这 时， 如 果 获 取 一 个 文 件， 那 么 会 正 常 地 返 回 文 件 内 容。 HTTP 响 应 码 通 常 是 200。
* exact：将 If-Modified-Since 头 部 包 含 的 时 间 与 将 要 返 回 的 文 件 上 次 修 改 的 时 间 做 精 确 比 较， 如 果 没 有 匹 配 上， 则 返 回 200 和 文 件 的 实 际 内 容， 如 果 匹 配 上， 则 表 示 浏 览 器 缓 存 的 文 件 内 容 已 经 是 最 新 的 了， 没 有 必 要 再 返 回 文 件 从 而 浪 费 时 间 与 带 宽 了， 这 时 会 返 回 304 Not Modified， 浏 览 器 收 到 后 会 直 接 读 取 自 己 的 本 地 缓 存。
* before：·before： 是 比 exact 更 宽 松 的 比 较。 只 要 文 件 的 上 次 修 改 时 间 等 于 或 者 早 于 用 户 请 求 中 的 If-Modified-Since 头 部 的 时 间， 就 会 向 客 户 端 返 回 304 Not Modified。



### 5.8.3 merge_slashes

**语法：**

merge_slashes on|off;

**默认：**

merge_slashes on;

**配置块：**

http、server、location

**说明：**

表示是否合并相邻的“/"



### 5.8.4 DNS解析地址

**语法：**

resolver address ...;

**配置块：**

http、server、location

**示例：**

```
resolver 127.0.0.1 192.0.2.1;
```

**说明：**

表示是否合并相邻的“/"



### 5.8.4 DNS解析超时时间

**语法：**

resolver timeout ${timeout}

**默认：**

resolver timeout 30s;

**配置块：**

http、server、location

**示例：**

```
resolver 127.0.0.1 192.0.2.1;
```



## 5.9 ngx_http_core_module模块提供的变量

## ![image-20210303100739754](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20210303100739754.png) 

![image-20210303100758201](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20210303100758201.png)



# 6.反向代理服务器

## 6.1 负载均衡基本配置

### 6.1.1 upstream

**语法：**

upstream ${name} {...}

**配置块：**

http

**示例：**

```nginx
upstream backend {
	server backend1.example.com;
	server backend2.example.com;
	server backend3.example.com;
}

server {
	location / {
		proxy_pass http://backend;
	}
}
```

**说明：**

upstream 定义了一个上有服务器的集群



### 6.1.2  server

**语法：**

server ${name}[parameters];

**配置块：**

upstream

**说明：**

server 配 置 项 指 定 了 一 台 上 游 服 务 器 的 名 字， 这 个 名 字 可 以 是 域 名、 IP 地 址 端 口、 UNIX 句 柄 等， 在 其 后 还 可 以 跟 下 列 参 数：

* wight = ${number} 

  设置这台上有服务器转发的权重，默认为1

* max_fails = ${number}

  与fail_timeout配合使用，若timeout时间内，向上游服务器转发失败次数超过number，则在timeout时间段内这台上有服务器不可用。max_fails默认为1，若设置为0，则表示不检查失败次数

* file_timeout = ${time}

  默认为10秒

* down

  表示永久下线，旨在使用ip_hash配置项时才使用。

* backup

  在使用ip_hash时无效，表示所在的上游服务器只是备份服务器，只有在非备份服务器全部失效后才会向所在的上游服务器转发请求



### 6.1.3  ip_hash

**语法：**

ip_hash

**配置块：**

upstream

**说明：**

在 有 些 场 景 下， 我 们 可 能 会 希 望 来 自 某 一 个 用 户 的 请 求 始 终 落 到 固 定 的 一 台 上 游 服 务 器 中。 例 如， 假 设 上 游 服 务 器 会 缓 存 一 些 信 息， 如 果 同 一 个 用 户 的 请 求 任 意 地 转 发 到 集 群 群 中 的 任 一 台 上 游 服 务 器 中， 那 么 每 一 台 上 游 服 务 器 都 有 可 能 会 缓 存 同 一 份 信 息， 这 既 会 造 成 资 源 的 浪 费， 也 会 难 以 有 效 地 管 理 缓 存 信 息。 ip_hash 就 是 用 以 解 决 上 述 问 题 的， 它 首 先 根 据 客 户 端 的 IP 地 址 计 算 出 一 个 key， 将 key 按 照 upstream 集 群 里 的 上 游 服 务 器 数 量 进 行 取 模， 然 后 以 取 模 后 的 结 果 把 请 求 转 发 到 相 应 的 上 游 服 务 器 中。 这 样 就 确 保 了 同 一 个 客 户 端 的 请 求 只 会 转 发 到 指 定 的 上 游 服 务 器 中。

ip_hash 与 weight（ 权 重） 配 置 不 可 同 时 使 用。 如 果 upstream 集 群 中 有 一 台 上 游 服 务 器 暂 时 不 可 用， 不 能 直 接 删 除 该 配 置， 而 是 要 down 参 数 标 识， 确 保 转 发 策 略 的 一 贯 性。 例 如：

```nginx
upstream backend {
	ip_hash;
	server backend1.example.com;
	server backend2.example.com down;
	server backend3.example.com;
}
```



## 6.2 反向代理基本配置

### 6.2.1 proxy_pass

**语法：**

proxy_pass ${URL}

**配置块：**

location、if

**示例：**

```nginx
location / {
	proxy_pass http://www.baidu.com
}
```

**说明：**

默认情况下反向代理不会转发请求中的Host头部，如果需要转发，则必须加上：

```nginx
proxy_set_header Host $host;
```



### 6.2.2  proxy_method

**语法：**

proxy_method ${method}

**配置块：**

http、server、location



### 6.2.3  proxy_hide_header

**语法：**

 proxy_hide_header ${header}

**配置块：**

http、server、location

**说明：**

Nginx 会 将 上 游 服 务 器 的 响 应 转 发 给 客 户 端， 但 默 认 不 会 转 发 以 下 HTTP 头 部 字 段： Date、 Server、 X-Pad 和 X-Accel-*。 使 用 proxy_hide_header 后 可 以 任 意 地 指 定 哪 些 HTTP 头 部 字 段 不 能 被 转 发。



### 6.2.4 proxy_pass_header

**语法：**

 proxy_pass_header ${header}

**配置块：**

http、server、location

**说明：**

与 proxy_hide_header 功 能 相 反， proxy_pass_header 会 将 原 来 禁 止 转 发 的 header 设 置 为 允 许 转 发。



### 6.2.5 proxy_pass_request_body

**语法：**

 proxy_pass_request_body on|off

**默认：**

 proxy_pass_request_body on

**配置块：**

http、server、location

**说明：**

作 用 为 确 定 是 否 向 上 游 服 务 器 发 送 HTTP 包 体 部 分。



### 6.2.6 proxy_pass_request_header

**语法：**

 proxy_pass_request_header on|off

**默认：**

 proxy_pass_request_header on

**配置块：**

http、server、location

**说明：**

作 用 为 确 定 是 否 向 上 游 服 务 器 发 送 HTTP 头 部 分。



### 6.2.7 proxy_redirect

**语法：**

 proxy_redirect [default|off|redirect replacement]

**默认：**

 proxy_redirect default;

**配置块：**

http、server、location

**说明：**

* default

  使 用 默 认 的 default 参 数 时， 会 按 照 proxy_pass 配 置 项 和 所 属 的 location 配 置 项 重 组 发 往 客 户 端 的 location 头 部。 例 如， 下 面 两 种 配 置 效 果 是 一 样 的,location都会变成http://up:port/one/

  ```nginx
  location /one/ {
  	proxy_pass http://up:port/two/;
  	proxy_redirect default;
  }
  
  location /one/ {
  	proxy_pass http://up:port/two/;
  	proxy_redirect http://up:port/two /one/;
  }
  ```

* off

  location 和refresh字段不变

* redirect replacement

  ```nginx
  #该配置的location结果为http://localhost:8000/two/test->http://fronted/one/test
  proxy_redirect http://localhost:8000/two/ http://frontend/one/;
  ```

  

