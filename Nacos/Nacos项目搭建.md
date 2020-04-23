# Nacos项目搭建

## 一、配置中心

### 添加依赖

```java
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-config</artifactId>
    <version>2.1.0.RELEASE</version>
    </dependency>

    <!-- https://mvnrepository.com/artifact/org.springframework.boot/spring-boot-starter-web -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <version>2.1.0.RELEASE</version>
    </dependency>
    <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <version>2.1.0.RELEASE</version>
    </dependency>
    <dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-core</artifactId>
    <version>2.6.2</version>
    </dependency>
```



### 写入bootstrap

在src/main/resources/bootstrap.properties加入以下内容

```java
spring.application.name=example
spring.cloud.nacos.config.server-addr=119.23.64.10:8848
```

spring.application.name会成为Configuration的Data ID的一部分，Data ID的组成如下：

```
${prefix}-${spring.profile.active}.${file-extension}
```

**prefix**的默认值就是spring.application.name，也可以通过spring.cloud.nacos.config.prefix配置

**spring.profile.active**即项目所在profile，如test、dev、prod，若为空，则Data ID会省略掉这一部分内容

**file-extension**是配置内容的数据格式，目前支持properties和yaml，可以通过spring.cloud.nacos.config.file-extension配置



### 注入配置举例

完成上面两步后，应用就会从Nacos Server获取外部配置，并添加到spring的Environment中。可以通过使用@Value注解来注入这些配置，举例如下:

```
@Value("${user.name}")
String userName;

@Value("${user.age}")
int age;
```



### 开启Nacos Server

虽然在spring中配置了nacos-cofngi-server-addr，但其实server还没启动，要在https://github.com/alibaba/nacos/releases下载压缩包，完成后解压进入文件目录，执行以下命令

```
sh startup.sh -m standalone
```

之后开发服务端口8848，打开浏览器输入ip:8848/nacos即可，初始账号密码均为nacos



### 编写Controller

```java
@RestController
@RequestMapping("/config")
@RefreshScope
public class ConfigController {

    @Value("${useLocalCache:false}")
    private boolean useLocalCache;

    @RequestMapping("/get")
    public boolean get() {
        return useLocalCache;
    }
}
```

@RefreshScope使Spirngcloud开启自动刷新配置功能



### 创建配置

通过使用linux命令行创建nacos配置

```
curl -X POST "http://127.0.0.1:8848/nacos/v1/cs/configs?dataId=example.properties&group=DEFAULT_GROUP&content=useLocalCache=true"
```

启动nacos-example，访问http://localhost:8080/config/get，发现返回true

修改配置如下：

```
curl -X POST "http://127.0.0.1:8848/nacos/v1/cs/configs?dataId=example.properties&group=DEFAULT_GROUP&content=useLocalCache=false"
```

http://localhost:8080/config/get,发现返回false



## 二、服务发现

当启用服务发现功能后，就可以将当前项目注册到nacos-server上

### 添加依赖

```
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
    <version>2.1.0.RELEASE</version>
</dependency>
<dependency>
    <groupId>org.apache.httpcomponents</groupId>
    <artifactId>httpclient</artifactId>
    <version>4.5.3</version>
</dependency>
```



### 配置服务属性

在src/resources/application.properties下加入以下配置:

```
server.port=8070
spring.application.name=service-provider
spring.cloud.nacos.discovery.server-addr=119.23.64.10:8848
```



### @EnableDiscoveryClient

在启动类上加加上该注解



再次启动项目，可以发现项目已经注册在nacos server上了，当有消费者需要调用该接口，只需要注册到nacos server就能通过如下方式访问

```
restTemplate.getForObject("http://service-provider/config/get", String.class)
```



