本手册采用docker-compose的方法进行Kafka的部署，zk的部署请参照zk安装文档

docker-compose文件内容如下：

```
version: '2'

services:
  kafka1:
    image: wurstmeister/kafka
    restart: always
    hostname: kafka1
    container_name: kafka1
    ports:
    - 9092:9092
    environment:
      KAFKA_ADVERTISED_HOST_NAME: kafka1
      KAFKA_ADVERTISED_PORT: 9092
      KAFKA_ZOOKEEPER_CONNECT: host.docker.internal:2181,host.docker.internal:2182,host.docker.internal:2183
    volumes:
    - /Users/kyle/MyTask/kafka/kafka1/logs:/kafka
    external_links:
    - zk1
    - zk2
    - zk3

  kafka2:
    image: wurstmeister/kafka
    restart: always
    hostname: kafka2
    container_name: kafka2
    ports:
    - 9093:9093
    environment:
      KAFKA_ADVERTISED_HOST_NAME: kafka2
      KAFKA_ADVERTISED_PORT: 9093
      KAFKA_ZOOKEEPER_CONNECT: host.docker.internal:2181,host.docker.internal:2182,host.docker.internal:2183
    volumes:
    - /Users/kyle/MyTask/kafka/kafka2/logs:/kafka
    external_links:
    - zk1
    - zk2
    - zk3

  kafka3:
    image: wurstmeister/kafka
    restart: always
    hostname: kafka3
    container_name: kafka3
    ports:
    - 9094:9094
    environment:
      KAFKA_ADVERTISED_HOST_NAME: kafka3
      KAFKA_ADVERTISED_PORT: 9094
      KAFKA_ZOOKEEPER_CONNECT: host.docker.internal:2181,host.docker.internal:2182,host.docker.internal:2183
    volumes:
    - /Users/kyle/MyTask/kafka/kafka3/logs:/kafka
    external_links:
    - zk1
    - zk2
    - zk3


```

