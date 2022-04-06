本文档使用脚本形式启动rockermq，版本为4.5.1，相关资源可以自行搜索，或使用官方提供的

```
http://rocketmq.apache.org/release_notes/release-notes-4.5.1/
```



## 1.启动name server

```
nohup sh bin/mqnamesrv &
```



## 2.启动broker

```
nohup sh bin/mqbroker -n localhost:9876 &
```



## 3.启动rocketmq console

console这里我从github拉下代码直接跑了

```
https://github.com/zhangyl/rocketmq-console
```

在application.properties里配置：

```
rocketmq.config.namesrvAddr=localhost:9876
```

启动后访问localhost:8080，正常启动应该如下：

![image-20220328145553212](/Users/kyle/Library/Application Support/typora-user-images/image-20220328145553212.png)