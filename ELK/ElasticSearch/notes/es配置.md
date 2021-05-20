es包含3个配置文件：

1.elasticsearch.yml

配置es



2.jvm.options

配置jvm



3.log4j2.properties

配置日志



通过修改ES_PATH_CONF变量，可以调整配置文件所在目录

```
ES_PATH_CONF=/path/to/my/config ./bin/elasticsearch
```



配置文件支持的格式：

![image-20210329085842840](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20210329085842840.png)