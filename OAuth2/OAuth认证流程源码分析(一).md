# OAuth2认证流程源码分析

本篇从/oauth/authorize作为入口探究oauth2的认证流程的源码分析。在分析之前，先以implicit模式为例梳理一下整个认证的大致流程，之后围绕下述步骤进行展开。

```
1.客户端发起implicit模式认证请求，形如http://localhost:12306/oauth/oauth/authorize?response_type=token&client_id=client&redirect_uri=http://www.baidu.com
2.oauth2框架检查步骤1的请求是否已经经过认证，若没有认证，则跳转登录页
3.在登陆页输入用户名密码并成功登录后，跳转至步骤1的redirect_uri，并携带token信息，完成认证
```



## 1.AuthorizationEndpoint#authorize

AuthorizationEndpoint中默认实现了/oauth/authorize接口，返回类型为ModelAndView，一看就是老SringMVC了。这里只讨论第一步骤，因此贴部分代码：

```java
@RequestMapping(value = "/oauth/authorize")
public ModelAndView authorize(Map<String, Object> model, @RequestParam Map<String, String> parameters,SessionStatus sessionStatus, Principal principal) {
    AuthorizationRequest authorizationRequest = getOAuth2RequestFactory().createAuthorizationRequest(parameters);
	//参数校验
    //……
	
    try {
        //此处通过principal检查当前请求是否经过认证，客户端在第一次请求时当然是没有的
        if (!(principal instanceof Authentication) || !((Authentication) principal).isAuthenticated()) {
            // 直接抛异常了
            throw new InsufficientAuthenticationException(
                "User must be authenticated with Spring Security before authorization can be completed.");
        }
        catch{
            ……
        }
        ……
    }
```



## 2.ExceptionTranslationFilter#doFilter

在AuthorizationEndpoint抛出异常后，ExceptionTranslationFilter将其捕获

```java
catch (Exception var10) {
    Throwable[] causeChain = this.throwableAnalyzer.determineCauseChain(var10);
    // 在这里ase对应的即是上面抛出的InsufficientAuthenticationException
    RuntimeException ase = (AuthenticationException)this.throwableAnalyzer.getFirstThrowableOfType(AuthenticationException.class, causeChain);
    // ……
    // 执行以下代码
    this.handleSpringSecurityException(request, response, chain, (RuntimeException)ase);
}
```



来看到handleSpringSecurityException方法：

```java
private void handleSpringSecurityException(HttpServletRequest request, HttpServletResponse response, FilterChain chain, RuntimeException exception) throws IOException, ServletException {
    //InsufficientAuthenticationException，会进入第一个条件判断，跳转sendStartAuthentication方法
    if (exception instanceof AuthenticationException) {
        this.logger.debug("Authentication exception occurred; redirecting to authentication entry point", exception);
        this.sendStartAuthentication(request, response, chain, (AuthenticationException)exception);
    } else if (exception instanceof AccessDeniedException) {
        //……
    }
}
```



来看到sendStartAuthentication方法：

```java
protected void sendStartAuthentication(HttpServletRequest request, HttpServletResponse response, FilterChain chain, AuthenticationException reason) throws ServletException, IOException {
    // 这里把SpringSecutiry的上下文中的Authentication置空了
    SecurityContextHolder.getContext().setAuthentication((Authentication)null);
    // 简单点来说，就是在request的session中塞入了一个属性，键为SPRING_SECURITY_SAVED_REQUEST，值为savedRequest对象，其中包含了request和一个portResolver
    this.requestCache.saveRequest(request, response);
    this.logger.debug("Calling Authentication entry point.");
    // commence表示开始的意思，这里意味着要进行认证了
    this.authenticationEntryPoint.commence(request, response, reason);
}
```



## 3.LoginUrlAuthenticationEntryPoint#commence

这里进入了LoginUrlAuthenticationEntryPoint的commence方法

```java
public void commence(HttpServletRequest request, HttpServletResponse response, AuthenticationException authException) throws IOException, ServletException {
    String redirectUrl = null;
    if (this.useForward) {
        // ……
    } else {
        // 这里构建了跳转的登录页地址
        redirectUrl = this.buildRedirectUrlToLoginPage(request, response, authException);
    }
	// 最终调用了response.sendRedirect
    this.redirectStrategy.sendRedirect(request, response, redirectUrl);
}
```



来看看buildRedirectUrlToLoginPage：

```java
protected String buildRedirectUrlToLoginPage(HttpServletRequest request, HttpServletResponse response, AuthenticationException authException) {
    // 获取登录页的url
    String loginForm = this.determineUrlToUseForThisRequest(request, response, authException);
    // 判断loginForm是绝对路径还是相对路径
    if (UrlUtils.isAbsoluteUrl(loginForm)) {
        // 如果是绝对路径，则直接返回，如http://www.baidu.com
        return loginForm;
    } else {
        // 如果是相对路径，那么就需要构建出一个新的url
        int serverPort = this.portResolver.getServerPort(request);
        String scheme = request.getScheme();
        RedirectUrlBuilder urlBuilder = new RedirectUrlBuilder();
        urlBuilder.setScheme(scheme);
        urlBuilder.setServerName(request.getServerName());
        urlBuilder.setPort(serverPort);
        urlBuilder.setContextPath(request.getContextPath());
        urlBuilder.setPathInfo(loginForm);
        if (this.forceHttps && "http".equals(scheme)) {
            Integer httpsPort = this.portMapper.lookupHttpsPort(serverPort);
            if (httpsPort != null) {
                urlBuilder.setScheme("https");
                urlBuilder.setPort(httpsPort);
            } else {
                logger.warn("Unable to redirect to HTTPS as no port mapping found for HTTP port " + serverPort);
            }
        }
        // 最终返回url
        return urlBuilder.getUrl();
    }
}
```



其实不难发现，跳转登录页的操作主要在determineUrlToUseForThisRequest这个方法，点进去可以发现，其实是返回了LoginUrlAuthenticationEntryPoint的loginFormUrl属性。而loginFormUrl属性则是在初始化LoginUrlAuthenticationEntryPoint时生成的，因此，在进行编码时需要对LoginUrlAuthenticationEntryPoint进行初始化，以下代码可以作为参考：

```java
@Configuration
public class DevConfiguration implements EnvironmentAvailable {
	@Bean
    public AuthenticationEntryPoint authenticationEntryPointBean() {
        return new LoginUrlAuthenticationEntryPoint("/page/login");
    }
}
```



## 4.Wait A Secend

到上面的代码为止，“认证未通过跳转至登录页”的步骤已经结束，大致来梳理一下整个流程：

```
1.authorize控制器检查认证参数与认证状态，发现未通过认证，抛出异常
2.异常处理过滤器捕获异常并处理，开始进行认证流程
3.构建登录页地址，并进行跳转
```

在进行下一步骤之前，再回顾下第二步中的sendStartAuthentication方法，这里通过设置一个cookie来标识当前请求的客户端，并通过savedRequest来保存该客户端的请求参数，这样，在这个客户端下一次成功登录时就能识别到他的认证信息并进行后续的页面跳转。



## 5.关于登录

oauth2可以拦截到发送到后端的登陆接口，比如/login，对url的拦截，自然是免不了Filter了，来看下debug模式下对login请求是如何拦截的：

<img src="http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20210108140446070.png" alt="image-20210108140446070" style="zoom: 67%;" />



### AbstractAuthenticationProcessingFilter#doFilter

```java
public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain)
    throws IOException, ServletException {

    HttpServletRequest request = (HttpServletRequest) req;
    HttpServletResponse response = (HttpServletResponse) res;
	// 在这一步校验是否需要判断登录，如果不是登录接口，则放行
    if (!requiresAuthentication(request, response)) {
        chain.doFilter(request, response);

        return;
    }

    if (logger.isDebugEnabled()) {
        logger.debug("Request is to process authentication");
    }

    Authentication authResult;
	// 下面就要开始进行认证操作了
    try {
        // 这一步是关键的认证流程
        authResult = attemptAuthentication(request, response);
        if (authResult == null) {
            // 没有完成认证，立即返回
            return;
        }
        sessionStrategy.onAuthentication(authResult, request, response);
    }
    catch (InternalAuthenticationServiceException failed) {
        logger.error(
            "An internal error occurred while trying to authenticate the user.",
            failed);
        unsuccessfulAuthentication(request, response, failed);

        return;
    }
    catch (AuthenticationException failed) {
        // Authentication failed
        unsuccessfulAuthentication(request, response, failed);

        return;
    }

    // Authentication success
    if (continueChainBeforeSuccessfulAuthentication) {
        chain.doFilter(request, response);
    }

    successfulAuthentication(request, response, chain, authResult);
}
```



## 6.UsernamePasswordAuthenticationFilter#attemptAuthentication

```java
public Authentication attemptAuthentication(HttpServletRequest request,
                                            HttpServletResponse response) throws AuthenticationException {
    if (postOnly && !request.getMethod().equals("POST")) {
        throw new AuthenticationServiceException(
            "Authentication method not supported: " + request.getMethod());
    }
	
    // 其实就是request.getParameter
    String username = obtainUsername(request);
    String password = obtainPassword(request);

    if (username == null) {
        username = "";
    }

    if (password == null) {
        password = "";
    }

    username = username.trim();

    // 将账号密码封装成一个UsernamePasswordAuthenticationToken对象，且认证状态为false
    UsernamePasswordAuthenticationToken authRequest = new UsernamePasswordAuthenticationToken(
        username, password);

    // 这里用来添加一些额外的属性
    setDetails(request, authRequest);

    // 使用AuthenticationManager来进行认证
    return this.getAuthenticationManager().authenticate(authRequest);
}
```



## 7.ProviderManager#authenticate

```java
public Authentication authenticate(Authentication authentication)
    throws AuthenticationException {
    Class<? extends Authentication> toTest = authentication.getClass();
    AuthenticationException lastException = null;
    AuthenticationException parentException = null;
    Authentication result = null;
    Authentication parentResult = null;
    boolean debug = logger.isDebugEnabled();

    // 这里通过获取provider来处理认证信息
    for (AuthenticationProvider provider : getProviders()) {
        if (!provider.supports(toTest)) {
            continue;
        }

        if (debug) {
            logger.debug("Authentication attempt using "
                         + provider.getClass().getName());
        }

        try {
            result = provider.authenticate(authentication);

            if (result != null) {
                copyDetails(authentication, result);
                break;
            }
        }
        catch (AccountStatusException e) {
            prepareException(e, authentication);
            throw e;
        }
        catch (InternalAuthenticationServiceException e) {
            prepareException(e, authentication);
            throw e;
        }
        catch (AuthenticationException e) {
            lastException = e;
        }
    }

    if (result == null && parent != null) {
        // 如果当前的manager无法处理认证，则让parentManager来处理，调用流程和当前方法一样
        try {
            result = parentResult = parent.authenticate(authentication);
        }
        catch (ProviderNotFoundException e) {
        }
        catch (AuthenticationException e) {
            lastException = parentException = e;
        }
    }

    if (result != null) {
        if (eraseCredentialsAfterAuthentication
            && (result instanceof CredentialsContainer)) {
            // 认证已完成，把密码清除掉防止泄露
            ((CredentialsContainer) result).eraseCredentials();
        }

        // 如果认证处理成功，就会发布一个success的事件，这个事件会由successHandler进行处理
        if (parentResult == null) {
            eventPublisher.publishAuthenticationSuccess(result);
        }
        return result;
    }

    // parent为空，或者没有provider来处理认证信息，就会抛出异常

    if (lastException == null) {
        lastException = new ProviderNotFoundException(messages.getMessage(
            "ProviderManager.providerNotFound",
            new Object[] { toTest.getName() },
            "No AuthenticationProvider found for {0}"));
    }

    // 认证失败的情况下，交给失败处理器来处理，eventPublisher.publishAuthenticationFailure(ex, auth)
    if (parentException == null) {
        prepareException(lastException, authentication);
    }

    throw lastException;
}
```



## 8.关于authenticate方法

![image-20210108144854505](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20210108144854505.png)

对于UsernamePasswordAuthenticationToken来说，默认的ProviderManager持有的Provider为anonymousAuthenticationProvider，无法处理，因此需要寻找Parent来处理，parent中的provider为DaoAuthenticationProvider，这个Provider其实并没有authenticate方法，真正执行其实在他的父类AbstractUserDetailsAuthenticationProvider中。



## 9.AbstractUserDetailsAuthenticationProvider#authenticate

这个方法的信息量非常的庞大！

```java
public Authentication authenticate(Authentication authentication)
    throws AuthenticationException {
    Assert.isInstanceOf(UsernamePasswordAuthenticationToken.class, authentication,
                        () -> messages.getMessage(
                            "AbstractUserDetailsAuthenticationProvider.onlySupports",
                            "Only UsernamePasswordAuthenticationToken is supported"));

    // 获取username
    String username = (authentication.getPrincipal() == null) ? "NONE_PROVIDED"
        : authentication.getName();

    boolean cacheWasUsed = true;
    UserDetails user = this.userCache.getUserFromCache(username);
	
    // 尝试用缓存拿User，如果运气不好，缓存里没有，那就要重新获取了~
    if (user == null) {
        
        cacheWasUsed = false;

        try {
            // 关键在这里，第10步分析，这个方法主要的任务就是根据用户名和密码返回一个UserDetails对象，来进行后续的操作
            user = retrieveUser(username,
                                (UsernamePasswordAuthenticationToken) authentication);
        }
        catch (UsernameNotFoundException notFound) {
            logger.debug("User '" + username + "' not found");

            if (hideUserNotFoundExceptions) {
                throw new BadCredentialsException(messages.getMessage(
                    "AbstractUserDetailsAuthenticationProvider.badCredentials",
                    "Bad credentials"));
            }
            else {
                throw notFound;
            }
        }

        Assert.notNull(user,
                       "retrieveUser returned null - a violation of the interface contract");
    }

    try {
        // 调用DefualtAuthenticationChecker#check，判断该用户是否过期，是否被锁定
        preAuthenticationChecks.check(user);
        // 密码校验，就是判断密码对不对
        additionalAuthenticationChecks(user,
                                       (UsernamePasswordAuthenticationToken) authentication);
    }
    catch (AuthenticationException exception) {
        if (cacheWasUsed) {
            cacheWasUsed = false;
            user = retrieveUser(username,
                                (UsernamePasswordAuthenticationToken) authentication);
            preAuthenticationChecks.check(user);
            additionalAuthenticationChecks(user,
                                           (UsernamePasswordAuthenticationToken) authentication);
        }
        else {
            throw exception;
        }
    }
	// 校验凭证（密码）是否过期
    postAuthenticationChecks.check(user);

    if (!cacheWasUsed) {
        this.userCache.putUserInCache(user);
    }

    Object principalToReturn = user;

    if (forcePrincipalAsString) {
        principalToReturn = user.getUsername();
    }
	// 在第11步分析，这里做的是
    return createSuccessAuthentication(principalToReturn, authentication, user);
}
```



## 10.DaoAuthenticationProvider#retrieveUser

```java
protected final UserDetails retrieveUser(String username,
                                         UsernamePasswordAuthenticationToken authentication)
    throws AuthenticationException {
    prepareTimingAttackProtection();
    try {
        // 关键的代码在这里，通过自定义实现一个UserService来返回一个用户对象
        UserDetails loadedUser = this.getUserDetailsService().loadUserByUsername(username);
        if (loadedUser == null) {
            throw new InternalAuthenticationServiceException(
                "UserDetailsService returned null, which is an interface contract violation");
        }
        return loadedUser;
    }
    catch (UsernameNotFoundException ex) {
        mitigateAgainstTimingAttack(authentication);
        throw ex;
    }
    catch (InternalAuthenticationServiceException ex) {
        throw ex;
    }
    catch (Exception ex) {
        throw new InternalAuthenticationServiceException(ex.getMessage(), ex);
    }
}
```



## 11.DaoAuthenticationProvider#createSuccessAuthentication

```java
protected Authentication createSuccessAuthentication(Object principal,
                                                     Authentication authentication, UserDetails user) {
    // 判断密码的安全性，决定是否需要再次加密
    boolean upgradeEncoding = this.userDetailsPasswordService != null
        && this.passwordEncoder.upgradeEncoding(user.getPassword());
    if (upgradeEncoding) {
        String presentedPassword = authentication.getCredentials().toString();
        String newPassword = this.passwordEncoder.encode(presentedPassword);
        user = this.userDetailsPasswordService.updatePassword(user, newPassword);
    }
    // 这里又回到父类的方法
    return super.createSuccessAuthentication(principal, authentication, user);
}
```



### AbstractUserDetailsAuthenticationProvider#createSuccessAuthentication

```java
protected Authentication createSuccessAuthentication(Object principal,
                                                     Authentication authentication, UserDetails user) {
    // 确保我们返回用户提供的原始凭证，这样即使使用编码的密码，后续尝试也会成功。还要确保返回原始的getDetails()，以便在缓存到期后的未来身份验证事件中包含详细信息
    UsernamePasswordAuthenticationToken result = new UsernamePasswordAuthenticationToken(
        principal, authentication.getCredentials(),
        authoritiesMapper.mapAuthorities(user.getAuthorities()));
    result.setDetails(authentication.getDetails());

    return result;
}
```



## 12.回到ProviderManager

定位到以下的代码，在上面的一步步流程走下来后，接下来就是最后一步了，如果登陆正常，那么就会由成功处理器处理，如果抛出了异常，那么异常就会在ProviderManager内被捕获，并由失败处理器处理。

```java
if (result != null) {
    if (eraseCredentialsAfterAuthentication
        && (result instanceof CredentialsContainer)) {
        // 认证已完成，把密码清除掉防止泄露
        ((CredentialsContainer) result).eraseCredentials();
    }

    // 如果认证处理成功，就会发布一个success的事件，这个事件会由successHandler进行处理
    if (parentResult == null) {
        eventPublisher.publishAuthenticationSuccess(result);
    }
    return result;
}

// parent为空，或者没有provider来处理认证信息，就会抛出异常

if (lastException == null) {
    lastException = new ProviderNotFoundException(messages.getMessage(
        "ProviderManager.providerNotFound",
        new Object[] { toTest.getName() },
        "No AuthenticationProvider found for {0}"));
}

// 认证失败的情况下，交给失败处理器来处理，eventPublisher.publishAuthenticationFailure(ex, auth)
if (parentException == null) {
    prepareException(lastException, authentication);
}

throw lastException;
```



## 13.successHandler & failureHandler

关于successHandler和failureHandler的配置如下需要在继承了WebSecurityConfigurerAdapter的类的configure(HttpSecurity http)方法中通过以下两个方法注入。

```
.successHandler(getSuccessHandler())
.failureHandler(getFailureHandler())
```

下面放两个最基本的处理方法：

### getSuccessHandler

```java
private RequestCache requestCache = new HttpSessionRequestCache();

private AuthenticationSuccessHandler getSuccessHandler() {
    return (request, response, authentication) -> {
    PrintWriter out = response.getWriter();
    out.write("{'success':'true','callback':'http://http://localhost:12306/oauth/oauth/authorize?response_type=token&client_id=client&redirect_uri=http://www.baidu.com'}");
    out.flush();
    out.close();
    };
}
```

### getFailureHandler

```java
private AuthenticationFailureHandler getFailureHandler() {
    return (request, response, authentication) -> {
            PrintWriter out = response.getWriter();
            out.write("{'success':'false'}");
            out.flush();
            out.close();
        };
    }
```



## 14.小结

通过以上的步骤，直到用户登陆的逻辑已经完成，不过事情当然没有这么简单，登陆成功后如何跳转需要访问的页面？token又在什么时机生成？请听下回分解~