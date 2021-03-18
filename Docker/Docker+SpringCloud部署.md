# 1.配置阿里云镜像

sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://8v0jihoa.mirror.aliyuncs.com"],
  "log-driver":"json-file",
  "log-opts":{ "max-size" :"1g","max-file":"1"}
}
EOF



# 2.安装docker及配置    

```
sudo yum install -y wget
sudo wget https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo -O /etc/yum.repos.d/docker-ce.repo
sudo yum -y install docker-ce-18.06.1.ce-3.el7
sudo systemctl enable docker && sudo systemctl start docker
```





# 3.nacos集群部署

使用3台nacos作为集群，以下是每台机器的操作，注意将server信息替换，或者在hosts添加地址解析，另外数据库配置需要修改。



## 3.1 操作1

```shell
docker pull nacos/nacos-server:1.3.2
sudo docker run --name=nacos-1 --network=host --restart=always -d -e NACOS_SERVER_IP=server1 -e NACOS_SERVERS=server1:8848,server2:8848,server3:8848 -e SPRING_DATASOURCE_PLATFORM=mysql -e MYSQL_SERVICE_HOST=127.0.0.1 -e MYSQL_SERVICE_PORT=3306 -e MYSQL_SERVICE_DB_NAME=nacos_config -e MYSQL_SERVICE_USER=root -e MYSQL_SERVICE_PASSWORD=123456 nacos/nacos-server:1.3.2
```



## 3.2 操作2

```shell
sudo docker pull nacos/nacos-server:1.3.2
sudo docker run --name=nacos-2 --network=host --restart=always -d -e NACOS_SERVER_IP=server2 -e NACOS_SERVERS=server1:8848,server2:8848,server3:8848 -e SPRING_DATASOURCE_PLATFORM=mysql -e MYSQL_SERVICE_HOST=127.0.0.1 -e MYSQL_SERVICE_PORT=3306 -e MYSQL_SERVICE_DB_NAME=nacos_config -e MYSQL_SERVICE_USER=root -e MYSQL_SERVICE_PASSWORD=123456 nacos/nacos-server:1.3.2
```



## 3.3 操作3

```shell
sudo docker pull nacos/nacos-server:1.3.2
sudo docker run --name=nacos-3 --network=host --restart=always -d -e NACOS_SERVER_IP=server3 -e NACOS_SERVERS=server1:8848,server2:8848,server3:8848 -e SPRING_DATASOURCE_PLATFORM=mysql -e MYSQL_SERVICE_HOST=127.0.0.1 -e MYSQL_SERVICE_PORT=3306 -e MYSQL_SERVICE_DB_NAME=nacos_config -e MYSQL_SERVICE_USER=root -e MYSQL_SERVICE_PASSWORD=123456 nacos/nacos-server:1.3.2
```

# 4.RabbitMq

```shell
sudo docker pull rabbitmq:3-management

sudo docker run --name rabbitmq -d -p 15672:15672 -p 5672:5672 ${image_id}

#进入容器
docker exec -it /bin/bash ${container_id}
rabbitmqctl   add_user  kyle Skyline302=
rabbitmqctl set_user_tags kyle  administrator
rabbitmqctl set_permissions -p / kyle '.*' '.*' '.*'
#退出容器
exit
```





# 5.elasticsearch:7.3.2

```shell
sudo docker pull elasticsearch:7.3.2

sudo docker tag  d7052f192d01 elasticsearch:7.3.2

sudo docker run -itd -p 9200:9200 -p 9300:9300  --privileged=true --name elasticsearch-server -e "discovery.type=single-node"  elasticsearch:7.3.2

#复制出配置
sudo docker cp elasticsearch-server:/usr/share/elasticsearch/config/elasticsearch.yml /opt/elasticsearch.yml
#修改增加配置
cluster.initial_master_nodes: ["node-1"]
node.name: node-1
http.cors.enabled: true
http.cors.allow-origin: "*"
#回写配置
sudo docker cp  /opt/elasticsearch.yml elasticsearch-server:/usr/share/elasticsearch/config/elasticsearch.yml 

#校验部署是否完成，最后的/不能少
curl http://localhost:9200/
```



# 6.Kibana:7.3.2

```shell
sudo docker pull docker.elastic.co/kibana/kibana:7.3.2

sudo docker tag docker.elastic.co/kibana/kibana:7.3.2  kibana:7.3.2

sudo docker run --name kibana7.3.2 --privileged=true --link elasticsearch-server:elasticsearch -e ELASTICSEARCH_HOSTS=http://elasticsearch:9200  -p 5601:5601 -d  kibana:7.3.2

#进入容器后操作
vi config/kibana.yml
#添加数据
i18n.locale: "zh-CN"
#退出容器
exit

#校验部署是否完成
curl http://localhost:5601/
```



# 7.logstash:7.3.2

```shell
sudo docker pull docker.elastic.co/logstash/logstash:7.3.2

sudo docker tag docker.elastic.co/logstash/logstash:7.3.2  logstash:7.3.2

sudo docker run -itd --privileged=true -p 5043:5043 --name logstash-server --link elasticsearch-server:elasticsearch logstash:7.3.2
```



# 8.通用应用部署

```shell
sudo docker run -d -v /usr/local/service/${服务名称}:/usr/local/webapps -v /home/security/js:/home/security/js  --restart=always --network host --privileged --name  ${服务名称} -w /usr/local/webapps  registry.cn-hangzhou.aliyuncs.com/kun/jre:urandom java -jar ${服务名称}.jar
```


