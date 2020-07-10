# RabbitMq模型(二)——WorkQueues

## 一、消息队列模型

消费者以竞争模式消费rabbitmq中的任务。在第一个教程中，我们编写了从指定队列发送和接收消息的程序。在本例中，将创建一个工作队列，用于在多个工作者之间分配耗时的任务。

![img](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/RabbitMqWorkQueues.png)

工作队列(即任务队列)背后的主要思想是避免立即执行资源密集型任务，并且必须等待它完成的场景出现。因此，我们希望把任务安排在以后完成。我们将任务封装为消息并将其发送到队列。在后台运行的工作进程将弹出任务并最终执行作业。当运行许多worker时，任务将在它们之间共享。

这个概念在web应用程序中特别有用，因为在短的HTTP请求窗口中不可能处理复杂的任务。



## 二、代码演示

**消息生产者**

```java
package testRabbitMq.Model2;

import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.ConnectionFactory;
import com.rabbitmq.client.MessageProperties;

public class NewTask {

    private static final String TASK_QUEUE_NAME = "task_queue";

    public static void main(String[] argv) throws Exception {
        ConnectionFactory factory = new ConnectionFactory();
        factory.setHost("119.23.64.10");
        factory.setUsername("Kyle");
        factory.setUsername("123456");
        try (Connection connection = factory.newConnection();
             Channel channel = connection.createChannel()) {
            channel.queueDeclare(TASK_QUEUE_NAME, true, false, false, null);

            String message = new Date().toString();

            channel.basicPublish("", TASK_QUEUE_NAME,
                    MessageProperties.PERSISTENT_TEXT_PLAIN,
                    message.getBytes("UTF-8"));
            System.out.println(" [x] Sent '" + message + "'");
        }
    }
}
```



**消息消费者**

旧Recv.java程序也需要一些更改:它需要为消息体中的每个点伪造一秒的工作时间。它将处理已交付的消息并执行任务

```java
package testRabbitMq.Model2;

import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.ConnectionFactory;
import com.rabbitmq.client.DeliverCallback;

public class Worker {

    private static final String TASK_QUEUE_NAME = "task_queue";

    public static void main(String[] argv) throws Exception {
        ConnectionFactory factory = new ConnectionFactory();
        factory.setHost("119.23.64.10");
        factory.setUsername("Kyle");
        factory.setPassword("123456");
        final Connection connection = factory.newConnection();
        final Channel channel = connection.createChannel();

        channel.queueDeclare(TASK_QUEUE_NAME, true, false, false, null);
        System.out.println(" [*] Waiting for messages. To exit press CTRL+C");

        channel.basicQos(1);

        DeliverCallback deliverCallback = (consumerTag, delivery) -> {
            String message = new String(delivery.getBody(), "UTF-8");

            System.out.println(" [x] Received '" + message + "'");
            try {
                doWork(message);
            } finally {
                System.out.println(" [x] Done");
                channel.basicAck(delivery.getEnvelope().getDeliveryTag(), false);
            }
        };
        channel.basicConsume(TASK_QUEUE_NAME, false, deliverCallback, consumerTag -> { });
    }

    private static void doWork(String task) {
        for (char ch : task.toCharArray()) {
            if (ch == '.') {
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException _ignored) {
                    Thread.currentThread().interrupt();
                }
            }
        }
    }
}
```



## 三、测试运行

在生产者生产消息之前，先将Worker启动，这里为了模拟竞争消息的场景，因此启动两个Worker，idea中如果要并行执行相同的类，需要在启动配置上选中parallel选项，具体操作可以看https://blog.csdn.net/sinat_41905822/article/details/97813057。之后多次启动NewTask类，可以发现每一个Worker都将有机会对队列中的消息进行消费。

默认情况下，RabbitMQ将按顺序将每个消息发送给下一个使用者。平均每个消费者将得到相同数量的消息。这种分发消息的方式称为round-robin(循环)



## 四、消息确认机制

来翻译一下官网对于rabbitmq的消息确认机制的描述：

与普通消息不同，若Worker完成一项任务可能需要几秒钟的时间，你可能想知道，如果其中一个消费者启动了一个很长的任务，但只完成了一部分就挂掉了，会发生什么情况。对于我们当前的代码，RabbitMQ一旦将消息传递给消费者，就会立即将其标记为删除。在这种情况下，如果杀死了一个Worker线程，我们将丢失它正在处理的消息。我们还将丢失所有发送给这个特定worker但尚未处理的消息。

但我们不想失去任何任务。如果一个Worker挂了，我们希望把这个任务交给另一个Worker。为了确保消息永远不会丢失，RabbitMQ支持消息确认。消费者发送一个ack(nowledgement)来告诉RabbitMQ，某个特定的消息已经被接收、处理，RabbitMQ可以自由地删除它。

如果消费者在一定时间内不发送ack的情况下死亡(其通道关闭、连接关闭或TCP连接丢失)，RabbitMQ将理解消息未被完全处理，并将重新对其排队。如果同时有其他的消费者在线，它会很快将其重新发送给另一个消费者。这样即使Worker偶尔挂了，你也可以确保没有信息丢失。

下面这句话在我看来很关键：

**RabbitMq没有任何消息超时机制。当使用者死亡时，RabbitMQ将重新传递消息。即使处理一条消息需要很长很长的时间，也没关系。**

我本来以为rabbitmq重新传递消息的机制在于为消息设置处理超时时间，因此当Worker与mq之间的连接断开后达到超时的标准，然后mq才重新传递。看完这句话后，其实发现不然，我想rabbitmq使用类似心跳机制来直接监控了自己与Worker之间的连接，而不是间接通过判断超时时间来触发消息重发机制。

接下来继续回到消息确认机制——手动消息确认在默认情况下是打开的。在前面的示例中，我们通过autoAck=true标志显式地关闭了它们。一旦我们完成了一个任务，就应该将这个标志设置为false并从worker发送一个适当的确认。因此，倘若在消费者的代码中将autoAck置为了true，而在消息确认时需要手动实现重传信息，则需要注意做如下操作。另外，ack信息需要在接收消息的Channel上发送，否则会产生异常。

```
autoAck = false
```

忘记发送ack信息是一个常见的错误。这是一个简单的错误，但后果很严重。当客户端退出时，消息将被重新发送(可能看起来像随机的重新发送)，但是RabbitMQ将消耗越来越多的内存，因为尽管消息已经被Worker处理了，但是在mq中仍是未处理状态，这可能会导致mq一直无法将其释放。

可以通过以下命令查看未被确认的消息

```
sudo rabbitmqctl list_queues name messages_ready messages_unacknowledged
```



## 五、消息持久化

我们已经学会了如何确保即使Worker死亡，任务也不会丢失。但是，如果RabbitMQ服务器停止运行，我们的任务仍然会丢失。当RabbitMQ退出或崩溃时，它将忘记队列和消息，除非我们告诉它不要这样做。

确保消息不丢失需要做两件事:**将队列和消息持久化**。

操作也很简单,可以参考前面的这两句代码：

```java
boolean durable = true;
channel.queueDeclare("hello", durable, false, false, null);
```

虽然这个命令本身是正确的，但是它在我们目前的设置中不能工作。那是因为我们在讲第一个模型的时候已经定义了一个名为hello的队列，它不是持久的。RabbitMQ**不允许我们使用不同的参数重新定义现有队列**，并将向任何试图这样做的程序返回一个错误。但是有一个快速的解决方法——让我们声明一个新的队列，例如task_queue：

```java
boolean durable = true;
channel.queueDeclare("task_queue", durable, false, false, null);
```

此时task_queue就被声明为持久化的了，我们确信即使RabbitMQ重新启动，task_queue队列也不会丢失。现在，我们需要将**消息声明为持久化**——通过将MessageProperties(它实现了BasicProperties)设置为PERSISTENT_TEXT_PLAIN值，就像这样：

```java
channel.basicPublish("", "task_queue",
            MessageProperties.PERSISTENT_TEXT_PLAIN,
            message.getBytes());
```

将消息标记为持久并不能完全保证消息不会丢失。虽然它告诉RabbitMQ将消息保存到磁盘，但是当RabbitMQ接受了一条消息并且还没有保存它时，仍然会有一个较短的窗口期。此外，RabbitMQ不会对每条消息都执行fsync(2)——它可能只是将消息保存到缓存中，而不是真正刷新到磁盘上。持久性保证并不强，但对于我们的简单任务队列来说已经足够了。如果需要更强的保证，那么则可以使用[publisher confirms](https://www.rabbitmq.com/confirms.html)。



# 六、任务公平分发

你可能会发现，循环调度仍然不能完全按照我们希望的方式工作。例如，在两个Worker的情况下，当所有重量级的消息很多，轻量级的消息很少，一个Worker会一直很忙，而另一个Worker几乎什么都不做，但是RabbitMQ对此一无所知，仍然会均匀地分发消息，这其实也是任务分配不均的问题。

这是因为RabbitMQ只在消息进入队列时发送消息。它不会查看Worker仍在进行的未被确认的任务。它只是盲目地将第n条消息发送给第n个消费者，尽管这可能会给一些Worker添堵。

为了解决这个问题，我们可以使用basicQos方法，并设置prefetchCount = 1。这告诉RabbitMQ一次不要向一个worker发送多个消息。或者，换句话说，在一个Worker处理并确认前一个消息之前，不向其发送新消息。相反，它将把消息发送给下一个不太忙的Worker。

具体操作如下：

```java
int prefetchCount = 1;
channel.basicQos(prefetchCount);
```

