# 倒排索引数据结构

1.包含当前关键词的document list

2.关键词在每个doc中出现的次数

3.关键词在整个索引中出现的次数

4.每个doc的长度，越长则该doc的相关度越低

5.包含当前关键词的所有的doc的平均长度



# Lucene在es中的position

1.创建倒排索引

2.提供了复杂的api

3.存在问题：单点



# ES优点

面向开发者友好，屏蔽了lucene复杂特性，集群自动发现

自动维护数据在多节点的建立

搜索请求的负载均衡

自动维护副本

可以构建大型分布式集群

复合查询、聚合分析、地理位置查询

全文检索、同义词处理、相关度排名、近实时处理。



# 应用领域

百度——高亮，全文检索，搜索推荐

网站用户行为日志——点击、浏览、收藏、评论

商业系统（BI）数据分析、数据挖掘统计

代码托管

elk



# 核心概念

cluster，集群，每个集群至少2个节点

node，一个节点不代表一台服务器，而是一个es程序

field，一个数据字段，与index和type一起可以定位一个doc

document，文档，可以手动指定doc的id

type，逻辑分类，7.x不使用

shard，分片，默认一个索引位5个分片

# es容错机制

1.master选举，从剩余的机器中选择新的master，可能会产生脑裂，解决：discovery.zen.minimum_master_nodes=n/2+1

2.replica容错，将副本分片提升为主分片

3.重启故障机

4.故障机的数据恢复和同步（增量）

![image-20210326101714729](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20210326101714729.png)



# 安装

（前提是需要jdk）

略略略



# es节点角色

## 1.master

master节点一般不做数据节点，每个集群有且只有一个

需要尽量避免mastrer节点的node.data=true

## 2.coordinating协调节点

默认每个节点都是协调节点

## 3.voting投票节点

在es master宕机之后，由投票节点负责重新选举master，默认数据节点数=投票节点数

如果设置了node.voting_only，该节点就只会用于处理竞选，而不会参加竞选

## 4.master-eligible

master候选节点，不存储数据

## 5.machine learning

机器学习节点

## 6.ingest

数据过滤节点

# 集群健康值检查

1.使用http://localhost:9200/_cluster/health检查健康值

使用以下命令：

```
GET /_cluster/health 
或 
GET / _cat/health/v
```

返回结果：

```json
{
    "cluster_name": "elasticsearch",
    "status": "green", #健康状况
    "timed_out": false,	#超时时间
    "number_of_nodes": 1,	#节点总数
    "number_of_data_nodes": 1,	#数据节点数量
    "active_primary_shards": 0,	#活跃主分片
    "active_shards": 0,#活跃分片
    "relocating_shards": 0,
    "initializing_shards": 0,
    "unassigned_shards": 0,
    "delayed_unassigned_shards": 0,
    "number_of_pending_tasks": 0,
    "number_of_in_flight_fetch": 0,
    "task_max_waiting_in_queue_millis": 0,
    "active_shards_percent_as_number": 100
}
```



## green

集群中primaryshard和replica shard都是active

## yellow

集群中至少一个replica shard不可用

## red

集群中至少有一个primary shard不可用



# CRUD

## 1.添加索引

PUT /${index}?pretty



## 2.查看所有索引

GET _cat/indices/v 



## 3.删除索引

DELETE /${index}?pretty

```
es删除数据后会在一定时间内保存旧的数据，使用版本号来保留
```



## 4.插入或更新数据

```
# 可以用于添加或修改，但是对于修改来说，必须传所有字段，否则不传的字段会被更新为null
PUT /${index}/_doc/${id}
#用于更新
POST /${index}/_doc/${id}/_update
{
	"doc":{
		"price":3999
	}
}
```



## 5.查询数据

```
GET /${index}/_doc/{$id}

GET /${index}/_search?sort=price:asc&name=iPhone

GET /${index}/_search
{
	"query":{
		"match":{
			"name":"haha"
		}
	}
}
```



## 6.查看索引信息

GET /${index}?pretty

