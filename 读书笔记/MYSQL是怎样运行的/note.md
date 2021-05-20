## Mysql执行流程

![image-20210518105518581](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20210518105518581.png)



## Mysql启动命令

### --skip-networking

禁止客户端使用TCP/IP网络进行通信



### --default-storage-engine=MyISAM

设置默认存储引擎

```
注意，写成--default-storage-engine = MyISAM，像这样携带了空白字符是不允许的
```



### -h || --host

设置主机名



### -u || --user

设置用户名



### -p || --password

设置密码



### -P || --Port

设置端口，默认3306



### -V || --version

版本信息



### --dafaults-file

该命令可以组织mysql到默认的路径下搜索配置文件



### --defaults-extra-file

添加额外的配置文件路径



## Mysql配置文件

### 配日志文件路径

windows:

![image-20210518161401811](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20210518161401811.png)



类unix：

![image-20210518161534550](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20210518161534550.png)



### 配置文件内容

![image-20210518161935928](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20210518161935928.png)

举例如下：

```
[server]
option1
option2 = value2

以上操作等同于mysqld --option1 --option2=value2
```

另外，如果选项组名称与程序名称一致，则选项下的参数将专门用于该程序，如[mysql]和[mysqld]。还有两个选项组比较特别：[server]、[client]，这两个组下边的启动选项将作用于所有的服务端或客户端程序。

![image-20210518163057406](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20210518163057406.png)



### 配置文件优先级

如果在多个配置文件中设置了相同的启动选项，会以最后一个配置文件中的为准。

另外，如果出现了如下的情况：

```
[server] 
default-storage-engine=InnoDB

[mysqld]
default-storage-engine=MyISAM
```

那么，将以最后一个出现的组中的启动选项为准



## Mysql系统变量

### 查看系统变量

show variables [like 匹配的模式];

```
show variables like 'default_storage_engine';
show variables like 'max_connections';
```



### 设置系统变量

1.通过命令行设置

mysqld --default-storage-engine=MyISAM --max-connections=10



2.通过配置文件设置

```
[server] 
default-storage-engine=MyISAM
max-connections=10
```



### 不同作用范围的系统变量

作用范围分为2种：

1.GLOBAL

全局变量，影响服务器的整体操作

2.SESSION（LOCAL）

会话变量，影响某个客户端连接的操作

![image-20210519102816542](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20210519102816542.png)

通过以下命令可以查看不同作用范围的系统变量

```
show [GLOBAL | SESSION] variables [like 匹配的模式]
```



## 字符集和比较规则

![image-20210519111834766](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20210519111834766.png)



### 字符集查看

show (character set | charset) [like 匹配的模式]



### 比较规则查看

show collation [like 匹配的模式]



### 比较规则

| 后缀 | 英文释义           | 描述             |
| ---- | ------------------ | ---------------- |
| _ai  | accent insensitive | 不区分重音       |
| _ci  | case insensitive   | 不区分大小写     |
| _cs  | case sensitive     | 区分大小写       |
| _bin | binary             | 以二进制方式比较 |



### 比较规则和字符集的级别

mysql有4个级别的字符集和比较规则

* 服务器级别
* 数据库级别
* 表级别
* 列级别

