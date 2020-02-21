# Nginx安装流程

## 安装相关依赖

### 安装gcc

yum install -y gcc-c++

### 安装pcre

yum install -y pcre pcre-devel

### 安装zlib

yum install -y zlib zlib-devel

### 安装openssl

yum install -y openssl openssl-devel

## 安装nginx

### 下载

wget http://nginx.org/download/nginx-1.14.0.tar.gz

### 解压

tar -zxvf nginx-1.14.0.tar.gz

### 编译&安装

第一步：cd nginx-1.14.0

第二步：./configure --prefix=/usr/local/nginx --pid-path=/run/nginx.pid --error-log-path=/var/log/nginx/error.log --http-log-path=/var/log/nginx/access.log  --with-http_ssl_module --with-http_v2_module --with-http_stub_status_module --with-pcre

第三步：make

第四步：make install

### 安装可能发生的错误

**错误**

nginx: [error] open() "/usr/local/nginx/logs/nginx.pid" failed (2: No such file or directory)

**解决方法**

 /usr/local/nginx/sbin/nginx -c /usr/local/nginx/conf/nginx.conf

## nginx启动脚本

在/etc/init.d中添加文件nginx并编辑

````
#! /bin/bash
# chkconfig: - 85 15
PATH=/usr/local/nginx
DESC="nginx daemon"
NAME=nginx
DAEMON=$PATH/sbin/$NAME
CONFIGFILE=$PATH/conf/$NAME.conf
PIDFILE=$PATH/logs/$NAME.pid
SCRIPTNAME=/etc/init.d/$NAME
set -e
[ -x "$DAEMON" ] || exit 0
do_start() {
$DAEMON -c $CONFIGFILE || echo -n "nginx already running"
}
do_stop() {
$DAEMON -s stop || echo -n "nginx not running"
}
do_reload() {
$DAEMON -s reload || echo -n "nginx can't reload"
}
case "$1" in
start)
echo -n "Starting $DESC: $NAME"
do_start
echo "."
;;
stop)
echo -n "Stopping $DESC: $NAME"
do_stop
echo "."
;;
reload|graceful)
echo -n "Reloading $DESC configuration..."
do_reload
echo "."
;;
restart)
echo -n "Restarting $DESC: $NAME"
do_stop
do_start
echo "."
;;
*)
echo "Usage: $SCRIPTNAME {start|stop|reload|restart}" >&2
exit 3
;;
esac
exit 0
````

这里需要注意，第三行的PATH变量默认是/usr/local/nginx,可能需要按照自己的路径来修改。

修改完成后，使用**chmod a+x /etc/init.d/nginx**使其变成可执行文件

## 相关命令

### 启动nginx

service nginx start

### **关闭nginx**

service nginx stop

### 重载nginx

service nginx reload

### 检查nginx配置

nginx -t

## 总结

以上是nginx的相关安装流程，nginx主要用来做反向代理和负载均衡，这两点以后补充。