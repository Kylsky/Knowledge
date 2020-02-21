# Redis配置文件解析

参考文章：https://www.cnblogs.com/itoyr/p/10069396.html

本文只介绍了一部分配置文件内容，剩余的配置部分在之后有碰到的使用场景再补充。

## 一、INCLUDES

可以用来导入其他配置文件，如：

include /path/to/local.conf



## 二、MODULES

在redis启动时加载模组，如：

loadmodule /path/to/my_module.so



## 三、NETWORK

### *tcp-backlog 511

设置tcp的backlog，backlog其实是一个连接队列，backlog队列总和=未完成三次握手队列 + 已经完成三次握手队列。

在高并发环境下你需要一个高backlog值来避免慢客户端连接问题。注意Linux内核会将这个值减小到/proc/sys/net/core/somaxconn的值，所以需要确认增大somaxconn和tcp_max_syn_backlog两个值来达到想要的效果



### *protected-mode <yes|no>

是否开启保护模式，默认开启。若配置中没有指定bind和密码，开启该参数后，redis只会本地访问，拒绝外部访问。



### *bind [ipAddr]

指定redis只接受来自于该IP地址的请求，若不进行设置，那么将处理所有请求



### *port [6379]

redis服务器监听的端口号



### timeout 0

客户端超时断开连接的时间，若为0，则服务端不会主动断开连接,单位为秒



### tcp-keepalive 300

监控对等端是否已经关闭，关闭时间为默认时间的两倍，默认时间为300秒。



## 四、General

### supervised no

可以通过upstart和systemd管理Redis守护进程
选项：
  supervised no - 没有监督互动
  supervised upstart - 通过将Redis置于SIGSTOP模式来启动信号
  supervised systemd - signal systemd将READY = 1写入$ NOTIFY_SOCKET
  supervised auto - 检测upstart或systemd方法基于 UPSTART_JOB或NOTIFY_SOCKET环境变量

### daemonize yes

是否开启后台启动方式，默认不开启，一般设置为开启



### pidfile /var/run/redis_6379.conf.pid

配置进程文件所在目录



### *loglevel notice

日志等级，分为debug、verbose、notice、warning

**debug（记录大量日志信息，适用于开发、测试阶段)**

**verbose（较多日志信息）**

**notice（适量日志信息，适用于生产环境）**

**warning（仅有部分重要、关键信息才会被记录）**



### logfile /var/redis/log/redis_6379.conf

日志输出路径



### syslog-enabled

是否把日志输出到系统日志中，默认为no



### syslog-ident redis

设置系统日志标识，默认是redis



### databases

16个库



### always-show-logo yes

redis只有在开始输出标准日志文件时默认展示logo



### syslog-facility local0

指定系统日志设置，必须是USER或是LOCAL0-LOCAL7之间的值



## 五、SNAPSHOTTING

### save [时间] [写入次数]

根据给定时间间隔和写入次数将数据RDB到磁盘，即n秒内至少有m次key值变化，则保存。可以通过save “”停用



### stop-writes-on-bgsave-error yes

如果用户开启了RDB快照功能，那么在redis持久化数据到磁盘时如果出现失败，默认情况下，redis会停止接受所有的写请求。这样做的好处在于可以让用户很明确的知道内存中的数据和磁盘上的数据已经存在不一致了。



### rdbcompression yes

采用LZF算法压缩RDB文件



### *rdbchecksum yes

存储快照后让redis使用CRC64算法来进行数据校验，这样会增加大约10%性能消耗



### dbfilename dump.rdb

设置快照文件名



### *dir /var/lib/redis/6379

RDB文件会被写入到指定目录下



### 六、REPLICATION

### replicaof <masterip> <matsterport>

称为指定redis服务的从机



### masterauth <master-password>

在从机复制之前，从机需要先通过masterauth校验master的密码。masterauth用于从机向master复制时使用



### replica-server-state-data-yes

当一个从机断开了与主机的连接，或者复制任务仍在进行，从机会使用两种不同的机制应对：

1.如果该选项为yes，从机仍会响应客户端的请求，并返回可能过期的数据。或者由于从机出于第一次同步因此数据是空的

2.如果该选项为no，从机会回复一个错误代码



### replica-read-only yes

从机只可读



### reply-diskless-sync no

主从复制采用disk(磁盘)传输或socket(套接字)传输，默认采用的是disk。新的从机和重新连接的从机连上主机后和主机的数据是不一致的，需要做full synchronization。一个RDB文件会被master传输到从机种，传输可分为两种方式：

1.Disk-backed。master创建一个子进程写入RDB到磁盘。稍后RDB文件被父进程传输到从机

2.Diskless。master创建一个子进程直接通过socket写入RDB到从机，直接跳过了磁盘

在硬盘备份的情况下，主站的子进程生成RDB文件。一旦生成，多个从站可以立即排成队列使用主站的RDB文件。
 在无硬盘备份的情况下，一次RDB传送开始，新的从站到达后，需要等待现在的传送结束，才能开启新的传送。
 如果使用无硬盘备份，主站会在开始传送之间等待一段时间（可配置，以秒为单位），希望等待多个子站到达后并行传送。
 在硬盘低速而网络高速（高带宽）情况下，无硬盘备份更好。



### repl-diskless-sync-delay 5

当启用无硬盘备份，服务器等待一段时间后才会通过套接字向从站传送RDB文件，这个等待时间是可配置的。 这一点很重要，因为一旦传送开始，就不可能再为一个新到达的从站服务。从站则要排队等待下一次RDB传送。因此服务器等待一段时间以期更多的从站到达。 延迟时间以秒为单位，默认为5秒。要关掉这一功能，只需将它设置为0秒，传送会立即启动。



### repl-ping-slave-period 10

从机会周期性的向master发出PING包，你可以通过repl_ping_slave_period指令来控制其周期，默认是10秒。



### repl-timeout 60

接下来的选项为以下内容设置备份的超时时间：
 1）从replica的角度，同步期间的批量传输的I/O
 2）replica角度认为的master超时（数据，ping）
 3）master角度认为的replica超时（REPLCONF ACK pings)
 确认这些值比定义的repl-ping-slave-period要大，否则每次主站和从站之间通信低速时都会被检测为超时。



### repl-disable-tcp-nodelay no

如果选择yes，对于从机的复制会占用更少的TCP报文和带块，但是这可能会在replica端出现延迟，大概时linux默认的40ms

如果选择no，replica延迟就会降低，带宽会更高

默认情况下将潜在因素优化，但在高负载情况下或者在主从都出于高延迟情况下， 切换为yes是个好主意



### repl-backlog-size 1mb

 设置备份的工作储备大小。工作储备是一个缓冲区，当从站断开一段时间的情况时，它替从站接收存储数据， 因此当从机重连时，通常不需要完全备份，只需要一个部分同步就可以，即把从站断开时错过的一部分数据接收。工作储备越大，从站可以断开并稍后执行部分同步的断开时间就越长。只要有一个从机连接，就会立刻分配一个工作储备。



### repl-backlog-ttl 3600

主机有一段时间没有与从机连接，对应的工作储备就会自动释放。这个选项用于配置释放前等待的秒数，秒数从断开的那一刻开始计算，若值设置为0则标识不释放。



### slave-priority 100

从机优先级，可以从redis的INFO命令中查询到。当主机无法工作时，redis哨兵会通过优先级来选择从机提升为master。哨兵会首先提升优先级最小的从机，但不会提升优先级为0的从机。



###  min-slaves-to-write 3

###  min-slaves-max-lag 10

当从机小于N个，数据滞后小于等于M秒，那么master可能会停止接受写请求。需要注意N个replica需要处于在线模式。

lag必须小于等于指定值，是通过replica发送的PING计算得到的，通常每秒都会发送。

这个选项不保证N个从机会接受写请求，但是会限制在指定秒数内由于从机数量不够导致的写操作丢失的情况。



### slave-announce-ip 5.5.5.5

### slave-announce-port 1234

向master报告一组特定的IP和端口



## 七、SECURITY

### requirepass foobared

设置redis连接密码。相对于masterauth，requirepass指的是redis客户端连接redis服务端所需要的密码。



 

### rename-command CONFIG ""

将命令重命名，为了安全考虑，可以将某些重要的、危险的命令重命名。当你把某个命令重命名成空字符串的时候就等于取消了这个命令。需要注意的是，改变那些被记录在AOF或者被传输到从机的命令可能会产生问题



## 八、CLIENTS

### maxclients 10000

设置客户端最大并发连接数，默认无限制，Redis可以同时打开的客户端连接数为Redis进程可以打开的最大文件描述符数-32（redis server自身会使用一些），如果设置 maxclients为0表示不作限制。当客户端连接数到达限制时，Redis会关闭新的连接并向客户端返回max number of clients reached错误信息



### 九、MEMORY MANAGEMENT

### maxmemory <bytes>

指定Redis最大内存限制，Redis在启动时会把数据加载到内存中，达到最大内存后，Redis会先尝试清除已到期或即将到期的Key。当此方法处理后，仍然到达最大内存设置，将无法再进行写入操作，但仍然可以进行读取操作。Redis新的vm机制，会把Key存放内存，Value会存放在swap区



### maxmemory-policy noeviction

 当内存使用达到最大值时，redis使用的清除策略。有以下几种可以选择：
 1.**volatile-lru**  利用LRU算法移除设置过过期时间的key (LRU:最近使用 Least Recently Used )
 2.**allkeys-lru**  利用LRU算法移除任何key
 3.**volatile-random** 移除设置过过期时间的随机key
 4.**allkeys-random** 移除随机key
 5.**volatile-ttl**  移除即将过期的key(minor TTL)
 6.**noeviction noeviction**  不移除任何key，只是返回一个写错误 ，默认选项



### maxmemory-samples 5

 LRU 和 minimal TTL 算法都不是精准的算法，而是相对精确的算法(为了节省内存)。你可以选择样本大小进行检测，redis默认选择3个样本进行检测，可以通过maxmemory-samples进行设置样本数

 

### replica-ignore-maxmemory yes

是否开启slave的最大内存





## 十、LAZY FREEING

### lazyfree-lazy-eviction no

### lazyfree-lazy-expire no

### lazyfree-lazy-server-del no

### replica-lazy-flush no

redis有两种基本的删除key的方法，一种叫DEL,是阻塞的，这表示server为了清除需要清理对象涉及的所有内存会以同步形式停止处理新的命令。若删除的对象比较小，则时间按复杂度为O(1)或O(logN)，但如果对象很大，server会阻塞很久。

因此redis提供了一个非阻塞的删除方法——**UNLINK**以及异步的**FLUSHALL和FLUSHDB方法**

lazy free可译为惰性删除或延迟释放；当删除键的时候,redis提供异步延时释放key内存的功能，把key释放操作放在bio(Background I/O)单独的子线程处理中，减少删除big key对redis主线程的阻塞。有效地避免删除big key带来的性能和可用性问题。

惰性删除主要为4个方面：内存满需要清除、过期键需要清除、服务端删除、flush清除



## 十一、APPEND ONLY MODE

### appendonly no

默认情况下redis异步备份数据集到磁盘中(RDB)，这种模式在和多场和使用，但是可能会导致数分钟的写丢失(依赖于save的配置)

AOF是一种可选的持久化模式，提供了更好的持久性。使用默认的数据fsync协议，redis只会丢失一秒的写请求。

AOF和RDB可以同时存在，如果AOF启用了，那么在redis启动时会优先加载AOF，以保证数据的持久化。默认情况AOF被关闭。



### appendfilename "appendonly.aof"

aof文件名



### appendfsync everysec

fsync()函数通知操作系统向磁盘写入数据，即使输出缓存没有被尽可能装满。一些OS会立刻刷新数据到磁盘，一些OS只会尽力而为

Redis支持3种写入模式：

**1.no**。不使用fsync，直接依靠操作系统刷新数据到磁盘，速度快

**2.always**。使用fsync，每一次写都立刻刷新缓冲，速度较慢，但是安全

**3.everysec**。每秒使用一次fsync，比较折衷的方案，默认选项



### no-appendfsync-on-rewrite no

当AOF fsync被设置为always或everysec，后台存储进程会会执行大量IO，一些Linux配置种redis可能会阻塞很长时间来调用fsync(),目前还没有解决方法。

为了减轻这个问题的影响，可以使用这个参数来阻止fsync()在主进程执行**BGSAVE或BGREWRITEOF**时被调用



### auto-aof-rewrite-percentage 100

### auto-aof-rewrite-min-size 64mb

redis会保存上一次重写时的aof文件大小，当aof文件到达**指定大小**的**百分比**时，redis会自动调用BGREWRITEAOF重写aof文件。



### aof-load-truncated yes

在Redis启动过程中，当AOF数据加载回内存时，可能会发现AOF文件在最后被截断。当运行Redis的系统崩溃时可能会发生这种情况，特别是在没有安装ext4文件系统时(然而，当Redis本身崩溃或中止，但操作系统仍然正常工作时，这种情况就不会发生)。

发生这种情况时，Redis既可以带着错误退出，也可以尽可能多地加载。如果发现AOF文件在最后被截断，则开始执行。aof-load-truncated选项控制这种行为。

如果将AOF -load-truncated设置为yes，则加载一个截断的AOF文件，并且Redis服务器开始发出日志通知用户该事件。否则，如果将该选项设置为no，则服务器将带错误中止并拒绝启动。当该选项设置为no时，用户需要使用“redis-check-aof”实用程序修复AOF文件，然后才能重新启动服务器。

注意，如果AOF文件在中间被破坏，服务器仍然会带着错误退出。此选项仅适用于当Redis试图从AOF文件中读取更多数据，但没有找到足够的字节时。



### aof-use-rdb-preamble yes

是否混用RDB和AOF混合模式



## 十二、REDIS CLUSTER

### cluster-enabled yes

普通的redis实例无法成为集群的一部分，只有以集群的节点形式启动才行。当该参数设置为yes，标识启用了集群模式



### cluster-config-file nodes-6379.conf

每个集群节点都有一个集群配置文件。此文件不打算手工编辑。它由Redis节点创建和更新。每个Redis集群节点需要一个不同的集群配置文件。确保在同一系统中运行的实例没有重叠的集群配置文件名。



### cluster-node-timeout 15000

集群节点超时是指一个节点必须在无法到达的时间内处于故障状态。大多数其他内部时间限制是多个节点超时。默认为15s



### cluster-replica-validity-factor 10

从机有效引子。当一个集群的master挂了的时候，replica会根据该值判断是否需要failover提升为master，若为0，则会一直尝试称为master，其他情况下值越小越会阻止replica提升为master



### cluster-migration-barrier 1

cluster-migration-barrier属性可以保证redis集群中不会出现裸奔的主节点（这个主节点没有对应的从节点），当某个主节点的从节点挂掉裸奔后，会从其他富余的主节点分配一个从节点过来，确保每个主节点都有至少一个从节点，不至于因为主节点挂掉而没有相应从节点替换为主节点导致集群崩溃不可用。



### cluster-require-full-coverage yes

当cluster-require-full-coverage为no时，表示当负责一个插槽的主库下线且没有相应的从库进行故障恢复时，集群仍然可用，反之集群不可用。



### cluster-replica-no-failover no

若设置为yes，在主节点失效期间,从节点不允许对master失效转移