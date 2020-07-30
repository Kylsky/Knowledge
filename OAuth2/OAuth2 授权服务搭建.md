# OAuth2 授权服务

## 一、前言

在我看来，OAuth2的整体的学习难度还是稍微有点大的，另外在学习OAuth2之前，需要知道一些spring security的前置知识，虽然不是必须的，但是能起到比较好的助推学习的作用。由于整个OAuth2的授权服务比较复杂，因此在此写篇文章做个记录。



## 二、思路

OAuth2授权服务的关键在于**@EnableAuthorizationServer**注解以及一个实现**AuthorizationServerConfigurer**接口的类并重写相关configure配置方法(或继承这个类的适配器**ResourceServerConfigurerAdapter**)。

重写3个configure方法，核心内容在于配置以下信息：

1. ClientDetailsService；
2. AuthorizationServerEndpointsConfigurer；
3. AuthorizationServerSecurityConfigurer。

因此，全文将会围绕以上几点进行展开，虽然东西很多，但是按照流程理清思路，就不至于无从下手。



## 三、@EnableAuthorizationServer

用来在当前应用context里(必须是一个DispatcherServlet context)开启一个授权server(例如AuthorizationEndpoint)和一个TokenEndpoint。server的多个属性可以通过自定义AuthorizationServerConfigurer类型(如AuthorizationServerConfigurerAdapter的扩展)的Bean来定制。通过正常使用spring security的特色EnableWebSecurity，用户负责保证授权Endpoint(/oauth/authorize)的安全，但Token Endpoint(/oauth/token)将自动使用http basic的客户端凭证来保证安全。通过一个或者多个AuthorizationServerConfigurers提供一个ClientDetailService来注册client(必须)

```
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import({AuthorizationServerEndpointsConfiguration.class, AuthorizationServerSecurityConfiguration.class})
public @interface EnableAuthorizationServer {

}
```

通过代码和翻译成中文的注释不难理解，要想实现授权服务器，OAuth2规定了必须的操作，也就是在第二节中提到的三点，来一一看一下。



## 四、ClientDetailsService

OAuth2通过继承**ResourceServerConfigurerAdapter**并重写以下方法来配置ClientDetailsService：

```java
@Override
public void configure(ClientDetailsServiceConfigurer clients) throws Exception {
    //TODO
}
```

来看看作为参数，ClientDetailsServiceConfigurer是怎么配置ClientDetailsService的：

```java
public class ClientDetailsServiceConfigurer extends
		SecurityConfigurerAdapter<ClientDetailsService, ClientDetailsServiceBuilder<?>> {
	//构造器，传入builder
	public ClientDetailsServiceConfigurer(ClientDetailsServiceBuilder<?> builder) {
		setBuilder(builder);
	}
	//根据传入的ClientDetailsService配置
	public ClientDetailsServiceBuilder<?> withClientDetails(ClientDetailsService clientDetailsService) throws Exception {
		setBuilder(getBuilder().clients(clientDetailsService));
		return this.and();
	}

    //配置ClientDetailsService为内存模式
	public InMemoryClientDetailsServiceBuilder inMemory() throws Exception {
		InMemoryClientDetailsServiceBuilder next = getBuilder().inMemory();
		setBuilder(next);
		return next;
	}
    //配置ClientDetailsService为jdbc模式
	public JdbcClientDetailsServiceBuilder jdbc(DataSource dataSource) throws Exception {
		JdbcClientDetailsServiceBuilder next = getBuilder().jdbc().dataSource(dataSource);
		setBuilder(next);
		return next;
	}
    //此处略过一些无关方法
    ……
}
```

可以发现，OAuth2提供了3种定义ClientDetailsService的方法：

1. 内存模式，在内存中存储client
2. jdbc模式，将client信息存储在数据库
3. 自定义的ClientDetailsService，如可以将client存储在redis或者mongo

尝试使用比较熟悉的jdbc作为示例：

```
@Resource
private DataSource dataSource;

@Override
public void configure(ClientDetailsServiceConfigurer clients) throws Exception {
    //TODO
    clients.jdbc(dataSource);
}
```

没错，就是这么简单了。



## 五、AuthorizationServerEndpoints

看到这么长的一串英文，你就应该想到事情并不简单,来看看OAuth2是怎么说的吧

```java
/**
	 * Configure the non-security features of the Authorization Server endpoints, like token store, token
	 * customizations, user approvals and grant types. You shouldn't need to do anything by default, unless you need
	 * password grants, in which case you need to provide an {@link AuthenticationManager}.
	 * 
	 * @param endpoints the endpoints configurer
	 */
	void configure(AuthorizationServerEndpointsConfigurer endpoints) throws Exception;
```

其实把注释翻译过来也不难理解：

```
配置授权服务器端点的非安全性特性，如token存储，token定制，用户凭证和授权类型等。默认情况下你不需要做任何配置。除非你需要配置密码授权类型，那你就得额外配上AuthenticationManager
```

当然了，使用默认配置那是不存在的，我们的目标就是搞事情~因此打开**AuthorizationServerEndpointsConfigurer**类的源码介绍一些常用的配置方法

```java
public final class AuthorizationServerEndpointsConfigurer {
    //设置TokenStore，用于存储token
	public AuthorizationServerEndpointsConfigurer tokenStore(TokenStore tokenStore) {
		this.tokenStore = tokenStore;
		return this;
	}
    
    //设置重定向处理器
    public AuthorizationServerEndpointsConfigurer redirectResolver(RedirectResolver redirectResolver) {
		this.redirectResolver = redirectResolver;
		return this;
	}
    
    //用于映射oauth自带的接口，映射后的地址会替换原来的地址，如pathMapping("/oauth/check_token", "/oauth/haha")
    public AuthorizationServerEndpointsConfigurer pathMapping(String defaultPath, String customPath) {
		this.patternMap.put(defaultPath, customPath);
		return this;
	}
    
    //设置允许的请求类型
    public AuthorizationServerEndpointsConfigurer allowedTokenEndpointRequestMethods(HttpMethod... requestMethods) {
		Collections.addAll(allowedTokenEndpointRequestMethods, requestMethods);
		return this;
	}
    
    //如果需要启动password授权模式，那么authenticationManager就必须得配置了
    public AuthorizationServerEndpointsConfigurer authenticationManager(AuthenticationManager authenticationManager) {
		this.authenticationManager = authenticationManager;
		return this;
	}
    
    //设置UserDetailsService
    public AuthorizationServerEndpointsConfigurer userDetailsService(UserDetailsService userDetailsService) {
		if (userDetailsService != null) {
			this.userDetailsService = userDetailsService;
			this.userDetailsServiceOverride = true;
		}
		return this;
	}
    
    //设置授权码模式下的code服务
    public AuthorizationServerEndpointsConfigurer authorizationCodeServices(
			AuthorizationCodeServices authorizationCodeServices) {
		this.authorizationCodeServices = authorizationCodeServices;
		return this;
	}
    
    //设置accessTokenConverter，accessTokenConverter用于返回token时进行额外的操作，如额外返回一些自定义的字段
    public AuthorizationServerEndpointsConfigurer accessTokenConverter(AccessTokenConverter accessTokenConverter) {
		this.accessTokenConverter = accessTokenConverter;
		return this;
	}
}
```



## 六、AuthorizationServerSecurityConfigurer

一看这名字就知道和安全脱不了关系了，来看看源码上的注释：

```java
/**
	 * Configure the security of the Authorization Server, which means in practical terms the /oauth/token endpoint. The
	 * /oauth/authorize endpoint also needs to be secure, but that is a normal user-facing endpoint and should be
	 * secured the same way as the rest of your UI, so is not covered here. The default settings cover the most common
	 * requirements, following recommendations from the OAuth2 spec, so you don't need to do anything here to get a
	 * basic server up and running.
	 * 
	 * @param security a fluent configurer for security features
	 */
	void configure(AuthorizationServerSecurityConfigurer security) throws Exception;

```

翻译一下：

```
配置授权服务器的安全性，即实际意义上的/oauth/token端点。当然，/oauth/authorize端点也需要是安全的，但是那是一个普通的面向用户的端点，应该像UI的其他部分一样安全，所以就是不需要在这里配置的意思了。默认设置涵盖了最常见的需求，遵循OAuth2规范的建议，因此在这里您不需要做任何事情就可以启动并运行基本服务器。
```

再次重申：使用默认配置那是不存在的，来看看我们能做什么：

```java
public final class AuthorizationServerSecurityConfigurer extends
		SecurityConfigurerAdapter<DefaultSecurityFilterChain, HttpSecurity> {
        
}
```







