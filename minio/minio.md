# 1.说明

本文档使用docker进行安装，minio版本可以参考

```
https://github.com/minio/minio/releases
```

# 2.安装流程

## 2.1 拉取镜像

```
docker pull minio/minio:RELEASE.2021-03-04T00-53-13Z
```



## 2.2 创建挂载路径

```
mkdir /home/docker_data/minio
```



## 2.3 启动容器

要求用户名不能小于3位，密码不能小于8位

```
docker run -p 9000:9000 --name minio \
  -e "MINIO_ROOT_USER=pubinfo" \
  -e "MINIO_ROOT_PASSWORD=Pubinfo918@" \
  -v /home/docker_data/minio:/data \
  minio/minio:RELEASE.2021-03-04T00-53-13Z server /data
```



## 2.4 测试

```
192.168.1.116:9000
pubinfo
Pubinfo918@
```



# 3.