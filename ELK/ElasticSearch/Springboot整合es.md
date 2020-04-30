# Springboot整合es

## 一、依赖配置

在spring initializer中加入Spring web + lombok + Spring data Elasticsearch



## 二、yml配置

```
spring:
  application:
    name: log
  data:
    elasticsearch:
      client:
        reactive:
          endpoints: 119.3.126.156:9200
          username: elastic
          password: changeme
```



## 三、Dao

```
package com.example.es.dao;

import com.example.es.entity.InterviewLog;
import org.springframework.data.repository.CrudRepository;

public interface InterviewLogDao extends CrudRepository<InterviewLog,Long> {
}
```

```
package com.example.es.dao;

import org.springframework.data.repository.CrudRepository;

public interface OperatorLogDao extends CrudRepository<OperatorLogDao,Long> {
}
```



