### 1.拉取镜像

```
docker pull nacos/nacos-server:1.3.1
or
docker pull nacos/nacos-server
```



### 2.启动容器

```
docker run -d -p 8858:8848 -e MODE=standalone -v /home/docker_data/nacos-1.3.1/logs:/home/nacos/logs --restart always --name nacos-1.3.1
```



### 3.访问页面

```
http://${ip}:${port}/nacos
```

