# logback整合rabbit

## 一、前言

最近在做日志采集的通用解决方案，需要解决一个比较小的需求：

在每次代码使用logger.error("msg")的时候将该日志信息发送到rabbit，并通过logstash传输到es做存储，传输的字段主要为以下几个：

```
level：日志等级(logback自带)
message：日志消息(logback自带)
time：日志时间(logback自带)
class-line：发送日志的代码位置(logback自带)
ip：发送日志的服务器ip(需要额外配置)
applicationName：服务名(需要额外配置)
```



## 二、logback

在网上找到了一些解决方案，大致思路如下：

**1**.导入logback、spring-amqp依赖

**2**.编写logback-spring.xml，并在application.yml中配置该文件路径

(由于需要额外配置ip，所以还需要额外多1步)

**3**.编写类继承ClassConverter，返回当前服务器ip



接下来一步步看：

### **1.导入依赖**

```
<dependency>
    <groupId>org.springframework.amqp</groupId>
    <artifactId>spring-rabbit</artifactId>
    <version>2.2.6.RELEASE</version>
</dependency>
```



### **2.编写logback-rabbit.xml，配置application.yml**

#### **logbak-rabbit.xml**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <!-- 引入ip converter以获取服务器ip -->
    <conversionRule conversionWord="ip" converterClass="cn.pubinfo.log_helper.config.IPLogConfig" />
    <!-- 引入spring中rabbit相关的配置 -->
    <springProperty scope="context" name="log_dir" source="log.path"/>
    <springProperty name="rabbitmqHost" source="spring.rabbitmq.host"/>
    <springProperty name="rabbitmqPort" source="spring.rabbitmq.port"/>
    <springProperty name="rabbitmqUsername" source="spring.rabbitmq.username"/>
    <springProperty name="rabbitmqPassword" source="spring.rabbitmq.password"/>
    <springProperty name="applicationName" source="spring.application.name"/>
    <!-- Console log -->
    <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>%red(%d{yyyy-MM-dd HH:mm:ss}) %green([%thread]) [%class:%line] %highlight(%-5level) - %cyan(%msg%n)
            </pattern>
            <charset>UTF-8</charset>
        </encoder>
    </appender>

    <appender name="AMQP" class="org.springframework.amqp.rabbit.logback.AmqpAppender">
        <filter class="ch.qos.logback.classic.filter.ThresholdFilter">
            //过滤warn以下等级的日志不会发送到rabbit
            <level>WARN</level>
        </filter>
        <layout>
            <!-- 以json格式配置消息体发送到rabbitmq -->
            <pattern>
                <![CDATA[
                {
                    "time":
                        "%d{yyyy-MM-dd HH:mm:ss}",
                    "class-line":
                        "%class.%method:%line",
                    "level":
                        "%-5level",
                    "message":
                        "%msg",
                    "applicationName":
                        "${applicationName}",
                    "ip":
                        "%ip"
                }]]>
            </pattern>
        </layout>
        <!-- 配置rabbit -->
        <addresses>${rabbitmqHost}:${rabbitmqPort}</addresses>
        <username>${rabbitmqUsername}</username>
        <password>${rabbitmqPassword}</password>

        <!-- 自动生成exchange -->
        <declareExchange>true</declareExchange>
        <!-- 指定应用id -->
        <applicationId>logback</applicationId>
        <!-- 设置交换器类型 -->
        <exchangeType>direct</exchangeType>
        <!-- 设置交换器名称 -->
        <exchangeName>logback</exchangeName>
        <!-- 设置路由键 -->
        <routingKeyPattern>logback</routingKeyPattern>
        <generateId>true</generateId>
        <charset>UTF-8</charset>
        <!-- 设置为持久化 -->
        <durable>true</durable>
        <deliveryMode>NON_PERSISTENT</deliveryMode>
    </appender>

    <!-- The file record only records the log of the specified package -->
    //这里制定了controller，若想整个项目都发送日志，则指定对应包名即可，如com.example
    <logger name="com.example.es.controller" level="warn" additivity="false">
        <appender-ref ref="STDOUT"/>
        <appender-ref ref="AMQP"/>
    </logger>

    <root level="INFO">
        <appender-ref ref="STDOUT"/>
    </root>
</configuration>
```

#### **application.yml**

```
追加以下内容：
logging:
  config: classpath:logback-rabbit.xml
```



### 3.编写类继承ClassConverter，返回当前服务器ip

```java
package cn.pubinfo.logback.config;

import ch.qos.logback.classic.pattern.ClassicConverter;
import ch.qos.logback.classic.spi.ILoggingEvent;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.net.*;
import java.util.ArrayList;
import java.util.Enumeration;
import java.util.List;
import java.util.Optional;

public class IPLogConfig extends ClassicConverter {
    private static final Logger logger = LoggerFactory.getLogger(IPLogConfig.class);
    private volatile static Inet4Address inetAddr = null;

    @Override
    public String convert(ILoggingEvent iLoggingEvent) {
        try {
            if (inetAddr==null){
                inetAddr = getLocalIp4Address().get();
            }
            return inetAddr.getHostAddress();
        } catch (SocketException e) {
            e.printStackTrace();
        }
        return null;
    }

    public static List<Inet4Address> getLocalIp4AddressFromNetworkInterface() throws SocketException {
        List<Inet4Address> addresses = new ArrayList<>(1);
        Enumeration e = NetworkInterface.getNetworkInterfaces();
        if (e == null) {
            return addresses;
        }
        while (e.hasMoreElements()) {
            NetworkInterface n = (NetworkInterface) e.nextElement();
            if (!isValidInterface(n)) {
                continue;
            }
            Enumeration ee = n.getInetAddresses();
            while (ee.hasMoreElements()) {
                InetAddress i = (InetAddress) ee.nextElement();
                if (isValidAddress(i)) {
                    addresses.add((Inet4Address) i);
                }
            }
        }
        return addresses;
    }

    /**
     * 过滤回环网卡、点对点网卡、非活动网卡、虚拟网卡并要求网卡名字是eth或ens开头
     *
     * @param ni 网卡
     * @return 如果满足要求则true，否则false
     */
    private static boolean isValidInterface(NetworkInterface ni) throws SocketException {
        return !ni.isLoopback() && !ni.isPointToPoint() && ni.isUp() && !ni.isVirtual()
                && (ni.getName().startsWith("eth") || ni.getName().startsWith("ens") ||ni.getName().startsWith("wlan"));
    }

    /**
     * 判断是否是IPv4，并且内网地址并过滤回环地址.
     */
    private static boolean isValidAddress(InetAddress address) {
        return address instanceof Inet4Address && address.isSiteLocalAddress() && !address.isLoopbackAddress();
    }


    private static Optional<Inet4Address> getIpBySocket() throws SocketException {
        try (final DatagramSocket socket = new DatagramSocket()) {
            socket.connect(InetAddress.getByName("www.baidu.com"), 10002);
            if (socket.getLocalAddress() instanceof Inet4Address) {
                return Optional.of((Inet4Address) socket.getLocalAddress());
            }
        } catch (UnknownHostException e) {
            throw new RuntimeException(e);
        }
        return Optional.empty();
    }

    public static Optional<Inet4Address> getLocalIp4Address() throws SocketException {
        final List<Inet4Address> ipByNi = getLocalIp4AddressFromNetworkInterface();
        if (ipByNi.isEmpty() || ipByNi.size() > 1) {
            final Optional<Inet4Address> ipBySocketOpt = getIpBySocket();
            if (ipBySocketOpt.isPresent()) {
                return ipBySocketOpt;
            } else {
                return ipByNi.isEmpty() ? Optional.empty() : Optional.of(ipByNi.get(0));
            }
        }
        return Optional.of(ipByNi.get(0));
    }

    public static void main(String[] args) throws UnknownHostException, SocketException {
        System.out.println(getIpBySocket().get().getHostAddress());
    }
}

```



## 三、配置logstash

```
input{
	rabbitmq{
        host => "*.*.*.*"
        port => 5672
        user => ""
        password => ""
        queue => "logback"
        key => "logback"
        exchange => "logback"
        durable => true
        subscription_retry_interval_seconds => 5
        type => logback
  	}
}

output{
	if[type] == "logback"{
    	elasticsearch {
      	hosts => ["127.0.0.1:9200"]
      	user => "elastic"
      	password => "changeme"
      	index => "logback"
    	}
  	}
}
```



## 四、大功告成

这样就差不多结束了，如果想要做es的自定义mapping，则需要额外操作es，不过对于一些简单的需求来说，es的自动生成映射也足够应付了。以下给出我的es mapping：

```
PUT /logback
{
  "mappings": {
    "properties": {
      "time":{
        "format": "yyyy-MM-dd HH:mm:ss",
        "type":"date"
      },
      "applicationName":{"type":"text"},
      "class-line":{"type":"text"},
      "ip":{"type":"text"},
      "level":{"type":"keyword"},
      "message":{"type":"text"}
    }
  }
}
```



## 五、其他

如果项目不使用配置中心如nacos，那么以上的操作就完全满足需求了，但是若使用nacos，则yml中关于logging和rabbit的配置则需要全部转移至nacos，否则会产生undefined错误，目前原因还没有找到。