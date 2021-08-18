# Redis持久化(二)——AOF

redis是一个缓存类型的数据库，所有的数据都存储在内存中，因此是属于掉电易失的，所以需要一种手段将内存中的数据持久化到磁盘中，这便是RDB和AOF，本文主要介绍AOF

## 一、AOF简介

AOF(Append Only File)记录每次对服务器写的操作,当服务器重启的时候会重新执行这些命令来恢复原始的数据,AOF命令以redis协议追加保存每次写的操作到文件末尾。Redis还能对AOF文件进行后台重写(bgrewrite),使得AOF文件的体积不至于过大。



## 二、AOF工作原理

当redis.conf中appendonly选项为yes，则开启AOF模式，此时redis不会再进行RDB，除非手动触发。当redis崩溃后重启或者正常启动时，会优先从aof文件中恢复数据备份。

AOF持久化功能的实现可以分为**命令追加**、**文件写入**、**文件同步**三个步骤。其中主要调用到的**fsync**函数需要关注



### 命令追加

当AOF持久化功能打开时，服务器在执行完一个写命令之后，会以协议格式将被执行的写命令追加到服务器状态的aof_buf缓冲区的末尾。



### 文件写入和同步

这里的文件即指aof文件，每当服务器常规任务函数被执行、 或者事件处理器被执行时， aof.c/flushAppendOnlyFile 函数都会被调用， 这个函数执行以下两个工作：

**WRITE**：根据条件，将 aof_buf 中的缓存写入到 AOF 文件。

**SAVE**：根据条件，调用 **fsync** 或 fdatasync 函数，将 AOF 文件保存到磁盘中。



### fsync

回顾之前解析配置文件时的**appendfsync everysec**参数，可以发现redis在写入aof文件时有3种不同的策略：**no**、**always**、**everysec**

**AOF_FSYNC_NO**

当每次调用flushAppendOnlyFile函数时，**WRITE**函数都会被执行，但是**SAVE**函数默认不会被执行，除非发生以下情况：

1.Redis被关闭

2.AOF功能被关闭

3.系统写缓存**被写满**后或**定期保存操作**引发的缓存刷新

另外这三种情况下的SAVE操作都会引起Redis主进程阻塞。AOF_SYNC_NO性能较高，因为SYNC函数并不会被调用，数据的同步会交给操作系统去实现。但是若系统缓存中的数据丢失会产生一定影响。

**AOF_FSYNC_ALWAYS**

每执行一条命令，WRITE和SAVE都会被执行，这种策略能使redis的数据完整性得到很大的保障，但是SAVE操作可能会高频率阻塞主进程，并产生大量IO，效率十分低下。

**AOF_SYNC_EVERYSEC**

每秒执行一次WRITE和SAVE操作，这种策略在安全和性能方面相较前两者更加折衷，也是redis的默认使用策略



## 三、AOF重写

由于AOF文件使用的是对每次写操作进行记录，并不像RDB那样以"打包"的形式持久化数据，因此，aof文件大小会比dump文件大很多，所以需要重写机制来改善aof文件的大小，aof的重写可以分为手动触发和自动触发

### 手动触发

使用命令**BGREWRITEOF**可以手动触发AOF的重写，在2.4版本之后，**BGREWRITEOF**会自动触发

### 自动触发

redis何时会自动触发？这里关注redis.conf里的三个参数**auto-aof-rewrite-percentage 100**、**auto-aof-rewrite-min-size 64mb**、**no-appendfsync-on-rewrite no**。前两个参数表示，当redis检测到aof文件的大小超出了(1+0.01*auto-aof-rewrite-percentage)xauto-aof-rewrite-min-size指定的值时，需要进行重写。重写完成后， 将**auto-aof-rewrite-min-size**记录为当前新的aof文件的大小。

当配置参数为**no-appendfsync-on-rewrite **为yes时，表示在rewrite过程中不会调用fsync函数。

### 重写原理

AOF自动重写首先需要满足以下条件：

1.没有BGSAVE或者AOF持久化在执行

2.没有BGREWRITEOF正在执行

3.当前AOF文件大小大于(**auto-aof-rewrite-percentage**x0.01+1)xauto-aof-rewrite-min-size



满足重写条件后，redis服务端创建一个子进程，子进程通过aof_rewrite函数创建新的AOF文件并将指令集写入。但是在子进程创建并写入数据到新的AOF文件的同时，主进程同时在处理新的写请求，这会导致新的aof文件会出现数据不一致现象，因此主进程需要执行以下三个工作来解决数据不一致的问题

1.执行client发来的写请求到redis缓存中

2.将写命令追加到现有的AOF文件中

3.将写命令追加到AOF的重写缓存中

4.子进程完成AOF文件重写，向父进程发送完成信号。父进程则将AOF重写缓存写入到新的AOF文件中，并对新的AOF文件改名，覆盖原有的AOF文件。

![img](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/BGREWRITEOF.png)



### Redis不同版本的重写机制

**4.0之前**

redis会删除抵消的命令和合并重复的命令

**4.0之后**

会将老的数据以rdb的形式append到aof中，将增量以指令的形式append到aof中，使aof文件成为rdb+aof的混合体，这既利用了rdb恢复的快速和aof的全量记录



## 四、AOF的优点

1.使用AOF 会让你的Redis更加耐久: 你可以使用不同的fsync策略：无fsync,每秒fsync,每次写的时候fsync.使用默认的每秒fsync策略,Redis的性能依然很好(fsync是由后台线程进行处理的,主线程会尽力处理客户端请求),一旦出现故障，你最多丢失1秒的数据

2.AOF文件是一个只进行追加的日志文件,所以不需要写入seek,即使由于某些原因(磁盘空间已满，写的过程中宕机等等)未执行完整的写入命令,你也也可使用redis-check-aof工具修复这些问题.

3.Redis 可以在 AOF 文件体积变得过大时，自动地在后台对 AOF 进行重写： 重写后的新 AOF 文件包含了恢复当前数据集所需的最小命令集合。 整个重写操作是绝对安全的，因为 Redis 在创建新 AOF 文件的过程中，会继续将命令追加到现有的 AOF 文件里面，即使重写过程中发生停机，现有的 AOF 文件也不会丢失。 而一旦新 AOF 文件创建完毕，Redis 就会从旧 AOF 文件切换到新 AOF 文件，并开始对新 AOF 文件进行追加操作。

4.AOF 文件有序地保存了对数据库执行的所有写入操作， 这些写入操作以 Redis 协议的格式保存， 因此 AOF 文件的内容非常容易被人读懂， 对文件进行分析（parse）也很轻松。 导出（export） AOF 文件也非常简单： 举个例子， 如果你不小心执行了 FLUSHALL 命令， 但只要 AOF 文件未被重写， 那么只要停止服务器， 移除 AOF 文件末尾的 FLUSHALL 命令， 并重启 Redis ， 就可以将数据集恢复到 FLUSHALL 执行之前的状态。



## 五、AOF缺点

1.对于相同的数据集来说，AOF 文件的体积通常要大于 RDB 文件的体积。

2.根据所使用的 fsync 策略，AOF 的速度可能会慢于 RDB 。 在一般情况下， 每秒 fsync 的性能依然非常高， 而关闭 fsync 可以让 AOF 的速度和 RDB 一样快， 即使在高负荷之下也是如此。 不过在处理巨大的写入载入时，RDB 可以提供更有保证的最大延迟时间（latency）。