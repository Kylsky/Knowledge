# RabbitMq模型(三)——发布订阅

## 一、消息队列模型

消息生产者同时向许多消费者发送消息

![img](http://kylescloud.top/site/pic/RabbitMqPubSub.png)

在前面的教程中，创建了一个工作队列。工作队列背后的假设是，每个任务只交付给一个工作者。在这一部分中，我们将做一些完全不同的事情——我们将向多个消费者传递消息。这种模式称为“发布/订阅”。

为了演示该模式，我们将构建一个简单的日志系统。它将由两个程序组成——第一个程序将发出日志消息，第二个程序将接收并打印这些消息。在我们的日志系统中，接收程序的每个运行副本都将获得消息。这样，我们就可以运行一个receiver，并将日志定向写到磁盘。同时，我们将能够运行另一个接收器并在屏幕上看到日志。从本质上讲，就是发布的日志消息将广播给所有的接收者。



## 二、Exchanges

在本教程前面的部分中，我们向队列发送和接收消息。现在是时候在Rabbit中引入完整的消息传递模型了。让我们快速回顾一下我们在之前的教程中所涵盖的内容:

1.生产者是发送消息的用户应用程序。

2.队列是存储消息的缓冲区。

3.使用者是接收消息的用户应用程序。

**RabbitMQ消息传递模型的核心思想**是，生产者从不直接向队列发送任何消息。实际上，通常情况下，生产者甚至根本不知道消息是否会被传递到任何队列。在rabbitmq中，生产者只能向交换器发送消息。交换是一件非常简单的事情。一边接收来自生产者的消息，另一边将消息推送到队列。交换器必须确切地知道如何处理它接收到的消息。它应该被附加到一个特定的队列吗?它应该被添加到许多队列中吗?或者它应该被丢弃吗？这些规则由exchange类型定义。

rabbitmq有几种可用的交换类型:direct、topic、headers和fanout。打开rabbitmq浏览器端的dashboard全部8个exchanges以及对应的类型，也可以通过命令sudo rabbitmqctl list_exchanges查看，如下：

```
AMQP default			direct
amq.direct				direct
amq.fanout				fanout
amq.headers				headers
amq.match				headers
amq.rabbitmq.log		topic
amq.rabbitmq.trace		topic
amq.topic				topic
```

另外可以在前几篇的代码里可以发现，我们指定的exchange为""，其实这样的编码对应了上述默认的exchange——AMQP default。不过这里主要主要还是跟着官网来看看fanout这个类型，并作一下代码演示，以下代码声明一个exchange，名称为logs，类型为fanout

```java
channel.exchangeDeclare("logs", "fanout");
```



## 三、临时队列

你可能还记得以前我们使用具有特定名称的队列(hello和task_queue)能够命名一个队列对我们来说是至关重要的——我们需要将Worker指向相同的队列。当希望在生产者和消费者之间绑定同一个队列时，给队列起一个名字是很重要的。

但这样的非临时队列不满足本篇将构建的日志系统，。我们想要了解所有的日志消息，而不仅仅是其中的一个子集。我们也只对当前新的消息感兴趣，而不是老消息。

要解决这个问题，我们需要两样东西。首先，每当我们连接到Rabbitmq时，我们需要一个新的空队列。为此，我们可以创建一个具有随机名称的队列，或者，更好的方法是让服务器为我们选择一个随机队列名称。其次，一旦我们断开消费者，队列应该被自动删除。

在Java客户端，当我们不向queueDeclare()提供参数时，我们创建一个非持久的、独占的、自动删除的队列，并生成一个名称:

```java
String queueName = channel.queueDeclare().getQueue();
```



