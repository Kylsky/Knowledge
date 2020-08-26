# Redis的安装

#### 前置要求

yum install wget

yum install gcc

#### 下载安装包

wget  http://download.redis.io/releases/redis-5.0.7.tar.gz 

#### 解压安装包

tar -zxvf redis-5.0.7.tar.gz

cd redis-5.0.7

#### 安装redis

make

make install PREF=/opt/redis5

#### 配置环境

vi /etc/profile

**在结尾加上：**

export REDIS_HOME=/opt/redis5

export PATH=$PATH:$REDIS_HOME/bin

**保存并退出编辑，同时执行：**

source /etc/profile

#### 配置redis_server

cd utils

./install search.sh

后续操作默认情况直接回车即可，端口为6379，执行完成后，脚本会自动启动redis

#### 测试redis是否启动

service redis_6379 status

