# LogStash+Rabbit+ES

## 一、Logstash input配置rabbitmq

工作中使用rabbitmq作为输入源，在logstash的配置中主要配置了以下参数，因此，rabbit在发送消息时要将其一一对应：

```
input {
  rabbitmq{
    host => ***.**.**.**
    port => 5672
    user => "Kyle"
    psssword => "123456"
    queue => "log"
    key => "logKey"
    exchange => "logExchange"
    durable => true
    subscription_retry_interval_seconds => 5
    #start_position => "beginning" 
  }
}
```



## 二、rabbit发送mq

在发送消息时，需要指定exchange、routingKey、queue

以下是rabbitmq原生api的实现，spring集成的话应该会更简单一些,之后会补上：

```java
public class EmitMessage {
    private static final String EXCHANGE_NAME = "logExchange";

    public static void main(String[] args) {
        ConnectionFactory factory = new ConnectionFactory();
        factory.setHost("119.23.64.10");
        factory.setUsername("Kyle");
        factory.setPassword("123456");
        try {
            JsonObject jsonObject = new JsonObject();
            jsonObject.addProperty("name","kyle");
            jsonObject.addProperty("type","interview_log");
            Connection connection = factory.newConnection();
            Channel channel = connection.createChannel();
           //设置exchange类型为direct，且为持久化
          channel.exchangeDeclare(EXCHANGE_NAME,"direct",true);
           //发送消息，指定exchange，routingKey以及数据
            channel.basicPublish(EXCHANGE_NAME,"logKey",null,jsonObject.toString().getBytes());
            System.out.println(" [x] Sent white message");

        } catch (TimeoutException e) {
            e.printStackTrace();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

需要注意的是，由于工作需要logstash的输出源是elasticsearch，es支持的数据格式为json，因此rabbit传输的数据格式需要依赖es，所以这里传的是json数据。若logstash没有配置输出源的数据模板，则rabbit发送的第一个消息会被作为模板并创建与之相对的index。因此建议在发送rabbit之前还是先在logstash中定义好数据模板比较好。



## 三、查看es索引内容

若rabbit发送成功，且json数据校验成功，那么index就会被创建，这里我创建的index名为interview_log，当需要获取interview_log信息时，在es所在服务器通过curl获取

```
curl -x GET "http://localhost:9200/interview_log/_search"
```