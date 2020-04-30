# LogStash安装

## 一、下载安装包

在官网下载最新安装包https://www.elastic.co/cn/downloads/logstash

```
cd /opt
wget https://artifacts.elastic.co/downloads/logstash/logstash-7.6.2.tar.gz
```

## 二、解压安装包

```
tar -xzvf logstash-7.6.2.tar.gz
```



## 三、修改配置文件连接es

```
cd /opt/logstash/config
cp logstash-sample.conf logstash.conf
vi logstash.conf
```

修改内容如下：

```
input {
  rabbitmq{
    host => 119.23.64.10
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

output {
  if[type] == "operator_log"{
    elasticsearch {
      hosts => ["http://localhost:9200"]
      user => "elasticsearch"
      password => "123456"
      index => "operator_log"
      #这里让logstash生成默认的模板，也可以设置自己的
      #template => "/opt/logstash/config/operator_log.json"
      #template_name => "/opt/logstash/conf/operator_log.json"
      #template_overwrite => true
    }   
  }
  if[type] == "interview_log"{
    elasticsearch {
      hosts => ["http://localhost:9200"]
      user => "elasticsearch"
      password => "123456"
      index => "interview_log"
      #template => "/opt/logstash/config/interview_log.json"
      #template_name => "interview_log.json"
      #template_overwrite => true
    }   
  }
}
```

修改后，检测配置文件是否正确

```
./logstash --path.settings /opt/logstash/config/ -f /opt/logstash/config/logstash.conf --config.test_and_exit
```

若配置正确，则会显示Configuration OK



## 四、启动logstash

```
./logstash --path.settings /opt/logstash/config/ -f /opt/logstash/config/logstash.conf
```



## 五、查看启动状态

```
 curl -X GET "localhost:9600"
```

若显示数据，说明启动成功