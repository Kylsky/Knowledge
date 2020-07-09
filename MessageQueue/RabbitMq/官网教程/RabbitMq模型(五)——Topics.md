# RabbitMq模型(五)——Topics

## 一、模型

前面讲到，routing模型让日志系统的功能更加细化了，消费者可以通过指定routing_key来获取指定类型的日志。

那么现在出现了新的问题，我们假设日志生产者是许多的设备，这些设备类型都各不相同，但都会发出info、error这两个类型的消息。现在，消费者不仅仅只需要基于消息类型来处理消息，还要根据设备类型来处理消息。比如消费者1需要处理设备A发出的info类型的日志消息，消费者2需要处理设备B发出的所有类型的日志消息，这时routing模型显然无法满足当下的要求了。

为了实现这一目标，是时候看看下面这个模型了：

![img](https://www.rabbitmq.com/img/tutorials/python-five.png)

你可能会发现这个模型长得其实和routing模型差不多，不过还是有一些差别的

**首先**，exchange的类型被指定为topic，而routing的类型为direct

**其次**，topic模型下的routing_key不能被随意指定，必须是以句号分割的字符串，字符串的大小限制为255字节，为了做区分，这里我们将其称为binding key，一个制定了binding key的message会像routing模型一样被分发到与binding key匹配的队列，但是这里有两种特殊的情况：*和#

```
* 可以代替一个单词
# 可以代替零个或多个单词。
```

设想最初我们想要满足的消费日志的条件，便可以按照这个思路来声明binding key

```
#消费者1需要处理设备A发出的info类型的日志消息
binding key: A.info
#消费者2需要处理设备B发出的所有类型的日志消息
binding key: B.*
```

甚至还可以做许多拓展

```
#消费者3需要处理所有设备的info类型日志
binding key: #
或 *.# 
或 *.*
```



## 二、代码演示

**消息生产者**

```java
package testRabbitMq.Model5;

import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.ConnectionFactory;

import java.io.IOException;
import java.util.concurrent.TimeoutException;

public class EmitLog {
    private static final String EXCHANGE_NAME = "logs";

    public static void main(String[] args) {
        ConnectionFactory factory = new ConnectionFactory();
        factory.setHost("119.23.64.10");
        factory.setUsername("Kyle");
        factory.setPassword("123456");
            try {
                Connection connection = factory.newConnection();
                Channel channel = connection.createChannel();
                //声明exchange类型
                channel.exchangeDeclare(EXCHANGE_NAME,"topic");
                //发送消息
                //A的info日志
                channel.basicPublish(EXCHANGE_NAME,"A.info",null,"A.info log".getBytes());
                //B的error日志
                channel.basicPublish(EXCHANGE_NAME,"B.error",null,"B.error log".getBytes());
                //B的所有日志
                channel.basicPublish(EXCHANGE_NAME,"B.*",null,"B all log".getBytes());
                //所有日志
                channel.basicPublish(EXCHANGE_NAME,"#",null,"all device all log".getBytes());
            } catch (IOException e) {
                e.printStackTrace();
            } catch (TimeoutException e) {
                e.printStackTrace();
            }
    }
}
```

**消息消费者**

```java
package testRabbitMq.Model5;

import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.ConnectionFactory;
import com.rabbitmq.client.DeliverCallback;
import com.sun.corba.se.pept.transport.ConnectionCache;

import java.io.IOException;
import java.util.concurrent.TimeoutException;

public class RecvLog {
    private static final String EXCHANGE_NAME = "logs";

    public static void main(String[] args) {
        ConnectionFactory factory = new ConnectionFactory();
        factory.setHost("119.23.64.10");
        factory.setUsername("Kyle");
        factory.setPassword("123456");

        try {
            Connection connection = factory.newConnection();
            Channel channel = connection.createChannel();
            //声明exchange
            channel.exchangeDeclare(EXCHANGE_NAME,"topic");
            //声明队列
            String queueName = channel.queueDeclare().getQueue();
            //绑定queue-exchange-routing_key
            channel.queueBind(queueName,EXCHANGE_NAME,"A.info");
            channel.queueBind(queueName,EXCHANGE_NAME,"B.error");
            //声明回调
            DeliverCallback deliverCallback = (consumerTag, delivery) -> {
                String message = new String(delivery.getBody(), "UTF-8");
                System.out.println(" [x] Received '" +
                        delivery.getEnvelope().getRoutingKey() + "':'" + message + "'");
            };

            //消费消息
            channel.basicConsume(queueName, true, deliverCallback, consumerTag -> { });

        } catch (IOException e) {
            e.printStackTrace();
        } catch (TimeoutException e) {
            e.printStackTrace();
        }
    }
}
```

测试过程中，可以通过改变queueBind中的routing_key来观察接收到的日志消息。