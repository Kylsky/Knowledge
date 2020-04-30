# ElasticSearch(四)aggregations

elastic search的聚合操作有点和mysql的聚合函数的概念有点像。Elasticsearch aggregations使你能够获得关于搜索结果的元信息，并回答诸如“德克萨斯州有多少帐户持有人”或“田纳西州的平均帐户余额是多少”之类的问题。你可以在一个请求中搜索文档、过滤搜索结果并使用聚合来分析结果。官网对聚合只做了简单的介绍，不过也足够了。

## 一、terms聚合

例如，下面的请求使用**term**聚合按**state**对**bank**索引中的所有帐户进行分组，并按降序返回帐户最多的十个州:

```
curl -X GET "localhost:9200/bank/_search?pretty" -H 'Content-Type: application/json' -d'
{
  "size": 0,
  "aggs": {
  	//这是一个声名，表示用来通过state进行分组
    "group_by_state": {
      //tems聚合类型
      "terms": {
        //字段
        "field": "state.keyword"
      }
    }
  }
}
'
```

以下是返回值：

```
{
  "took": 29,
  "timed_out": false,
  //分片信息
  "_shards": {
    "total": 1,
    "successful": 1,
    "skipped" : 0,
    "failed": 0
  },
  //匹配到的文档
  "hits" : {
     "total" : {
        "value": 1000,
        "relation": "eq"
     },
    "max_score" : null,
    "hits" : [ ]
  },
  //聚合结果
  "aggregations" : {
    "group_by_state" : {
      "doc_count_error_upper_bound": 20,
      "sum_other_doc_count": 770,
      "buckets" : [ {
        "key" : "ID",
        "doc_count" : 27
      }, {
        "key" : "TX",
        "doc_count" : 27
      }, {
        "key" : "AL",
        "doc_count" : 25
      }, {
        "key" : "MD",
        "doc_count" : 25
      }, {
        "key" : "TN",
        "doc_count" : 23
      }, {
        "key" : "MA",
        "doc_count" : 21
      }, {
        "key" : "NC",
        "doc_count" : 21
      }, {
        "key" : "ND",
        "doc_count" : 21
      }, {
        "key" : "ME",
        "doc_count" : 20
      }, {
        "key" : "MO",
        "doc_count" : 20
      } ]
    }
  }
}
```

响应中的bucket属性展示了state字段名(key)以及对应的文档数量(doc_count),因为请求中set大小=0，所以响应只包含聚合结果。



## 二、avg聚合

可以组合聚合来构建更复杂的数据摘要。例如，下面的请求在先前的terms聚合中嵌套一个avg聚合，以计算每个状态的平均余额。

```
curl -X GET "localhost:9200/bank/_search?pretty" -H 'Content-Type: application/json' -d'
{
  "size": 0,
  "aggs": {
    "group_by_state": {
      "terms": {
        "field": "state.keyword"
      },
      "aggs": {
        "average_balance": {
          "avg": {
            "field": "balance"
          }
        }
      }
    }
  }
}
'
```

可以使用嵌套地avg聚合的结果进行排序，而不是通过state数量对结果进行排序，方法是在聚合项中指定顺序:

```
curl -X GET "localhost:9200/bank/_search?pretty" -H 'Content-Type: application/json' -d'
{
  "size": 0,
  "aggs": {
    "group_by_state": {
      "terms": {
        "field": "state.keyword",
        "order": {
          "average_balance": "desc"
        }
      },
      "aggs": {
        "average_balance": {
          "avg": {
            "field": "balance"
          }
        }
      }
    }
  }
}
'
```

除了这些基本的过滤和度量聚合之外，Elasticsearch还提供了专门的聚合，用于操作多个字段和分析特定类型的数据，如日期、IP地址和地理数据。您还可以将单个聚合的结果提供给管道聚合以进行进一步分析。

聚合提供的核心分析功能支持使用机器学习等高级功能来检测异常。