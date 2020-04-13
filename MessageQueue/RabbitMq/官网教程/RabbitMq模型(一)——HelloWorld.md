# Hello RabbitMq

在网上找了很多rabbitMq的教程，发现还不如跟着官网学"第一手资料"，先贴出链接，有需要可以自己看，官方从消息队列的五种模型引出了具体的学习指导

```
http://next.rabbitmq.com/tutorials/tutorial-one-java.html
```

本篇先从最简单的消息队列形式及相关代码演示开始讲起



## 一、消息队列模型

可以看到最简单的消息队列模型为一个生产者-一条消息队列-一个消费者

![img](http://kylescloud.top/site/pic/rabbitMq-HelloWorld.png)





## 二、代码演示

### 1.引入相关依赖

这里采用maven的形式引入依赖

```
<!-- https://mvnrepository.com/artifact/com.rabbitmq/amqp-client -->
<dependency>
    <groupId>com.rabbitmq</groupId>
    <artifactId>amqp-client</artifactId>
    <version>5.7.1</version>
</dependency>
```



### 2.编写代码

**消息生产者**

```java
import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.ConnectionFactory;

import java.nio.charset.StandardCharsets;

public class Send {

    private final static String QUEUE_NAME = "hello";

    public static void main(String[] argv) throws Exception {
        ConnectionFactory factory = new ConnectionFactory();
        //设置主机
        factory.setHost("localhost");
        //设置端口
        factory.setPort(5672);
        //设置用户名密码
        factory.setUsername("Kyle");
        factory.setPassword("123456");
        //建立连接
        try (Connection connection = factory.newConnection();
             //创建信道	
            Channel channel = connection.createChannel()) {
            channel.queueDeclare(QUEUE_NAME, false, false, false, null);
            //设置消息
            String message = "Hello World!";
            //发送消息
            channel.basicPublish("", QUEUE_NAME, null, message.getBytes(StandardCharsets.UTF_8));
            System.out.println(" [x] Sent '" + message + "'");
        }
    }
}
```

**消息消费者**

```java
package testRabbitMq;

import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.ConnectionFactory;
import com.rabbitmq.client.DeliverCallback;

import java.nio.charset.StandardCharsets;

public class Recv {

    private final static String QUEUE_NAME = "hello";

    public static void main(String[] argv) throws Exception {
        ConnectionFactory factory = new ConnectionFactory();
        factory.setHost("localhost");
        //设置用户名密码
        factory.setUsername("Kyle");
        factory.setPassword("123456");
        //建立连接
        Connection connection = factory.newConnection();
        //创建信道
        Channel channel = connection.createChannel();
		//声明队列名
        channel.queueDeclare(QUEUE_NAME, false, false, false, null);
        System.out.println(" [*] Waiting for messages. To exit press CTRL+C");
		//通过回调获取消息
        DeliverCallback deliverCallback = (consumerTag, delivery) -> {
            String message = new String(delivery.getBody(), "UTF-8");
            System.out.println(" [x] Received '" + message + "'");
        };
        //对消息进行消费
        channel.basicConsume(QUEUE_NAME, true, deliverCallback, consumerTag -> {
        });
    }
}
```



# 三、总结

初次使用java操作rabbitmq，可以发现里面的许多api基本不知道做了啥，不过这也没关系，先对这些代码混个眼熟，因为更重要的是了解当前描述的消息队列模型是如何工作的。

不难发现在这个最简单的消息队列模型中，生产者和消费者在与队列的通信过程中都需要首先创建与队列的连接并开辟信道后才能传输具体的消息。我想之后的模型大多也是如此吧。