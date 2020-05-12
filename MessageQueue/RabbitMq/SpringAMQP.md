# SpringAMQP

翻译自Spring官网

```
https://docs.spring.io/spring-amqp/docs/2.2.6.RELEASE/reference/html/
```

## 1.前言

Spring AMQP项目将核心Spring概念应用于基于AMQP的消息传递解决方案的开发。我们提供了一个“模板”作为发送和接收消息的高级抽象。我们还为消息驱动的pojo提供支持。这些库促进了AMQP资源的管理，同时促进了依赖项注入和声明式配置的使用。在所有这些情况下，您都可以看到与Spring框架中的JMS支持的相似之处



## 2.懒人指引

### 2.1 导入依赖

```xml
<dependency>
  <groupId>org.springframework.amqp</groupId>
  <artifactId>spring-rabbit</artifactId>
  <version>2.2.6.RELEASE</version>
</dependency>
```



### 2.2 方法一——导包&代码

```java
import org.springframework.amqp.core.AmqpAdmin;
import org.springframework.amqp.core.AmqpTemplate;
import org.springframework.amqp.core.Queue;
import org.springframework.amqp.rabbit.connection.CachingConnectionFactory;
import org.springframework.amqp.rabbit.connection.ConnectionFactory;
import org.springframework.amqp.rabbit.core.RabbitAdmin;
import org.springframework.amqp.rabbit.core.RabbitTemplate;

//连接工厂
ConnectionFactory connectionFactory = new CachingConnectionFactory();
//创建连接Connection，开辟信道Channel
AmqpAdmin admin = new RabbitAdmin(connectionFactory);
//声明队列
admin.declareQueue(new Queue("myqueue"));
//获取模板
AmqpTemplate template = new RabbitTemplate(connectionFactory);
//发送消息
template.convertAndSend("myqueue", "foo");
//接收消息
String foo = (String) template.receiveAndConvert("myqueue");
```



### 2.3 方法二——基于注解的配置

显而易见，使用配置的方式声明了ConnectionFactory和AMQPFactory等信息减少了代码耦合度

```java
ApplicationContext context =
    new AnnotationConfigApplicationContext(RabbitConfiguration.class);
AmqpTemplate template = context.getBean(AmqpTemplate.class);
template.convertAndSend("myqueue", "foo");
String foo = (String) template.receiveAndConvert("myqueue");

........

@Configuration
public class RabbitConfiguration {

    @Bean
    public ConnectionFactory connectionFactory() {
        return new CachingConnectionFactory("localhost");
    }

    @Bean
    public AmqpAdmin amqpAdmin() {
        return new RabbitAdmin(connectionFactory());
    }

    @Bean
    public RabbitTemplate rabbitTemplate() {
        return new RabbitTemplate(connectionFactory());
    }

    @Bean
    public Queue myQueue() {
       return new Queue("myqueue");
    }
}
```



### 2.3 方法三——springboot快速配置

是springboot在启动时发送消息

```java
@SpringBootApplication
public class Application {

    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }

    @Bean
    public ApplicationRunner runner(AmqpTemplate template) {
        return args -> template.convertAndSend("myqueue", "foo");
    }

    @Bean
    public Queue myQueue() {
        return new Queue("myqueue");
    }

    @RabbitListener(queues = "myqueue")
    public void listen(String in) {
        System.out.println(in);
    }

}
```



## 3.使用

#### 3.1 大纲

springAMQP包含了两个模块：1.spring-amqp；2.spring-rabbit。

前者包含了**org.springframework.amqp.core**包，这个包里可以找到amqp核心类，spring的目的是提供不依赖于任何特定AMQP代理实现或客户机库的通用抽象。最终用户代码可以更容易地跨供应商实现移植，因为它可以只针对抽象层进行开发。以下是springrabbit的相关实体类



##### **1.MESSAGE**

AMQP中并没有定义消息实体类，所有的消息都是用字节数组通过basicPublish发送的，同时消息意外的其他属性作为单独的property传递。

springAMQP定义了一个Message实体类作为消息模型，目的是为了将消息体body和属性property封装在单个实例中，从而简化API。其定义如下:

```java
public class Message {
    //请求参数
    private final MessageProperties messageProperties;
    //请求体
    private final byte[] body;

    public Message(byte[] body, MessageProperties messageProperties) {
        this.body = body;
        this.messageProperties = messageProperties;
    }

    public byte[] getBody() {
        return this.body;
    }

    public MessageProperties getMessageProperties() {
        return this.messageProperties;
    }
}
```

MessageProperties接口定义了一些通用的属性，如以下：

```
messageId
timestamp
contentType
……
```

同时也可以通过自定义请求头来扩展这些属性：

```
 setHeader(String key, Object value)
```



##### **2.EXCHANGE**

Exchange接口表示AMQP交换机概念，这是消息生成器发送到的对象。代理的虚拟主机中的每个交换器都有一个惟一的名称和一些其他属性。下面的例子展示了Exchange接口:

```java
public interface Exchange {

    String getName();

    String getExchangeType();

    boolean isDurable();

    boolean isAutoDelete();

    Map<String, Object> getArguments();

}
```

Exchange还有一个“type”属性，由ExchangeTypes中定义的常量表示

```java
public abstract class ExchangeTypes {
    public static final String DIRECT = "direct";
    public static final String TOPIC = "topic";
    public static final String FANOUT = "fanout";
    public static final String HEADERS = "headers";
    public static final String SYSTEM = "system";

    public ExchangeTypes() {
    }
}
```



##### **3.Queue**

Queue类表示消息使用者从其中接收消息的组件。与各种Exchange类一样，我们的实现旨在成为这个核心AMQP类型的抽象表示。下面的代码展示了Queue类：

```java
public class Queue  {

    private final String name;

    private volatile boolean durable;

    private volatile boolean exclusive;

    private volatile boolean autoDelete;

    private volatile Map<String, Object> arguments;

    public Queue(String name) {
        this(name, true, false, false);
    }
}
```

Queue的构造器接收一个队列名并调用重载构造方法生成队列，默认的三个boolean值分别表示持久化的，非独有的,不自动删除的。相反，若需要定义一个临时队列，则需要false，true，true来表示临时的、独有的、自动删除的。



##### **4.Binding**

将队列连接到交换器的绑定对于通过消息传递将生产者和消费者连接起来至关重要。在Spring AMQP中，我们定义了一个绑定类来表示这些连接。

```
//direct exchange
new Binding(someQueue, someDirectExchange, "foo.bar");

//topic exchange
new Binding(someQueue, someTopicExchange, "foo.*");

/fanout exchange
new Binding(someQueue, someFanoutExchange);
```

另外springamqp还通过Builder来提供方便的api，同时由于bind方法是static的所以性能更好(误)。

一个Binding只和一个Connection关联，因此可以说它并不灵活。不过在之后的AmqpAdmin类和AmqpTemplate中会有更灵活的使用。



### 3.2连接管理

尽管我们在前一节中描述的AMQP模型是通用的，适用于所有的实现，但是资源管理这部分功能是依赖特定的实现的，如rabbitmq。

连接即Connection，rabbitmq的核心连接管理组件是**ConnectionFactory**接口，顾名思义，这是一个用来提供连接的工厂，实例为org.springframework.amqp.rabbit.connection.**Connection**类。该接口的具体实现目前只有一个，叫做**CachingConnectionFacroty**，它通过建立一个**单一的**可由应用程序共享的连接代理来进行工作。之所以只需要一个Connection，是因为aqmp的最小通信单元为channel，且一个connection可以开辟多个channel(这和JMS中的connection和session有点像)。

connection对象拥有一个createChannel方法，它根据channel是否是事务性的来为其维护单独的缓存。在创建ConnectionFactory时，可以通过构造方法提供hostname、username和password，并通过调用setChannelCacheSize方法设置channel缓存数量(默认值为25) 

从1.3版本后，可以通过配置CachingConnectionFacroty缓存connections(之前只能缓存chanel好像)，这样的话，每次调用creatConnection创建新的连接，但是在close时则并不会将其关闭，而是将其置入缓存中(如果缓存满了还是会关闭的)。对connection的缓存主要适用场景可以是HA cluster(高可用集群)，请求通过负载均衡连接到不同集群实例的connection，这样能有效提高集群可用性及性能。开启connection缓存只需设置cacheMode为CacheMode.CONNECTION即可。

```
需要注意的是，当connection缓存开启时，是不支持自动声明exchange、queue和binding的。
同时，在写数据时，amqp-client会为每一个connection创建一个线程池，大小是cpu核心数的两倍。当connection数庞大时，开发者应该使用Executor来管理线程池。这个Executor可以被多个Connection共享。
执行程序的线程池应该是无界的，或者为预期的使用进行适当的设置(通常，每个连接至少有一个线程)。如果在每个连接上创建多个通道，池的大小会影响并发性。
因此，使用变量(或简单缓存)线程池执行器是最合适的。
```

有一点很重要，那就是cache的大小并不是说只允许多少channel存在，而是指能缓存的channel数量。假设cache为10，那么则代表10个以内的channel都可以被使用。若channel超过10了，则只会有10个进行工作，超过10个的会被关闭。

1.6版本后，默认cache容量从1加到了25。对于一个高容量，多现成的环境来说，一个小容量的cache会导致channel频繁创建和关闭，而增加cache容量则能避免这个问题。如果发现channel的频繁变化，则需要考虑在AdminUI中调整大小。另外，cache只需要在必要扩容的时候扩容。

1.4.2版本后，CachingConnectionFactory  有一个channelCheckOutTImeout的属性，当该属性值大于0，channelCacheSize则会表示每个connection允许创建的最大channel数量。当channel到达最大值，则会阻塞线程，直到channel可用或者超时(超时时会抛出异常)

以下代码演示了Connection的创建：

```java
CachingConnectionFactory connectionFactory = new CachingConnectionFactory("somehost");
connectionFactory.setUsername("guest");
connectionFactory.setPassword("guest");

Connection connection = connectionFactory.createConnection();
```



#### Connection命名

为了提高可读性，1.7版本以后通过ConnectionNameStrategy支持为connection命名，代码如下：

```java
connectionFactory.setConnectionNameStrategy(connectionFactory -> "MY_CONNECTION");
```

也可以通过实现类SimplePropertyValueConnectionNameStrategy命名，如:

```java
@Bean
public ConnectionNameStrategy cns() {
    return new SimplePropertyValueConnectionNameStrategy("spring.application.name");
}

@Bean
public ConnectionFactory rabbitConnectionFactory(ConnectionNameStrategy cns) {
    CachingConnectionFactory connectionFactory = new CachingConnectionFactory();
    ...
    connectionFactory.setConnectionNameStrategy(cns);
    return connectionFactory;
}
```



#### 阻塞的连接以及资源控制

连接可能因为与服务器内存警报做交互而阻塞。从2.0版本开始，Connection可由BlockedListener提供被通知连接阻塞和非阻塞的事件。此外，AbstractConnectionFactory通过该内部监听器(BlockedListener)发出ConnectionBlockedEvent和ConnectionUnblockedEvent事件，这使开发者能够提供应用程序逻辑来对代理上的问题做出适当的反应，并(例如)采取一些纠正措施。



### 3.3添加定制连接属性

```java
connectionFactory.getRabbitConnectionFactory().getClientProperties().put("thing1", "thing2");
```



### 3.4AmqpTemplate

这是一个接口，定义了发送和接收消息的API，它的唯一实现类是RabbitTemplate。

#### 重试功能

1.3本本后，通过配置RabbitTemplate可以使用RetryTemplate来处理连接出现问题的情况。以下的代码为使用指数后退策略和默认的SimpleRetryPolicy：

```java
@Bean
public AmqpTemplate rabbitTemplate() {
    RabbitTemplate template = new RabbitTemplate(connectionFactory());
    RetryTemplate retryTemplate = new RetryTemplate();
    //指数后退策略
    ExponentialBackOffPolicy backOffPolicy = new ExponentialBackOffPolicy();
    backOffPolicy.setInitialInterval(500);
    backOffPolicy.setMultiplier(10.0);
    backOffPolicy.setMaxInterval(10000);
    retryTemplate.setBackOffPolicy(backOffPolicy);
    //配置重试模板
    template.setRetryTemplate(retryTemplate);
    return template;
}
```



#### 异步检查结果

发布消息是通过异步机制实现的，默认情况下，不能被正确路由的消息会被rabbitmq丢弃。消息成功发布则发送者会接收到一个异步的确认。以下是两种发送失败的场景

```
1.发送到exchange了，但是找不到指定的queue
2.找不到指定的exchange
```

第一种情况会rturn，第二种消息会被丢弃，不会有return，可以通过在CachingConnectionFactory注册一个ChannelListener来处理这样的事件。

```java
this.connectionFactory.addConnectionListener(new ConnectionListener() {

    @Override
    public void onCreate(Connection connection) {
    }

    @Override
    public void onShutDown(ShutdownSignalException signal) {
        ...
    }

});
```



#### publisher的消息确认及返回

要能获取returnedmessage，template需要设置mandatory为true，或者mandatory-expression为true。同时需要CachingConnectionFactory的publisherreturns属性为true。同时需要template重写returnedMessages方法：

```java
void returnedMessage(Message message, int replyCode, String replyText,
          String exchange, String routingKey);
```



### 3.5 发送消息

可以使用以下方法发送消息

```java
void send(Message message) throws AmqpException;

void send(String routingKey, Message message) throws AmqpException;

void send(String exchange, String routingKey, Message message) throws AmqpException;
```

由于第三个方法的参数声明最清晰，因此讨论该方法。该方法在运行时获取一个exchange name、routing key和message，一个运行实例如下：

```java
amqpTemplate.send("marketData.topic", "quotes.nasdaq.THING1",
    new Message("12.34".getBytes(), someProperties));
```

或者也可以通过使用第二个方法并额外声明exchange

```java
amqpTemplate.setExchange("marketData.topic");
amqpTemplate.send("quotes.nasdaq.FOO", new Message("12.34".getBytes(), someProperties));
```

若exchange和routingkey都已经声明了，则只需要发送消息即可

```java
amqpTemplate.setExchange("marketData.topic");
amqpTemplate.setRoutingKey("quotes.nasdaq.FOO");
amqpTemplate.send(new Message("12.34".getBytes(), someProperties));
```

#### Message Builder

要编辑一个Message实体类，可以通过MessageBuilder和MessagePropertiesBuilder类来提供API，操作如下：

**1.MessageBuilder**

```java
Message message = MessageBuilder.withBody("foo".getBytes())
    .setContentType(MessageProperties.CONTENT_TYPE_TEXT_PLAIN)
    .setMessageId("123")
    .setHeader("bar", "baz")
    .build();
```

以及一些重要静态方法：

```java
//构建器创建的消息有一个直接引用参数的消息体
public static MessageBuilder withBody(byte[] body) 
//生成的消息内容是参数的副本
public static MessageBuilder withClonedBody(byte[] body) 
//消息体的一部分
public static MessageBuilder withBody(byte[] body, int from, int to) 
//直接赋值Message对象
public static MessageBuilder fromMessage(Message message) 
//赋值Message对象的副本
public static MessageBuilder fromClonedMessage(Message message) 
```

**2.MessagePropertiesBuilder**

```java
MessageProperties props = MessagePropertiesBuilder.newInstance()
    .setContentType(MessageProperties.CONTENT_TYPE_TEXT_PLAIN)
    .setMessageId("123")
    .setHeader("bar", "baz")
    .build();
Message message = MessageBuilder.withBody("foo".getBytes())
    .andProperties(props)
    .build();
```

以及一些重要静态方法：

```java
//新建一个实例，带有默认值
public static MessagePropertiesBuilder newInstance() 
//通过传入的已有properties初始化
public static MessagePropertiesBuilder fromProperties(MessageProperties properties) 
//会新建一个入参的副本作为properties
public static MessagePropertiesBuilder fromClonedProperties(MessageProperties properties) 
```

#### 批量消息发送

1.4.2版本后引入BatchingRabbitTemplate，作为RabbitTemplate的子类，它重写了send方法并通过实现BatchingStrategy支持批量消息发送功能，BatchingStrategy接口定义如下：

```java
public interface BatchingStrategy {

	MessageBatch addToBatch(String exchange, String routingKey, Message message);

	Date nextRelease();

	Collection<MessageBatch> releaseBatches();

}
```

这些批量数据会被放置在内存中，因此，如果系统出现故障，尚未发送的消息可能会丢失。

### 3.6 接收信息

springAMQP有多种接收消息的方法，逐个来看看

#### receive

```java
Message receive() throws AmqpException;

Message receive(String queueName) throws AmqpException;

Message receive(long timeoutMillis) throws AmqpException;

Message receive(String queueName, long timeoutMillis) throws AmqpException;
```

这四个比较简单



#### receiveAndConvert

```java
Object receiveAndConvert() throws AmqpException;

Object receiveAndConvert(String queueName) throws AmqpException;

Object receiveAndConvert(long timeoutMillis) throws AmqpException;

Object receiveAndConvert(String queueName, long timeoutMillis) throws AmqpException;
```

receiveAndConvert方法的返回类型为Object，因此更具有普遍性。在2.0版本以后，这些方法的一些变体使用附加的ParameterizedTypeReference参数来转换复杂类型，这要求template必须配置一个SmartMessageConverter



#### receiveAndReply

```java
<R, S> boolean receiveAndReply(ReceiveAndReplyCallback<R, S> callback)
	   throws AmqpException;

<R, S> boolean receiveAndReply(String queueName, ReceiveAndReplyCallback<R, S> callback)
 	throws AmqpException;

<R, S> boolean receiveAndReply(ReceiveAndReplyCallback<R, S> callback,
	String replyExchange, String replyRoutingKey) throws AmqpException;

<R, S> boolean receiveAndReply(String queueName, ReceiveAndReplyCallback<R, S> callback,
	String replyExchange, String replyRoutingKey) throws AmqpException;

<R, S> boolean receiveAndReply(ReceiveAndReplyCallback<R, S> callback,
 	ReplyToAddressCallback<S> replyToAddressCallback) throws AmqpException;

<R, S> boolean receiveAndReply(String queueName, ReceiveAndReplyCallback<R, S> callback,
			ReplyToAddressCallback<S> replyToAddressCallback) throws AmqpException;
```

1.3版本后，amqpTemplate拥有了以下更方便的方法来异步实现消息接收、处理、确认