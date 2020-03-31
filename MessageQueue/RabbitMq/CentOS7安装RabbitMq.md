# CentOS7安装RabbitMq

## 一、下载erlang源码

rabbitmq是erlang语言编写的，安装rabbitmq之前，需要先安装erlang，这里用erlang的源码进行安装，erlang安装包官网下载地址：http://erlang.org/download/

安装流程如下：

```
//下载erlang源码
wget http://erlang.org/download/otp_src_20.3.tar.gz
//安装源码编译环境
yum install gcc gcc-c++ glibc-devel make ncurses-devel openssl-devel autoconf git
```



## 二、编译&安装

1.进入文件夹

```
cd otp_src_20.3
```

2.编译

```
./otp_build autoconf
```

3.安装

```
./configure && make && sudo make install
```

4.检验安装是否正确

```
whereis erl
```



## 三、安装RabbitMq

下载rabbitmq

```
wget https://www.rabbitmq.com/releases/rabbitmq-server/v3.6.15/rabbitmq-server-3.6.15-1.el6.noarch.rpm
```

安装

```
rpm -ivh --nodeps rabbitmq-server-3.6.15-1.el6.noarch.rpm 
```

启动

```
rabbitmq-server start
```

额，发现启动报错了。。。。网上查了一下，原来是erlang的版本号太高了，这里发下连接查看对应版本号要求：https://www.rabbitmq.com/which-erlang.html，好吧，只能重新装一遍erlang了，看到这里读者不用担心，因为上面我已经改正了，不过于我而言还是要卸载一下erlang



## 四、卸载Erlang

```
whereis erl
//这里每个人的安装路径可能会不同，但是默认路径应该是一样的
rm -rf /usr/local/lib/erlang/
```



## 五、再次启动

这回想必没有问题啦

```
rabbitmq-server start
```

