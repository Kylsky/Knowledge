# ElasticSearch(二)——启动es

## 一、启动elasticsearch

**linux下下载安装包**

```
curl -L -O https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-7.6.2-linux-x86_64.tar.gz
```

**解压**

```
tar -xvf elasticsearch-7.6.2-linux-x86_64.tar.gz
```

**启动**

```
cd elasticsearch-7.6.2/bin
./elasticsearch
```

**启动多台实例**

```
./elasticsearch -Epath.data=data2 -Epath.logs=log2
./elasticsearch -Epath.data=data3 -Epath.logs=log3
```

额外的节点被分配唯一的id。因为在本地运行了三个节点，所以它们会自动与第一个节点加入集群。

**查看实例启动情况**

使用cat health API来验证您的三节点集群是否正在运行。cat api以一种比原始JSON更易于阅读的格式返回关于集群和索引的信息。

通过向Elasticsearch REST API提交HTTP请求，可以直接与集群交互。如果已经安装并运行了Kibana，您还可以打开Kibana并通过开发控制台提交请求。

```
curl -s http://localhost:9200/_cat/health?v
```

**注意**

不能以root用户启动elasticsearch，那么就创建一个

```
adduser elasticsearch
passwd elasticsearch
#第一个指用户名，第二个为可执行文件
chown -R elasticsearch:elasticsearch /opt/elasticsearch-7.6.2/
su elasticsearch
./elasticsearch -d
```



## 二、为文档建立索引

一旦集群启动并运行，就可以为一些数据建立索引了。Elasticsearch有各种各样的摄取选项，但最终它们都做相同的事情:将JSON文档放入到Elasticsearch索引中。

### 发送request

想服务器发送PUT请求，uri表示向名为**customer**的index传输类型为_doc的文档，该文档的id为1，doc的内容为"name":"John Doe"

```
curl -X PUT "localhost:9200/customer/_doc/1?pretty" -H 'Content-Type: application/json' -d'
{
  "name": "John Doe"
}
'
```

### 接收response

```
{
  "_index" : "customer",
  "_type" : "_doc",
  "_id" : "1",
  "_version" : 1,
  "result" : "created",
  "_shards" : {
    "total" : 2,
    "successful" : 1,
    "failed" : 0
  },
  "_seq_no" : 0,
  "_primary_term" : 1
}
```

由于这是一个新文档，因此响应表明操作的结果是创建了文档的版本1

### 获取doc

```
#这是请求
curl -X GET "localhost:9200/customer/_doc/1?pretty"
#这是响应
{
  "_index" : "customer",
  "_type" : "_doc",
  "_id" : "1",
  "_version" : 1,
  "_seq_no" : 26,
  "_primary_term" : 4,
  "found" : true,
  "_source" : {
    "name": "John Doe"
  }
}
```



## 三、批量索引文档

如果您有很多文档需要索引，可以使用bulk API分批提交它们。使用bulk来批量处理文档操作比单独提交请求要快得多，因为它最小化了网络往返的传输。

最佳批数量取决于许多因素:文档大小和复杂性、索引和搜索负载以及集群可用的资源。一个好的起点是批量处理1,000到5,000个文档，总有效负载在5MB到15MB之间。在这个范围内，可以尝试找到最佳位置。

```
#下载多个doc
wget https://github.com/elastic/elasticsearch/blob/master/docs/src/test/resources/accounts.json
#批量索引doc
curl -H "Content-Type: application/json" -XPOST "localhost:9200/bank/_bulk?pretty&refresh" --data-binary "@accounts.json"
#查看index
curl "localhost:9200/_cat/indices?v"
```



## 四、配置外网访问es

在es安装路径下配置elasticsearch.yml，需要修改内容如下：

```
network.host: ***.**.**.**

port: 9200

cluster.initial_master_nodes: ["node1"]
```

