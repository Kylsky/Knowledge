# Springboot整合es

## 一、依赖配置

在spring initializer中加入Spring web + lombok + Spring data Elasticsearch



## 二、yml配置

```
spring:
  application:
    name: log
  elasticsearch:
    rest:
      uris: 119.3.126.156:9200
      username: elastic
      password: changeme


server:
  port: 1234
```



## 三、Dao

```
package com.example.es.dao;

import com.example.es.entity.InterviewLog;
import org.springframework.data.repository.CrudRepository;

public interface InterviewLogDao extends CrudRepository<InterviewLog,Long> {
}
```



## 四、Service

### EsService

```java
package com.example.es.service;

import com.example.es.entity.InterviewLog;
import org.springframework.stereotype.Service;

public interface EsService {
    public Iterable<InterviewLog> findAll();
}

```

### EsServiceImpl

```java
package com.example.es.service.impl;

import com.example.es.dao.InterviewLogDao;
import com.example.es.entity.InterviewLog;
import com.example.es.service.EsService;
import org.springframework.stereotype.Service;

import javax.annotation.Resource;
import java.util.List;

@Service
public class EsServiceImpl implements EsService {
    @Resource
    private InterviewLogDao interviewLogDao;

    @Override
    public Iterable<InterviewLog> findAll(){

        return interviewLogDao.findAll();
    }
}
```



## 五、Controller

该Controller用于获取所有的日志信息

```java
package com.example.es.controller;

import com.example.es.entity.InterviewLog;
import com.example.es.service.EsService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;
import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.RequestMapping;

import javax.annotation.Resource;
import java.util.ArrayList;
import java.util.List;

@Controller
@RequestMapping("/interview")
public class InterviewController {
    @Resource
    private EsService esService;

    @RequestMapping("/findAll")
    public List<InterviewLog> findAll(){
       Iterable<InterviewLog> interviewLogs = esService.findAll();
       List<InterviewLog> logs = new ArrayList<>();
       interviewLogs.forEach(interviewLog -> {
           logs.add(interviewLog);
       });
       return logs;
    }
}
```



## 六、实体类

```java
package com.example.es.entity;

import org.springframework.data.annotation.Id;
import org.springframework.data.elasticsearch.annotations.Document;
import org.springframework.data.elasticsearch.annotations.Field;
import org.springframework.data.elasticsearch.annotations.FieldType;

@Document(indexName = "interview_log",type = "_doc")
public class InterviewLog {
    @Id
    private Long id;
    @Field(type = FieldType.Text,name = "type")
    private String type;

    @Field(type = FieldType.Text)
    private String kyle;

    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }

    public String getType() {
        return type;
    }

    public void setType(String type) {
        this.type = type;
    }

    public String getKyle() {
        return kyle;
    }

    public void setKyle(String kyle) {
        this.kyle = kyle;
    }
}
```

