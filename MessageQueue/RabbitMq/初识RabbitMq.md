# 初识RabbitMq

## 一、简介

先看看RabbitMq的百度词条，看看它是如何介绍的：

**RabbitMQ**是实现了高级消息队列协议（AMQP）的开源消息代理软件（亦称面向消息的中间件）。RabbitMQ服务器是用Erlang语言编写的，而集群和故障转移是构建在开放电信平台框架上的。所有主要的编程语言均有与代理接口通讯的客户端库。

在学习RabbitMq之前需要先了解两个概念——MQ和AMQP，关于这两个概念，我觉得https://blog.csdn.net/hellozpc/article/details/81436980讲述地还可以，这里不做赘述了。本篇主要讲解一下RabbitMq的工作流程和模型，这一切嘛，还得从AMQP开始讲起



## 二、AMQP

AMQP（Advanced Message Queuing Protocol，高级消息队列协议）是一个进程间传递**异步消息**的**网络协议**。当前各种应用大量使用异步消息模型，并随之产生众多消息中间件产品及协议，标准的不一致使应用与中间件之间的耦合限制产品的选择，并增加维护成本。AMQP是一个提供统一消息服务的应用层标准协议，基于此协议的客户端与消息中间件可传递消息，并不受客户端/中间件不同产品，不同开发语言等条件的限制。

来看看AMQP的模型：

![img](http://kylescloud.top/site/pic/AMQPmodel.jpg)

以下是对模型中的概念的简单阐述

### Publisher

消息生产者。生产者将生产的消息发送到Server端。

### Consumer

消息消费者。消费者只需要监听Message Queue，当队列中有消息的时候，就拿出来消费。因此在Exchange和Message Queue之间有绑定的关系存在，

### Server

消息服务

### VirtualHost

虚拟主机，比较上层的一个路由，类似于路由器的概念

### Exchange

交换机，生产者直接将消息投递到Exchange中。但是要经历3个过程 ->server->Virtual host->Exchange。

### MessageQueue

消息队列



另外还包括一些核心概念，部分与上述内容重复：

- Server: 又称Broker,接收客户端的链接，实现AMQP实体服务
- Connection: 连接，应用程序与Broker的网络连接
- Channel:网络信道，几乎所有的操作（数据的读、写）都在Channel中进行，Channel是进行消息读写的通道。客户端可建立多个Channel，每个Channel代表一个会话任务。
- Message：消息，服务器和应用程序之间传送的数据，由Properties和Body组成。Properties可以对消息进行修饰，比如消息的优先级、延迟等高级特性；Body则就是消息体内容。
- Virtual host:虚拟主机，用于进行逻辑隔离，是最上层的消息路由。一个Virtual host 里面可以有若干个Exchange和Queue，同一个Virtual Host里面不能有相同名称的Exchange和Queue。拿数据库（用MySQL）来类比：RabbitMq相当于MySQL，RabbitMq中的VirtualHost就相当于MySQL中的一个库。
- Exchange：交换机，接收消息，根据路由键转发消息到绑定的队列
- Binding:Exchange 和Queue之间的虚拟连接，Binding中可以包含routing key
- Routing key:一个路由规则，虚拟机可用它来确定如何路由一个特定消息。
- Queue:也称为message Queue,消息队列，保存消息并将它们转发给消费者。



## 三、Message

在讲解RabbitMq之前，需要知道的是——消息队列是基于消息(Message)来工作的，因此Message的数据构成会影响其在Rabbit工作的流向，来简单了解一下它的组成：

**Message**，服务器和应用程序之间传送的数据，本质上就是一段数据，由RoutingKey、Delivery mode、Headers、Properties和Payload（ Body ）组成。

### RoutingKey

路由键，用于绑定到指定的MessageQueue

### **Delivery mode**

表示是否持久化

### **Headers**

用于存储消息的头部信息。

### **Properties**

主要包含了以下信息：

- content_type ： 消息内容的类型
- content_encoding： 消息内容的编码格式
- priority： 消息的优先级
- correlation_id：关联id
- reply_to：用于指定回复的队列的名称
- expiration： 消息的失效时间
- message_id： 消息id
- timestamp：消息的时间戳
- type： 类型
- user_id： 用户id
- app_id： 应用程序id
- cluster_id： 集群id



## 四、RabbitMq

再来看下RabbitMq在这个协议下的工作模型:

![img](http://kylescloud.top/site/pic/RabbitMqModel.jpg)

1.消息生产者连接到RabbitMQ Broker，创建connection，开启channel。
2.生产者声明交换机类型、名称、是否持久化等。
3.发送消息，并指定消息是否持久化等属性和routing key。
4.exchange收到消息之后，根据routing key路由到跟当前交换机绑定的相匹配的队列里面。
5.消费者监听接收到消息之后开始业务处理，然后发送一个ack确认告知消息已经被消费。
6.RabbitMQ Broker收到ack之后将对应的消息从队列里面删除掉。



## 五、总结

第一次接触RabbitMq，写的其实不好，不过学习新技术需要一个过程，之后对mq更熟悉了会对文章做详尽的修改。