# OAuth2入门(一)——Provider

介绍分为两部分——OAuth2 Provider和OAuth2 Client。这里先讲OAuth2 Provider



## 一、介绍

provider主要用于暴露OAuth2的受保护资源。他的配置使client能**获取这些资源**或让client能**代表一个用户**。provider通过管理和校验token来使client获取资源。同时，provider必须提供一个接口来使用户能够对资源进行授权访问。



## 二、Provider实现

provider这一角色在OAuth2中实际被分为Authorization Service(授权服务)和Resource application(资源服务)，通过spring security你可以将他们拆分到两个应用，并且可以实现多个资源服务共享同一个授权服务。**client对令牌的请求**(授权请求)被SpringMVC controller的endpoints处理，**client对受保护资源的访问**通过标准的Spring Security请求过滤器处理。

1.要实现OAuth2的Authorization Server(授权服务器)，以下两个端点需要在Spring Security filter chain中声明：

**AuthorizationEndPoint**：用于授权服务的请求，默认URL:/oauth/authorize

**TokenEndpoint**:用于access token的请求，默认URL:/oauth/token



2.要实现OAuth2的Resource Server(资源服务器)，则需要以下过滤器：

**OAuth2AuthenticationProcessingFilter**：用于认证一个带有token的请求



## 三、授权服务的配置

在配置授权服务器时，必须考虑客户端用于从最终用户获取访问令牌的授权类型。配置必须实现**client的详细服务**、实现**token服务**以及实现**全局启用禁用该机制**的功能。注意，还可以为每个客户端专门配置权限，使其能够使用特定的授权机制和授权类型。

可以通过@EnableAuthorizationServer注解，以及任何继承了**AuthorizationServerConfigurer**接口的类(或继承这个类的适配器**ResourceServerConfigurerAdapter**)来配置授权服务。

以下属性被委托给Spring创建并传递到AuthorizationServerConfigurer的独立配置器，在其构造方法中被重写实现，也就是说，要配置以下属性，可以通过继承**ResourceServerConfigurer(Adapter)**类，并在其重写的configure方法中进行相关编码：

**ClientDetailsServiceConfigurer**

定义client详细服务的配置程序。可以初始化client详细信息，也可以引用现有模板。

**AuthorizationServerSecurityConfigurer**

定义TokenEndpoint上的安全约束

**AuthorizationServerEndpointsConfigurer**

定义AuthorizationEndPoint和TokenEndpoint以及token服务。



有一个重要的地方在于，在provider将授权代码提供给OAuth client的方式下，授权码由OAuth客户端通过将终端用户引导到一个**授权页面**来获得，终端用户可以在该页面中输入自己的凭据，并从提供授权码的provider处重定向回OAuth客户端。授权服务的主要配置有以下几个方面：



### 1.配置client详细信息

前面讲到的三个Configurer中，**ClientDetailsServiceConfigurer**用于定义client详细信息服务(以下称ClientDetails)的内存实现或JDBC实现。一个client重要的属性如下：

**clientId**：客户端id

**secret**：客户端密码

**scope**：客户端的受限范围

**authorizedGrantTypes**：认证类型

**authorities**：客户端被授予的权限



通过直接访问底层存储(如数据库)或ClientDetailsManager接口（实现了ClientDetailsService接口的都可以），可以在运行时更新ClientDetails



### 2.管理token

**AuthorizationServerTokenServices**接口定义了管理令牌的必要实现，需要注意以下两点：

```
1.创建token时，必须存储，以便token对应的资源可以在以后识别它
2.token的功能是加载一个authentication(认证)，这个认证被用来授权token的创建
```

当创建AuthorizationServerTokenServices实现类时，可以使用默认的DefaultTokenServices，它包含了很多可以插入的策略来更改访问token的格式和存储token。默认情况下，它通过随机值创建令牌，并处理除了被委托给TokenStore的持久化令牌之外的所有事务。token默认的存储是在内存中实现的，但是还有其他一些可用的实现。如使用jdbc或redis存储code



### 3.授权类型

**AuthorizationEndpoint**支持的授权类型(grant types)可以通过AuthorizationServerEndpointsConfigurer进行配置，默认情况下，除了密码之外，所有的授权类型该类都是支持的。先来看一下OAuth2中的五种授权类型：

- 授权码模式（authorization code）
- 简化模式（implicit）
- 密码模式（resource owner password credentials）
- 客户端模式（client credentials）
- 扩展模式（Extension）

下列属性会影响授权类型:

**authenticationManager**：通过注入AuthenticationManager来开启**密码授权类型**

**userDetailService**：如果注入了一个UserDetailsService或者已经存在该类的全局配置(如配置在一个GlobalAuthenticationConfigurerAdapter中)，那么refreshtoken将包含对用户详细信息的检查，以确保帐户仍然处于活动状态

**authorizationCodeServices**：为授权码(authorizationCode)定义服务(AuthorizationCodeServices实例)，如授权码的存储功能等。

**implicitGrantService**：管理implicit授权期间的状态。

**tokenGranter**：TokenGranter(完全控制授权并忽略上面的其他属性)



### 4.配置endpoint url

如果需要配置自己的endpoint url，可以使用AuthorizationServerEndpointsConfigurer 的pathMapping方法，它需要传入两个参数：1.默认的url路径；2.需要的自定义路径(以“/”开头)。OAuth2提供的url路径为以下几个：

```
/oauth/authorize		用于授权
/oauth/token			用于提供token服务
/oauth/confirm_access	用户用户获取资源
/oauth/error			用于授权服务处理错误
/oauth/check_token		用于资源服务器解码token
/oauth/check_token      若使用JWT令牌则公开用于验证的公钥
```

需要注意的是/oauth/authorize应该使用Spring security进行保护，以便仅对经过身份验证的用户进行访问。比如通过继承WebSecurityConfigurer(Adapter)并重写configure方法来实现：

```java
@Override
    protected void configure(HttpSecurity http) throws Exception {
        http
            .authorizeRequests().antMatchers("/login").permitAll().and()
        // default protection for all resources (including /oauth/authorize)
            .authorizeRequests()
                .anyRequest().hasRole("USER")
        // ... more configuration, e.g. for form login
}
```