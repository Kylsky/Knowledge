# Nacos配置中心

## 我的设想

一个Spring项目在没有配置中心的时候，通常需要写4个配置文件，application.yml,application-dev.yml,application-prod.yml和application-test.yml。

一般来说application.yml定义了项目的一些基本信息，如**srping.application.name**,**server.port**,**logging.level**等基本信息，而数据库、mq等配置信息则可能需要根据不同的环境来进行配置，因此基本会放在其他三个配置中，并在application.yml中通过spring.profiles.active来指定项目环境。当使用nacos来进行项目的配置时，由于spring启动时需要从远程获取配置信息，所以从前application.yml相关的配置都需要在bootstrap.yml中，以此类推，dev、prod、test也是如此。差别在于，现在不需要将配置写在这些yml中了，只需要在项目的bootstrap*.yml中配置需要从远端获取Configuration的元数据即可由于没有具体实现过，所以还是来看看到底应该怎么做



## 添加nacos server配置

不管怎么样，将如dev、prod这些环境配置在nacos  server上都是必须的，nacos支持非常多样化的配置。先来回顾一下nacos的namespace、Data ID、Group：

**Namespace**

```
命名空间，用于隔离用户配置。不同的命名空间可以有相同的Group和DataID。Namespace的一个常见场景是在不同的环境中区分和隔离配置，比如在开发和测试环境以及生产环境中。
```

**Data ID**

```
Data ID组成如下：
${prefix}-${spring.profile.active}.${file-extension}
prefix的默认值就是spring.application.name，也可以通过spring.cloud.nacos.config.prefix配置

spring.profile.active即项目所在profile，如test、dev、prod，若为空，则Data ID会省略掉这一部分内容，可以通过spring.profiles-active配置

file-extension是配置内容的数据格式，目前支持properties和yaml，可以通过spring.cloud.nacos.config.file-extension配置
```

**Group**

```
群组，简单来说区分具有相同Data ID的配置集。
```



不难发现，要在nacos server上配置dev、prod配置有很多方法，我来简单列举一些：

1. 设置多个namespace，每一个namespace存放不同环境的配置文件，项目在配置时只需指定namespace即可
2. 设置一个namespace，统一使用DEFAULT_GROUP，通过Data ID中的${spring.profile.active}区分不同环境的配置文件，项目在配置时需要指定namespace和spring.active.profile
3. 设置一个namespace，使用不同Group，项目在配置时需要指定namespace和Group
4. 其实还有更多方案，比如使用dev、prod、test划分namespace，使用Group来划分不同的服务，不是也可以吗？……

ps：如果不指定namespace，则会使用默认的namespace，不过建议还是自己创建~



上面的方案对于当前项目来说都可用，但是从微服务配置的角度来说，到底哪个方案适合当前来用，才是最重要的。对于微服务来说，首先，一个完整的项目会涉及到多个服务，如日志服务、用户服务、订单服务。那么对于这些服务，我想我会考虑将其配置在不同的namespace中，即每一个服务对应一个namespace。

随后，将目标指向单个服务，在微服务中，每一个服务都会拥有多台实例，每一台实例在相同的环境中会使用相同的配置，回顾上面三个概念的介绍，可以发现namespace>Group>Data ID，因此，对于不同实例使用的不同环境配置，使用Group来加以区分，如Test_Group、Dev_Group、Prod_Group，随后通过Data ID来标识对应的环境配置。

当然，这里还有一个问题我是存疑的（是有关服务注册方面，所以对这里的操作影响不大，跳过不看也可）如果以服务来进行namespace划分，那么之后注册在namespace1的服务能否调用到注册在namespace2的服务呢？

因此，基于严谨认真的态度，决定使用第三种方案来做配置！

### 新建namespace

ui界面的操作就不展示了，比较简单，在左侧命名空间栏点进去找到添加按钮，命名空间自动生成就好了，空间名可以以服务名来标识，会比较清晰。

### 新建配置

这里用不同数据库作为两个环境的区分，前者为unit，后者为homstay

**dev环境**

```
Data ID:example.dev.yaml
Group:DEV_GROUP

配置内容：
spring:
  datasource:
    driver-class-name: com.mysql.jdbc.Driver
    username: root
    password: 123456
    url: jdbc:mysql://localhost:3306/unit?useUnicode=true&characterEncoding=utf-8&serverTimezone=GMT&useSSL=false
```

**prod环境**

```
Data ID:example.dev.yaml
Group:DEV_GROUP

配置内容：
spring:
  datasource:
    driver-class-name: com.mysql.jdbc.Driver
    username: root
    password: 123456
    url: jdbc:mysql://localhost:3306/homestay?useUnicode=true&characterEncoding=utf-8&serverTimezone=GMT&useSSL=false
```



## 项目中配置

nacos server配置完毕后，项目中需要获取对应的远程配置，则需要在bootstrap*.yml中进行相应的配置

### bootstrap.yml

这里其实和application.yml一样，配置一些基本的属性，如本地端口，注册中心地址，配置中心地址，应用名以及使用环境

```yaml
server:
  port: 8070

spring:
  cloud:
    nacos:
      config.:
        server-addr: 119.23.64.10:8848
      discovery:
        server-addr: 119.23.64.10:8848
  application:
    name: example
  profiles:
    active: dev
```

### bootstrap-dev.yml

dev环境需要获取到nacos server的远程dev配置

```yaml
spring:
  cloud:
    nacos:
      config:
        group: DEV_GROUP
        namespace: 7adf435f-aafe-4512-90fe-eb8b51b7cce2
        file-extension: yaml
      discovery:
        namespace: 7adf435f-aafe-4512-90fe-eb8b51b7cce2
  profiles: dev
```

### bootstrap-prod.yml

prod环境需要获取到nacos server的远程prod配置

```yaml
spring:
  cloud:
    nacos:
      config:
        group: PROD_GROUP
        namespace: 7adf435f-aafe-4512-90fe-eb8b51b7cce2
        file-extension: yaml
      discovery:
        namespace: 7adf435f-aafe-4512-90fe-eb8b51b7cce2
  profiles: prod
```



## 启动项目

修改Controller

```java
@RestController
@RequestMapping("/config")
@RefreshScope
public class ConfigController {
    @Value("${spring.datasource.url}")
    private String dataBaseUrl;


    @RequestMapping("/str")
    public String str(){return dataBaseUrl;}
}
```

切换环境为dev，启动项目，访问localhost:8070/config/get

```
jdbc:mysql://localhost:3306/unit?useUnicode=true&characterEncoding=utf-8&serverTimezone=GMT&useSSL=false
```

切换为prod，启动项目，访问localhost:8070/config/get

```
jdbc:mysql://localhost:3306/homestay?useUnicode=true&characterEncoding=utf-8&serverTimezone=GMT&useSSL=false
```

发现获取到的数据库配置不一样了，到此，本次配置测试就完成啦~