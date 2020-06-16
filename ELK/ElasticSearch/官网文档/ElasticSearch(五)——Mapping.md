# ElasticSearch(五)——Mapping

## 一、Mapping

Mapping是定义如何存储和索引文档及其包含的字段的过程。例如，使用映射来定义：

1.哪些字符串字段应该作为全文字段处理

2.哪些字段包含数字、日期或地理位置

3.日期格式化处理

4.控制动态添加字段的映射的自定义规则

一个mapping定义包含两部分：**Meta-fields**和**Fields or Properties**。

Meta-fields用于定制如何处理与文档相关的元数据，包括文档的_index、_id和_source字段。

Fields or Properties包含与文档相关的字段或属性列表。



## 二、Field数据类型

每个字段有一个数据类型，可以是：

1.简单类型：text、keyword、date、long、double、boolean或ip

2.一种支持JSON分层特性的类型，比如object或nested

3.geo_point、geo_shape或completion等专门化类型



以不同的目的使用不同的方式索引同一个字段通常很有用。例如，字符串字段可以作为text字段进行索引，以便进行全文搜索，也可以作为keyword字段进行索引，以便进行排序或聚合。或者，可以使用标准分析器、英语分析器和法语分析器为字符串字段建立索引。



### 防止映射爆炸

在索引中定义太多字段会导致映射爆炸，从而导致内存不足错误和难以恢复的情况。这个问题可能比预期的更常见。例如，考虑这样一种情况:插入的每个新文档都会引入新字段。这在动态映射中非常常见。每次文档包含新字段时，这些字段都会出现在索引的映射中。对于少量数据来说，这并不令人担忧，但随着映射的增长，这可能会成为一个问题。以下设置允许限制field的数量

**index.mapping.total_fields.limit**

fields的最大数量，默认为1000

**index.mapping.depth.limit**

字段的最大深度，用内部对象的数量来衡量。例如，如果所有字段都在根对象级别定义，则深度为1。如果存在一个对象映射，那么深度就是2，以此类推。默认值是20。

**index.mapping.nested_fields.limit**

索引中不同嵌套映射的最大数目，默认为50。

**index.mapping.nested_objects.limit**

跨所有嵌套类型的单个文档内嵌套JSON对象的最大数量，默认为10000

**index.mapping.field_name_length.limit**

设置字段名的最大长度。默认值是Long。MAX_VALUE(没有限制)。这个设置实际上并不能解决映射爆炸问题，但是如果您想限制字段长度，它仍然很有用。通常不需要设置此设置。除非用户开始添加大量名称很长的字段，否则默认设置是可以的。



## 三、动态映射

字段和映射类型在使用之前不需要定义。由于有了动态映射，新的字段名称将自动添加，只需索引一个文档。可以向顶级映射类型、内部对象和嵌套字段添加新字段。



## 四、显式映射

你对数据的了解要比Elasticsearch所能猜测的要多，因此尽管动态映射在开始时可能很有用，但在某些时候你可能希望指定自己的显式映射。

你可以在创建索引或者向索引新增字段时创建fields

#### **创建索引**

```
PUT /my-index
{
  "mappings": {
    "properties": {
      "age":    { "type": "integer" },  
      "email":  { "type": "keyword"  }, 
      "name":   { "type": "text"  }     
    }
  }
}
```

#### **新增字段**

```
PUT /my-index/_mapping
{
  "properties": {
    "employee-id": {
      "type": "keyword",
      "index": false
    }
  }
}
```



#### 注意事项

除了受支持的映射参数外，你不能更改现有字段的映射或字段类型。更改现有字段可能会使已经索引的数据无效。如果需要更改字段的映射，请创建具有正确映射的新索引，并将数据重新索引到该索引中。重命名字段会使已在旧字段名称下建立索引的数据无效。相反，添加别名字段来创建替代字段名称。



## 五、获取映射

```console
GET /my-index/_mapping
```