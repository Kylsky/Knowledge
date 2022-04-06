本文使用docker指令方式来进行zk搭建



# 1.启动节点

````
docker run -d --name zk1 -p 2181:2181 -d zookeeper
docker run -d --name zk2 -p 2182:2181 -d zookeeper
docker run -d --name zk3 -p 2183:2181 -d zookeeper
````



# 2.获取ip地址

```
docker inspect <container_name>
```



# 3.修改配置文件

```
# 如果没有vi，可以通过apt-get安装或使用touch
# 进入容器
docker exec -it zk1 /bin/bash
# 容器中默认是没有vi或vim的
# 安装vim 
apt-get update
apt-get install vim
# 修改zk的配置文件
vim /conf/zoo.cfg
# 修改配置文件之后查看
cat /conf/zoo.cfg

# 修改配置文件信息如下：
clientPort=2181
dataDir=/data
dataLogDir=/datalog
tickTime=2000
initLimit=5
syncLimit=2
# 2888是zk集群之间通信的端口
# 3888是zk集群之间投票选主的端口
server.1=172.17.0.2:2888:3888
server.2=172.17.0.3:2888:3888
server.3=172.17.0.4:2888:3888

# 将1输出到/data/myid文件中
# 从配置中看出，zk1的服务名为server.1, 这 1 是指定的服务名称  
# 下面三条指令分别对应3台zk
echo 1 > /data/myid
echo 2 > /data/myid
echo 3 > /data/myid
```



# 4.重启容器

```
docker restart zk1 zk2 zk3
```



# 5.验证集群状态

```
docker exec -it zk1 /bin/bash
cd bin 
./zkServer.sh status
```

如果碰到出现状态异常的，将3台zk全部重启后重试