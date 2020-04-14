# RabbitMq模型(四)——Routing

## 一、模型

在讨论第四个模型前，先来回顾一下第二个和第三个模型

**WorkQueues：**

![img](http://kylescloud.top/site/pic/RabbitMqWorkQueues.png)

**Pub/Sub:**

![img](http://kylescloud.top/site/pic/RabbitMqPubSub.png)

在第二个模型中，生产者生产消息并发向指定队列，消费者再从指定队列中消费消息。到了第三个模型，生产者和消费者不再与队列直接关联，而是与exchange进行通信，间接向队列生产消息或从队列消费消息，这相对于第二个模型来说减少了与队列之间的耦合。

但是Pub/Sub模型只适合消息发布订阅模式，每一个消费者都会接收到消息广播，但是倘若每一个消费者都想像WorkQueues模型一样抢占式消费特定的(而非公用的)消息，又能够通过exchange间接与消息队列通信，那么就有了第四种模型——

**Routing：**

![img](http://kylescloud.top/site/pic/RabbitMqRouting.png)



## 二、绑定routing_key

Routing模型在exchange与queue的关联关系上其实没有本质上的差异，回顾Pub/Sub模型的绑定代码：

```java
channel.queueBind(queueName, EXCHANGE_NAME, "");
```

可以发现queueBind方法中的第三个参数是空的，为了实现上面提到的需求，可以使用额外的routingKey作为第三个参数。如：

```java
channel.queueBind(queueName, EXCHANGE_NAME, "black");
```



## 三、Direct exchange

当然，光是这样做还不够，Pub/Sub模型中我们提到过，exchange中的fanout类型用于广播消息，在Routing模型中肯定是不适用的，因此，我们将用Direct exchange代替。Direct exchange背后的路由算法很简单——消息传递到其routing_key完全匹配的队列。就像这张图所描述的一样：

![img](http://kylescloud.top/site/pic/RabbitMqRouting.png)

在这个设置中，我们可以看到Direct exchange X，它绑定了两个队列。第一个队列用routing_key **orange**绑定，第二个队列有两个routing_key，一个绑定**black**，另一个绑定**green**。

相关代码应做如下修改：

```java
channel.exchangeDeclare(EXCHANGE_NAME, "direct");
```



## 四、多重绑定

![img](http://kylescloud.top/site/pic/RabbitMqMultipleBinding.png)

使用相同的routing_key绑定多个队列是完全合法的。在我们的示例中，可以使用binding键black在X和Q1之间添加一个绑定。在这种情况下，Direct将像fanout一样工作，并将消息广播到所有匹配的队列。带有路由键black的消息将被发送到Q1和Q2。



## 五、代码演示

**消息生产者**

```java
package testRabbitMq.Model4;

import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.ConnectionFactory;

import java.io.IOException;
import java.util.concurrent.TimeoutException;

public class EmitMessage {
    private static final String EXCHANGE_NAME = "hello";

    public static void main(String[] args) {
        ConnectionFactory factory = new ConnectionFactory();
        factory.setHost("119.23.64.10");
        factory.setUsername("Kyle");
        factory.setPassword("123456");
        try {
            Connection connection = factory.newConnection();
            Channel channel = connection.createChannel();
            //设置exchange类型
            channel.exchangeDeclare(EXCHANGE_NAME,"direct");
           //发送消息，指定exchange_name,routing_key,消息体
          channel.basicPublish(EXCHANGE_NAME,"black",null,"black message".getBytes());
        } catch (TimeoutException e) {
            e.printStackTrace();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}

```

**消息消费者**

```java
package testRabbitMq.Model4;

import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.ConnectionFactory;
import com.rabbitmq.client.DeliverCallback;

import java.io.IOException;
import java.util.concurrent.TimeoutException;

public class RecvMessage {
    private static final String EXCHANGE_NAME = "hello";

    public static void main(String[] args) {
        ConnectionFactory factory = new ConnectionFactory();
        factory.setHost("119.23.64.10");
        factory.setUsername("Kyle");
        factory.setPassword("123456");
        try{
            Connection connection = factory.newConnection();
            Channel channel = connection.createChannel();
            //设置exchange类型
            channel.exchangeDeclare(EXCHANGE_NAME,"direct");
            //定义队列名
            String queueName = channel.queueDeclare().getQueue();
            //绑定队列
            channel.queueBind(queueName,EXCHANGE_NAME,"black");
            //设置回调
            DeliverCallback deliverCallback = (consumerTag, delivery) -> {
                String message = new String(delivery.getBody(), "UTF-8");
                System.out.println(" [x] Received '" +
                        delivery.getEnvelope().getRoutingKey() + "':'" + message + "'");
            };
            //消费消息
            channel.basicConsume(queueName, true, deliverCallback, consumerTag -> { });
        } catch (TimeoutException e) {
            e.printStackTrace();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

