# OAuth2 授权服务搭建(一)

## 1.前言

前面在OAuth授权服务概述中大致介绍了OAuth授权服务的一些核心内容，接下来进行授权服务的搭建。搭建之前，先预设用户信息及client信息存储在mysql中，token、授权码、session等服务使用redis进行控制



## 2.引入项目依赖

```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>2.1.13.RELEASE</version>
</parent>

<properties>
    <java.version>1.8</java.version>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
    <spring-boot.version>2.3.0.RELEASE</spring-boot.version>
</properties>

<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-jdbc</artifactId>
        <version>2.1.13.RELEASE</version>
    </dependency>

    <dependency>
        <groupId>mysql</groupId>
        <artifactId>mysql-connector-java</artifactId>
        <version>8.0.19</version>
    </dependency>

    <dependency>
        <groupId>org.springframework.session</groupId>
        <artifactId>spring-session-data-redis</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework.security.oauth</groupId>
        <artifactId>spring-security-oauth2</artifactId>
        <version>2.3.3.RELEASE</version>
    </dependency>

    <dependency>
        <groupId>com.baomidou</groupId>
        <artifactId>mybatis-plus-boot-starter</artifactId>
        <version>2.3</version>
    </dependency>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
        <exclusions>
            <exclusion>
                <groupId>org.junit.vintage</groupId>
                <artifactId>junit-vintage-engine</artifactId>
            </exclusion>
        </exclusions>
    </dependency>
</dependencies>
```



## 3.编写配置文件

```yml
spring:
  application:
    name: oauth
  datasource:
    url: jdbc:mysql://localhost:3306/user?useUnicode=true&characterEncoding=utf-8&serverTimezone=GMT&useSSL=false
    username: root
    password: 123456
  redis:
    host: 127.0.0.1
      port: 6379
      password: 123456
      database: 10
  session:
    store-type: redis
    redis:
      flush-mode: on_save
      namespace: auth

server:
  port: 8765
  servlet:
    context-path: oauth
    session:
      cookie:
        path: /
        name: _oauth2_token_
        max-age: 86400

```



## 4.创建数据库&代码生成

### 4.1 user表

```sql
SET NAMES utf8mb4;
SET FOREIGN_KEY_CHECKS = 0;

-- ----------------------------
-- Table structure for user
-- ----------------------------
DROP TABLE IF EXISTS `user`;
CREATE TABLE `user`  (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `username` varchar(64) CHARACTER SET utf8 COLLATE utf8_general_ci DEFAULT NULL,
  `password` varchar(64) CHARACTER SET utf8 COLLATE utf8_general_ci DEFAULT NULL,
  `gmt_update` datetime(0) DEFAULT NULL ON UPDATE CURRENT_TIMESTAMP(0),
  `gmt_create` datetime(0) DEFAULT CURRENT_TIMESTAMP,
  `status` tinyint(4) DEFAULT 1 ,
  PRIMARY KEY (`id`) USING BTREE
) ENGINE = InnoDB AUTO_INCREMENT = 1 CHARACTER SET = utf8 COLLATE = utf8_general_ci ROW_FORMAT = Dynamic;

SET FOREIGN_KEY_CHECKS = 1;
```



### 4.2 client表

```sql
SET NAMES utf8mb4;
SET FOREIGN_KEY_CHECKS = 0;

-- ----------------------------
-- Table structure for oauth_client_details
-- ----------------------------
DROP TABLE IF EXISTS `client`;
CREATE TABLE `client`  (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `client_id` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci DEFAULT NULL,
  `client_secret` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci DEFAULT NULL,
  `name` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci DEFAULT NULL,
  `scope` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci DEFAULT 'demo',
  `grant_types` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci DEFAULT NULL,
  `autoapprove` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci DEFAULT '',
  `access_token_validity` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci DEFAULT '9000',
  `refresh_token_validity` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci DEFAULT '90000',
  `authorities` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci DEFAULT NULL,
  `status` tinyint(4) DEFAULT NULL,
  `create_time` datetime(0) DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `update_time` datetime(0) DEFAULT CURRENT_TIMESTAMP COMMENT '修改时间',
  PRIMARY KEY (`id`) USING BTREE
) ENGINE = InnoDB AUTO_INCREMENT = 1 CHARACTER SET = utf8 COLLATE = utf8_general_ci ROW_FORMAT = Dynamic;

SET FOREIGN_KEY_CHECKS = 1;
```



### 4.3 代码生成

这里使用mybatis代码生成器生成实体类、Service(Impl)、Controller、Dao、Mapper等文件。详见github：https://github.com/kylsky/oauth2



## 5.配置启动类

```java
@SpringBootApplication
@EnableRedisHttpSession
@EnableAuthorizationServer
public class AuthenticationApplication {
    public static void main(String[] args) {
        SpringApplication.run(AuthenticationApplication.class, args);
    }
}
```



## 6.Configurer配置

Configurer的配置是OAuth2授权服务的重头戏，这里新建一个config包并创建OAuth2Configurer类，实现AuthorizationServerConfigurer接口，重写相关接口方法：

```java
@Configuration
public class OAuth2Configuration implements AuthorizationServerConfigurer {
    @Override
    public void configure(ClientDetailsServiceConfigurer clients) throws Exception {
		//TODO
    }
    
    @Override
    public void configure(AuthorizationServerEndpointsConfigurer endpoints) throws Exception {
		//TODO
    }
    
    @Override
    public void configure(AuthorizationServerSecurityConfigurer security) throws Exception {
		//TODO
    }
}
```



### 6.1 ClientDetailsServiceConfigurer

概述中讲到，OAuth2通过继承AuthorizationServerConfigurer并重写**configure(ClientDetailsServiceConfigurer clients)**方法来配置ClientDetailsService，由于使用mysql对client数据进行存储，因此配置如下：

```java
@Resource
private DataSource dataSource;

@Override
public void configure(ClientDetailsServiceConfigurer clients) throws Exception {
    //TODO
    clients.jdbc(dataSource);
}
```



### 6.2 AuthorizationServerEndpointsConfigurer

```
配置授权服务器端点的非安全性特性，如token存储，token定制，用户凭证和授权类型等。默认情况下你不需要做任何配置。除非你需要配置密码授权类型，那你就得额外配上AuthenticationManager
```

根据官方文档对**configure(AuthorizationServerEndpointsConfigurer endpoints)**方法的描述，来对其进行配置

```java
@Resource
private UserService userService;

@Resource
private AuthenticationManager authenticationManager;

@Resource
private AuthorizationCodeService authorizationCodeService;

@Resource
private TokenStore tokenStore;

@Resource
private RedirectResolver redirectResolver;

@Override
public void configure(AuthorizationServerEndpointsConfigurer endpoints) {
    endpoints
        .pathMapping("/oauth/check_token","/oauth/hello")
        .authenticationManager(authenticationManager)
        .userDetailsService(userService)
        .allowedTokenEndpointRequestMethods(HttpMethod.GET, HttpMethod.POST)
        .authorizationCodeServices(authorizationCodeService)
        .tokenStore(tokenStore)
        .accessTokenConverter(new TokenConverter())
        .redirectResolver(redirectResolver)
}
```

#### 6.2.1 pathMapping()

上述示例中，pathMapping("/oauth/check_token","/oauth/hello")用来表示将oauth2自带的/oauth/check_token映射到/oauth/hello，此时，oauth校验token时需要使用/oauth/hello作为路径，原有的路径会无法使用或被重写的接口代替。



#### 6.2.2 userDetailsService()

用于配置用户相关的服务，由于使用mysql作为user的数据存储，因此在mvc的service层中令UserService继承UserDetailsService，并在UserServiceImpl中继承loadUserByUsername方法即可

```java
//String类型参数用于传输用户名
public interface UserDetailsService {
    UserDetails loadUserByUsername(String var1) throws UsernameNotFoundException;
}
```

来简单实现一下UserServiceImpl：

```java
@Service
public class UserServiceImpl extends ServiceImpl<UserMapper, User> implements UserService {
    @Resource
    private UserMapper userMapper;

    @Override
    public UserDetails loadUserByUsername(String s) throws UsernameNotFoundException {
        List<User> users = userMapper.selectList(new EntityWrapper<User>()
                .eq("username",s));
        if (CollectionUtils.isEmpty(users)){
            throw new UsernameNotFoundException("用户名不存在");
        }else if (users.size()>1){
            throw new UsernameNotFoundException("用户名重复，联系管理员");
        }
        User user = users.get(0);
        return new org.springframework.security.core.userdetails.User(user.getUsername(),
                user.getPassword(),
                true,
                true,
                true,
                true,
                null);
    }
}
```

上述代码简单判断了用户名是否存在以及是否存在同名问题，若存在则抛出异常，其他异常类型可以在**AuthenticationException**的继承类中查看。在return的参数中，参数分别为：**用户名**、**密码**、**用户是否可用**、**用户是否有效**、**凭证是否有效**、**账户是否锁定**、**用户权限**，这里就简单配置一下以作参考。



#### 6.2.3 authorizationCodeServices()

使用Redis存储授权码

```java
@Configuration
public class AuthorizationCodeService extends RandomValueAuthorizationCodeServices {
    @Resource
    private StringRedisTemplate<String, OAuth2Authentication> stringRedisTemplate;

    @Override
    protected void store(String code, OAuth2Authentication authentication) {
        stringRedisTemplate.opsForValue().set(code, authentication, 1, TimeUnit.MINUTES);
    }

    @Override
    protected OAuth2Authentication remove(String code) {
        OAuth2Authentication auth2Authentication;
        try {
            auth2Authentication = stringRedisTemplate.opsForValue().get(code);
        } catch (Exception e) {
            return null;
        }
        if (auth2Authentication != null) {
            stringRedisTemplate.delete(code);
        }
        return auth2Authentication;
    }
}
```



#### 6.2.4 tokenStore()

tokenStore用于存储token，这里使用redis存储token，使用打印日志的方式模拟业务逻辑，如果需要存储更多信息，如用户的电话、邮箱等信息，则需要创建类并继承OAuth2Authentication，通过构造函数传入OAuth2AccessToken、OAuth2Authentication对象，并实现额外的需求。

```java
public class MyTokenStore extends RedisTokenStore {
    private static final Logger logger = LoggerFactory.getLogger(MyTokenStore.class);
    
    @Resource
    private UserService userService;

    public MyTokenStore(RedisConnectionFactory connectionFactory) {
        super(connectionFactory);
    }

    @Override
    public void storeAccessToken(OAuth2AccessToken token, OAuth2Authentication authentication) {
        logger.info(authentication.getName());
        super.storeAccessToken(token, authentication);
    }
}
```



#### 6.2.5 accessTokenConverter(）

TokenStore用于存储token、authentication，而TokenConverter相反的则从token和authentication中获取信息，并通过map传输。同样的，如果需要传输额外的用户信息，则需要创建类实现OAuth2Authentication，并在convertAccessToken中通过相关get方法获取信息并传入map。以下示例仅作为参考：

```java
public class TokenConverter extends DefaultAccessTokenConverter {
    @Override
    @SuppressWarnings("unchecked")
    public Map<String, ?> convertAccessToken(OAuth2AccessToken token, OAuth2Authentication authentication) {
        Map<String, Object> map = (Map<String, Object>) super.convertAccessToken(token, authentication);
        map.put("curDate",new Date());
        return map;
    }
}
```



#### 6.2.6 redirectResolver()

默认使用RedirectResolver



### 6.3 AuthorizationServerSecurityConfigurer

这里不做说明了，之前在概述里有讲过

```java
@Override
public void configure(AuthorizationServerSecurityConfigurer oauthServer) {
    oauthServer
        .tokenKeyAccess("permitAll()")
        .checkTokenAccess("permitAll()")
        .allowFormAuthenticationForClients()
        ;
}
```



## 7.SpringSecurity相关配置

这里也要配置挺多东西的，请看下回分解。