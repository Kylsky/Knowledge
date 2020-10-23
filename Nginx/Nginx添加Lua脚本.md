### 1.安装LuaJIT

```
wget http://luajit.org/download/LuaJIT-2.0.5.tar.gz
tar -zxvf LuaJIT-2.0.5.tar.gz
cd LuaJIT-2.0.5
make
make install
export LUAJIT_LIB=/usr/local/lib
export LUA_INC=/usr/local/include/luajit-2.0
```

### 2.下载ngx_devel_kit

```
wget https://github.com/simpl/ngx_devel_kit/archive/v0.3.0.tar.gz
解压到/usr/local下
```

### 3.下载ngx_lua

```
wget https://github.com/openresty/lua-nginx-module/archive/v0.10.10.tar.gz
解压到/usr/local下
```

### 4.重新编译nginx

```
cd到nginx目录
nginx -V 将内容复制
./configure <复制的内容> --add-module=/usr/local/ngx_devel_kit-0.3.0 --add-module=/usr/local/lua-nginx-module-0.10.10
make
cp /usr/local/nginx/sbin/nginx /usr/local/nginx/sbin/nginx.bak
cp obj/nginx /usr/local/nginx/sbin/nginx /usr/local/nginx/sbin/
```

### 5.关闭nginx

```
service nginx stop
```

### 6.检测

```
nginx -t
```

### 7.no such file or directory解决方案

```
ln -s /usr/local/lib/libluajit-5.1.so.2 /lib64/
```

### 8.重启nginx

```
service nginx start
```