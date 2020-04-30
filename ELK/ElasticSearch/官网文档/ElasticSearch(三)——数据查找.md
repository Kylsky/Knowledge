# ElasticSearch(三)——数据查询

## 一、简单查询

一旦将一些数据放入到Elasticsearch索引中，就可以通过向_search端点发送请求来搜索它。要访问整套搜索功能，可以使用Elasticsearch Query DSL来指定请求体中的搜索条件。在请求URI中指定要搜索的索引的名称。

例如，下面的请求检索按帐号排序的银行索引中的所有文档:

```
curl -X GET "localhost:9200/bank/_search?pretty" -H 'Content-Type: application/json' -d'
{
  "query": { "match_all": {} },
  "sort": [
    { "account_number": "asc" }
  ]
}
'
```

默认情况下，响应的hits部分包含匹配搜索条件的前10个文档，相应比较长，就不贴出来了

响应还提供了以下关于请求的信息:

**took**：Elasticsearch运行查询需要多长时间(以毫秒为单位)

**timed_out**：请求是否超时

**_shards**：搜索了多少分片，并对多少分片成功、失败或跳过进行了细分。

**max_score**：找到最相关的文档的得分

**hits.total.value**：找到了多少匹配的文档

**hits.sort**：文档的排序位置(当不根据相关性得分排序时)

**hits._score**：文档的相关性评分(在使用match_all时不适用)

每个搜索请求都是自包含的:Elasticsearch不跨请求维护任何状态信息。要浏览搜索结果，请在请求中指定from和size参数。

例如，下面的请求从10次到19次:

```
curl -X GET "localhost:9200/bank/_search?pretty" -H 'Content-Type: application/json' -d'
{
  "query": { "match_all": {} },
  "sort": [
    { "account_number": "asc" }
  ],
  "from": 10,
  "size": 10
}
'
```



## 二、复杂查询

现在你已经了解了如何提交基本的搜索请求，可以开始构建比match_all更有趣的查询。要在字段中搜索特定的术语，可以使用match查询。例如，下面的请求搜索地址字段，以查找地址包含mill或lane的客户:

### match

```
curl -X GET "localhost:9200/bank/_search?pretty" -H 'Content-Type: application/json' -d'
{
  "query": { "match": { "address": "mill lane" } }
}
'
```

### match_phrase

要执行短语搜索而不是匹配单个词汇，可以使用match_phrase而不是match。例如，下面的请求只匹配包含短语mill lane的地址:

```
curl -X GET "localhost:9200/bank/_search?pretty" -H 'Content-Type: application/json' -d'
{
  "query": { "match_phrase": { "address": "mill lane" } }
}
'
```

### bool

要构造更复杂的查询，可以使用bool查询来组合多个查询条件。可以指定需要(必须匹配)、需要(应该匹配)或不需要(必须不匹配)的条件。

```
curl -X GET "localhost:9200/bank/_search?pretty" -H 'Content-Type: application/json' -d'
{
  "query": {
    "bool": {
      "must": [
        { "match": { "age": "40" } }
      ],
      "must_not": [
        { "match": { "state": "ID" } }
      ]
    }
  }
}
'
```

每个must、should和must_not元素都被称为查询子句。文档符合每个“必须”或“应该”子句中的标准的程度决定了文档的相关性评分。得分越高，文档越符合您的搜索条件。默认情况下，Elasticsearch返回根据这些相关性得分排序的文档。

例如，下面的请求使用范围筛选器将结果限制到余额在$20,000到$30,000(包括)之间的帐户：

```
curl -X GET "localhost:9200/bank/_search?pretty" -H 'Content-Type: application/json' -d'
{
  "query": {
    "bool": {
      "must": { "match_all": {} },
      "filter": {
        "range": {
          "balance": {
            "gte": 20000,
            "lte": 30000
          }
        }
      }
    }
  }
}
'
```

