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



### 2.2 方法一——导包&敲代码

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

#### 3.1