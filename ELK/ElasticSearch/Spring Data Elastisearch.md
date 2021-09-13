# Spring Data Elastisearch

## 一、Spring Data Repository

在学习spring data es之前，先来看官网如何介绍spring data repository：

Spring data repository的目标是显著减少为各种持久性存储实现数据访问层所需的样板代码量，很好理解，其实就是帮助开发者更好地进行crud(逃)，看来Spring Data Repository也是基于这个操作来的。

### 核心概念——Repository

spring data repository的核心接口，它以T和ID类型作为泛型来管理。此接口主要用作抽象接口，用于捕获要使用的类型并帮助您发现扩展此接口的接口。CrudRepository为正在管理的实体类提供了复杂的CRUD功能实现：

```java
public interface CrudRepository<T, ID> extends Repository<T, ID> {
  //保存操作
  <S extends T> S save(S entity);      
  //查找
  Optional<T> findById(ID primaryKey); 
  //查找所有
  Iterable<T> findAll();               
  //计数
  long count();                        
  //删除
  void delete(T entity);               
  //判断是否存在
  boolean existsById(ID primaryKey);   

  // … more functionality omitted.
}
```

上面只对spring data repository进行了简单的介绍，以下才是spring data es的主题。



## 二、Elasticsearch Clients

本章说明了所支持的Elasticsearch客户端实现的配置和使用。

### 1.TransportClient

这东西Spring说es7已经报废了，es8会移除，那就不看了……



### 2.High Level REST Client

Elasticsearch的默认客户端，TransportClient的直接替代品，它接受和返回完全相同的请求/响应对象，因此依赖于Elasticsearch核心项目。异步调用是在客户端管理的线程池上操作的，并且要求在请求完成时通知回调。

java配置如下：

```java
import org.springframework.beans.factory.annotation.Autowired;@Configuration
static class Config {

  @Bean
  RestHighLevelClient client() {

    ClientConfiguration clientConfiguration = ClientConfiguration.builder() 
      .connectedTo("localhost:9200", "localhost:9201")
      .build();

    return RestClients.create(clientConfiguration).rest();                  
  }
}

// ...

  @Autowired
  RestHighLevelClient highLevelClient;

  RestClient lowLevelClient = highLevelClient.lowLevelClient();             

// ...

IndexRequest request = new IndexRequest("spring-data", "elasticsearch", randomID())
  .source(singletonMap("feature", "high-level-rest-client"))
  .setRefreshPolicy(IMMEDIATE);

IndexResponse response = highLevelClient.index(request);
```



### 3.Reactive Client

基于WebClient的非官方驱动，它使用Elasticsearch核心项目提供的请求/响应对象。调用直接在响应堆栈上操作，而不是将异步(线程池绑定)响应包装为响应类型。



### 4.Client Configuration

客户端行为可以通过允许设置SSL、连接和套接字超时选项的ClientConfiguration进行更改。

```java
HttpHeaders defaultHeaders = new HttpHeaders();
defaultHeaders.setBasicAuth(USER_NAME, USER_PASS);                      

ClientConfiguration clientConfiguration = ClientConfiguration.builder()
  .connectedTo("localhost:9200", "localhost:9291")                      
  .withConnectTimeout(Duration.ofSeconds(5))                            
  .withSocketTimeout(Duration.ofSeconds(3))                             
  .useSsl()                                                             
  .withDefaultHeaders(defaultHeaders)                                   
  .withBasicAuth(username, password)                                    
  . // ... other options
  .build();
```



## 三、Elasticsearch Object Mapping

spring data es允许开发者通过抽象接口EntityMapper实现的两种映射：

Json Object Mapping和Meta Model Object Mapping

### Json Object Mapping

代码如下，AbstractElasticsearchConfiguration已经通过ElasticsearchConfigurationSupport定义了一个基于Jackson2的entityMapper，

通过重写elasticsearchClient方法获取到client后即可通过Json来做数据映射

```
@Configuration
public class Config extends AbstractElasticsearchConfiguration { 

  @Override
  public RestHighLevelClient elasticsearchClient() {
    return RestClients.create(ClientConfiguration.create("localhost:9200")).rest();
  }
}
```



### Meta Model Object Mapping

基于Meta Model的方法使用类型信息从Elasticsearch读取/写入数据。这允许为特定的类型映射注册转换器实例。

```java
@Configuration
public class Config extends AbstractElasticsearchConfiguration {

  @Override
  public RestHighLevelClient elasticsearchClient() {
    return RestClients.create(ClientConfiguration.create("localhost:9200")).rest()
  }

  @Bean
  @Override
  public EntityMapper entityMapper() {                                 
//覆盖来自ElasticsearchConfigurationSupport的默认EntityMapper并将其公开为bean。
    ElasticsearchEntityMapper entityMapper = new ElasticsearchEntityMapper(
      elasticsearchMappingContext(), new DefaultConversionService()    
    );
   //使用提供的SimpleElasticsearchMappingContext来避免不一致性，并为转换器注册提供一个GenericConversionServicefor。
   entityMapper.setConversions(elasticsearchCustomConversions());     //可选地设置自定义转换(如果适用)
  	return entityMapper;
  }
}
```

**元数据映射的注解**

ElasticsearchEntityMapper可以使用元数据来驱动对象到文档的映射。提供下列注释:

1. @Id。在java字段上标记用于标识目的的字段。
2. @Document。在类上标记，用来指定es中的index，该注解有以下属性：indexName：索引名；type：文档类型，如果不设置，会设置为类名的小写形式；shards：索引的shard数量；replica：索引的replica数量……
3. @Transient。默认情况下，所有私有字段都映射到文档，该注释将应用它的字段从数据库中排除
4. @PersistenceConstructor。标记一个给定的构造函数，甚至是protected类型的，以便在从数据库实例化对象时使用。构造函数参数通过名称映射到检索到的文档中的键值。
5. @Field。应用于字段级，并定义字段的属性。name：es字段名，若不指定，则为字段名；type：es数据类型，请看es字段类型文章；format、pattern：日期格式；store：标记字段默认值是否应该存储在Elasticsearch中，默认为false……



## 四、Mapping Rules

### Type Hints

class类型映射。映射使用嵌入到发送到服务器的文档中的类型提示来允许泛型类型映射。这些类型提示在文档中表示为_class属性，并为每个聚合根编写。这段话翻译过来有点僵硬，放官网的两个代码就能理解：

```
//实体类
public class Person {              
  @Id String id;
  String firstname;
  String lastname;
}
```

```
//映射结果
{
  "_class" : "com.example.Person", 
  "id" : "cb7bef",
  "firstname" : "Sarah",
  "lastname" : "Connor"
}
```

默认的，类名会被用来做es中的type，为了能够自定义type名，使用@TypeAlias来自定义：

```
@TypeAlias("human")                
public class Person {
  @Id String id;
}
```



#### Geospatial Types

地理空间类型映射。地理空间类型，如Point类型和GeoPoint类型被转换成lat/lon对。

```java
public class Address {
  String city, street;
  Point location;
}
```

```json
{
  "city" : "Los Angeles",
  "street" : "2800 East Observatory Road",
  "location" : { "lat" : 34.118347, "lon" : -118.3026284 }
}
```



### Collections

集合类型映射，直接看代码

```
public class Person {
  List<Person> friends;
}
```

```
{
  "friends" : [ { "firstname" : "Kyle", "lastname" : "Reese" } ]
}
```



### Maps

map映射，直接看代码

```java
public class Person {
  Map<String, Address> knownLocations;
}
```

```json
{
  // ...
  "knownLocations" : {
    "arrivedAt" : {
       "city" : "Los Angeles",
       "street" : "2800 East Observatory Road",
       "location" : { "lat" : 34.118347, "lon" : -118.3026284 }
     }
  }
}
```



### 自定义转换

回顾之前的代码：ElasticsearchCustomConversions允许实现特定的规则来映射实体类和简单类型

```java
@Configuration
public class Config extends AbstractElasticsearchConfiguration {

  @Override
  public RestHighLevelClient elasticsearchClient() {
    return RestClients.create(ClientConfiguration.create("localhost:9200")).rest();
  }

  @Bean
  @Override
  public EntityMapper entityMapper() {

    ElasticsearchEntityMapper entityMapper = new ElasticsearchEntityMapper(
      elasticsearchMappingContext(), new DefaultConversionService());
    entityMapper.setConversions(elasticsearchCustomConversions());  

  	return entityMapper;
  }

  @Bean
  @Override
  public ElasticsearchCustomConversions elasticsearchCustomConversions() {
    return new ElasticsearchCustomConversions(
      Arrays.asList(new AddressToMap(), new MapToAddress()));       
  }

  @WritingConverter                                                 
  static class AddressToMap implements Converter<Address, Map<String, Object>> {

    @Override
    public Map<String, Object> convert(Address source) {

      LinkedHashMap<String, Object> target = new LinkedHashMap<>();
      target.put("ciudad", source.getCity());
      // ...

      return target;
    }
  }

  @ReadingConverter                                            
  static class MapToAddress implements Converter<Map<String, Object>, Address> {

    @Override
    public Address convert(Map<String, Object> source) {

      // ...
      return address;
    }
  }
}
```

```
{
  "ciudad" : "Los Angeles",
  "calle" : "2800 East Observatory Road",
  "localidad" : { "lat" : 34.118347, "lon" : -118.3026284 }
}
```



## 五、Elasticsearch Operations

Spring Data Elasticsearch使用两个接口来定义可以针对Elasticsearch索引调用的操作：**ElasticsearchOperations**、**ReactiveElasticsearchOperations**，第一种方法与传统的同步实现一起使用，而第二种方法使用响应式基础结构。这两种默认的接口实现了以下功能：

1. 实体类的es读写

2. 丰富的api

3. 资源管理和异常转换

   

### ElasticsearchOperations

#### **#1.ElasticsearchTemplate**

ElasticsearchOperations 的实现类，使用的是Transport Client，前情提要，es8会剔除Transport Client，所以这个template生死未卜，不过还是来看看怎么实现的把：

```java
@Configuration
public class TransportClientConfig extends ElasticsearchConfigurationSupport {

  @Bean
  public Client elasticsearchClient() throws UnknownHostException {                 
    Settings settings = Settings.builder().put("cluster.name", "elasticsearch").build();
    TransportClient client = new PreBuiltTransportClient(settings);
    client.addTransportAddress(new TransportAddress(InetAddress.getByName("127.0.0.1"), 9300));
    return client;
  }

  @Bean(name = {"elasticsearchOperations", "elasticsearchTemplate"})
  public ElasticsearchTemplate elasticsearchTemplate() throws UnknownHostException { 
  	return new ElasticsearchTemplate(elasticsearchClient(), entityMapper());
  }

  // use the ElasticsearchEntityMapper
  @Bean
  @Override
  public EntityMapper entityMapper() {                                               
    ElasticsearchEntityMapper entityMapper = new ElasticsearchEntityMapper(elasticsearchMappingContext(),
  	  new DefaultConversionService());
    entityMapper.setConversions(elasticsearchCustomConversions());
    return entityMapper;
  }
}
```



#### **2.ElasticsearchRestTemplate**

ElasticsearchOperations的实现类，使用的是High Level Client。

```java
@Configuration
public class RestClientConfig extends AbstractElasticsearchConfiguration {
  @Override
  public RestHighLevelClient elasticsearchClient() {       
    return RestClients.create(ClientConfiguration.localhost()).rest();
  }

  // 基类AbstractElasticsearchConfiguration已经提供了elasticsearchRestTemplate bean，因此不需要创建           

  // use the ElasticsearchEntityMapper
  @Bean
  @Override
  public EntityMapper entityMapper() {                     
    ElasticsearchEntityMapper entityMapper = new ElasticsearchEntityMapper(elasticsearchMappingContext(),
        new DefaultConversionService());
    entityMapper.setConversions(elasticsearchCustomConversions());

    return entityMapper;
  }
}
```





### Reactive Elasticsearch Operations

**Reactive Elasticsearch Template**

默认实现方式。使用[Reactive Client](https://docs.spring.io/spring-data/elasticsearch/docs/3.2.7.RELEASE/reference/html/#elasticsearch.clients.reactive)

ReactiveElasticsearchTemplate使用用例：

```
@Document(indexName = "marvel", type = "characters")
public class Person {

  private @Id String id;
  private String name;
  private int age;
  // Getter/Setter omitted...
}
```

```java
template.save(new Person("Bruce Banner", 42))                    	 //save后打印
  .doOnNext(System.out::println)
  .flatMap(person -> template.findById(person.id, Person.class)) 
    //查找后打印
  .doOnNext(System.out::println)
  .flatMap(person -> template.delete(person))                       //删除后打印
  .doOnNext(System.out::println)
  .flatMap(id -> template.count(Person.class))                     //计数后打印
  .doOnNext(System.out::println)
  .subscribe(); 
```



## 六、Elasticsearch Repositories

这张包含了Elasticsearch存储库实现的细节。

### Query检索策略

Elasticsearch模块支持所有基本的查询构建特性，如字符串查询、本地搜索查询、基于条件的查询，或者根据方法的查询。

从方法名派生查询并不总是足够的，并且/或者可能导致不可读的方法名。在这种情况下，可以使用@Query注释，如

```java
@Query("{\"match\": {\"name\": {\"query\": \"?0\"}}}")
```

### 创建Query

通常，Elasticsearch的查询按照查询方法中的参数来执行。下面是一个关于Elasticsearch查询方法的简短例子:

```java
interface BookRepository extends Repository<Book, String> {
  List<Book> findByNameAndPrice(String name, Integer price);
}
```

相对的请求为：

```json
{
    "query": {
        "bool" : {
            "must" : [
                { "query_string" : { "query" : "?", "fields" : [ "name" ] } },
                { "query_string" : { "query" : "?", "fields" : [ "price" ] } }
            ]
        }
    }
}
```

