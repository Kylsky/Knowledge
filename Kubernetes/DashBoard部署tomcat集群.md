## 1.在node节点加入docker阿里云仓库

参考：

```
https://www.cnblogs.com/allenjing/p/12575972.html
```



## 2.创建容器

随便选一个

![image-20201215100715718](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20201215100715718.png)



![image-20201215101156471](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20201215101156471.png)

如上图，主要需要观察的是需要配置2个端口，第一个是指pod的端口，第二个则是指容器映射的tomcat的端口，点击部署即可。



## 3.问题排查

如果tomcat:latest安装的是9.x版本，可能无法直接通过ip+端口访问，需要进入docker容器进行以下操作：

```
docker exec -it <containerId> /bin/bash
rm -rf webapps
mv webapps.dist webapps
```

有点麻烦，所以推荐安装8.x版本