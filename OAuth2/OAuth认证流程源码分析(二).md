# OAuth2认证流程源码分析(二)

## 1.回顾

前一篇文章讲解了客户端从调用/oauth/authorize接口->判断认证->跳转登录->用户名密码校验->成功及失败处理的一些主要流程，接下来的内容会从前文讲到的请求成功处理器展开，因为只有登陆成功以后，后续的访问页面以及相关的token生成才能被顺利展开。

在开始讲请求成功处理器之前，来看下讲到过的简单的认证流程，前一篇文章主要讲完了第1、2步，以及第3步的登陆成功操作。**跳转至步骤1的redirect_uri，并携带token信息，完成认证**，是本文主要讨论的内容

```
1.客户端发起implicit模式认证请求，形如http://localhost:12306/oauth/oauth/authorize?response_type=token&client_id=client&redirect_uri=http://www.baidu.com
2.oauth2框架检查步骤1的请求是否已经经过认证，若没有认证，则跳转登录页
3.在登陆页输入用户名密码并成功登录后，跳转至步骤1的redirect_uri，并携带token信息，完成认证
```

需要注意的是，第1步中的url是极其关键的，这个url表达了我们希望通过认证（调用authorize接口）并携带token跳转至指定的地方（redirect_uri），在第一次的认证中，我们由于并没有进行登录，所以会被oauth2要求进行登录校验。在认证通过后，其实从原则上我们应该仍希望通过上述的url来进行通过认证及跳转页面的操作。

此时前文讲到的savedRequest又浮出水面了。(在上一篇的第2步中sendStartAuthentication提到过)



## 2.ExceptionTranslationFilter#sendStartAuthentication

```java
private RequestCache requestCache = new HttpSessionRequestCache();


protected void sendStartAuthentication(HttpServletRequest request,
                                       HttpServletResponse response, FilterChain chain,
                                       AuthenticationException reason) throws ServletException, IOException {
    SecurityContextHolder.getContext().setAuthentication(null);
    // 这就是关键
    requestCache.saveRequest(request, response);
    logger.debug("Calling Authentication entry point.");
    authenticationEntryPoint.commence(request, response, reason);
}
```



## 3.HttpSessionRequestCache#saveRequest

```java
public void saveRequest(HttpServletRequest request, HttpServletResponse response) {
    if (requestMatcher.matches(request)) {
        // 生成一个savedRequst
        DefaultSavedRequest savedRequest = new DefaultSavedRequest(request,
                                                                   portResolver);

        if (createSessionAllowed || request.getSession(false) != null) {
            // 在request中设置属性，键为"SPRING_SECURITY_SAVED_REQUEST",值为savedRequest
            request.getSession().setAttribute(this.sessionAttrName, savedRequest);
            logger.debug("DefaultSavedRequest added to Session: " + savedRequest);
        }
    }
    else {
        logger.debug("Request not saved as configured RequestMatcher did not match");
    }
}
```

来看看这个DefaultSavedRequest，其实可以理解成封装了request请求中的一些属性

```java
private final ArrayList<SavedCookie> cookies = new ArrayList<>();
private final ArrayList<Locale> locales = new ArrayList<>();
private final Map<String, List<String>> headers = new TreeMap<>(
String.CASE_INSENSITIVE_ORDER);
private final Map<String, String[]> parameters = new TreeMap<>();
private final String contextPath;
private final String method;
private final String pathInfo;
private final String queryString;
private final String requestURI;
private final String requestURL;
private final String scheme;
private final String serverName;
private final String servletPath;
private final int serverPort;
```



## 4.getSuccessHandler

到此，终于可以讲successHandler了，前文讲到下面的代码，其实很简略：

```java
private RequestCache requestCache = new HttpSessionRequestCache();
private AuthenticationSuccessHandler getSuccessHandler() {
    return (request, response, authentication) -> {
    PrintWriter out = response.getWriter();
    out.write("{'success':'true','callback':'http://localhost:12306/oauth/oauth/authorize?response_type=token&client_id=client&redirect_uri=http://www.baidu.com'}");
    out.flush();
    out.close();
    };
}
```

现在使用SavedRequest，就能引入每一个请求需要跳转的地址：

![image-20210112103444237](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20210112103444237.png)

改进后的代码：

```java
private RequestCache requestCache = new HttpSessionRequestCache();
SavedRequest savedRequest = requestCache.getRequest(request, response);
private AuthenticationSuccessHandler getSuccessHandler() {
    return (request, response, authentication) -> {
    PrintWriter out = response.getWriter();
    JSONObject result = new JSONObject();
    result.put("success",true);
    // 这里的redirectUri就是最开始的那串/oauth/authorize的连接
    result.put("callback",savedRequest.getRedirectUri());
    out.write(result.toJSONString());
    out.flush();
    out.close();
    };
}
```

由于这里是把响应回写给前端的，因此前端只需要通过window.location.href进行跳转即可，相当于重新向http://localhost:12306/oauth/oauth/authorize?response_type=token&client_id=client&redirect_uri=http://www.baidu.com发起了请求，不过这个时候会带上Cookie作为之前同一个请求的标识。于是一切又回到了最开始的**AuthorizationEndpoint#authorize**



## 5.AuthorizationEndpoint#authorize

```java
@RequestMapping(value = "/oauth/authorize")
public ModelAndView authorize(Map<String, Object> model, @RequestParam Map<String, String> parameters,
                              SessionStatus sessionStatus, Principal principal) {

    // ……省略一些之前讲过的东西
    try {

        // ……

        // 认证已通过后，代码流程会从这里开始
        // 首先通过clientId加载一个ClientDetails对象
        ClientDetails client = getClientDetailsService().loadClientByClientId(authorizationRequest.getClientId());

        // 解析的重定向URI是来自参数的redirect_uri或来自clientDetails的redirect_uri。无论采用哪种方式，我们都需要将其存储在AuthorizationRequest上。
        String redirectUriParameter = authorizationRequest.getRequestParameters().get(OAuth2Utils.REDIRECT_URI);
        String resolvedRedirect = redirectResolver.resolveRedirect(redirectUriParameter, client);
        if (!StringUtils.hasText(resolvedRedirect)) {
            throw new RedirectMismatchException(
                "A redirectUri must be either supplied or preconfigured in the ClientDetails");
        }
        // authorizationRequest存储了一些授权信息
        authorizationRequest.setRedirectUri(resolvedRedirect);

        // 对请求中的参数进行scope校验
        oauth2RequestValidator.validateScope(authorizationRequest, client);

        // 设置approved标志位
        authorizationRequest = userApprovalHandler.checkForPreApproval(authorizationRequest,
                                                                       (Authentication) principal);
        // TODO: is this call necessary?
        boolean approved = userApprovalHandler.isApproved(authorizationRequest, (Authentication) principal);
        authorizationRequest.setApproved(approved);

        // 校验完成了，开始处理authorizationRequest
        if (authorizationRequest.isApproved()) {
            // 由于是implicit模式，因此进入以下方法
            if (responseTypes.contains("token")) {
                return getImplicitGrantResponse(authorizationRequest);
            }
            if (responseTypes.contains("code")) {
                return new ModelAndView(getAuthorizationCodeResponse(authorizationRequest,
                                                                     (Authentication) principal));
            }
        }

        // Place auth request into the model so that it is stored in the session
        // for approveOrDeny to use. That way we make sure that auth request comes from the session,
        // so any auth request parameters passed to approveOrDeny will be ignored and retrieved from the session.
        model.put("authorizationRequest", authorizationRequest);

        return getUserApprovalPageResponse(model, authorizationRequest, (Authentication) principal);

    }
    catch (RuntimeException e) {
        sessionStatus.setComplete();
        throw e;
    }

}
```



## 6.AuthorizationEndPoint#getImplicitGrantResponse

```java
private ModelAndView getImplicitGrantResponse(AuthorizationRequest authorizationRequest) {
    try {
        // tokenResut中包含了grantType、scope、clientId等基本信息
        TokenRequest tokenRequest = getOAuth2RequestFactory().createTokenRequest(authorizationRequest, "implicit");
        // storedOAuth2Request包含了resourceIds、authorities等涉及token的信息
        OAuth2Request storedOAuth2Request = getOAuth2RequestFactory().createOAuth2Request(authorizationRequest);
        // 将上面两个对象传入geAccessTokenForImplicitGrant方法中，以生成token，这个方法的重要性不言而喻
        OAuth2AccessToken accessToken = getAccessTokenForImplicitGrant(tokenRequest, storedOAuth2Request);
        if (accessToken == null) {
            throw new UnsupportedResponseTypeException("Unsupported response type: token");
        }
        //把token信息追加到重定向地址的url上
        return new ModelAndView(new RedirectView(appendAccessToken(authorizationRequest, accessToken), false, true, false));
    }
    catch (OAuth2Exception e) {
        return new ModelAndView(new RedirectView(getUnsuccessfulRedirect(authorizationRequest, e, true), false, true, false));
    }
}
```



## 7.AuthorizationEndPoint#getAccessTokenForImplicitGrant

```java
private OAuth2AccessToken getAccessTokenForImplicitGrant(TokenRequest tokenRequest,
                                                         OAuth2Request storedOAuth2Request) {
    OAuth2AccessToken accessToken = null;
    // 这个方法调用必须是原子的，否则ImplicitGrantService会有一个竞争条件，其中一个线程在另一个线程有机会赎回令牌请求之前删除它。nnnnn
    synchronized (this.implicitLock) {
        // 这里就要生成accesstoken了！
        accessToken = getTokenGranter().grant("implicit",                                              new ImplicitTokenRequest(tokenRequest, storedOAuth2Request));
    }
    return accessToken;
}
```



## 8.AbstractTokenGranter#grant

```java
public OAuth2AccessToken grant(String grantType, TokenRequest tokenRequest) {
	// type校验
    if (!this.grantType.equals(grantType)) {
        return null;
    }

    // 获取clientId
    String clientId = tokenRequest.getClientId();
    // 加载clientDetails对象
    ClientDetails client = clientDetailsService.loadClientByClientId(clientId);
    // 检查grantType
    validateGrantType(grantType, client);

    if (logger.isDebugEnabled()) {
        logger.debug("Getting access token for: " + clientId);
    }

    // 获取token
    return getAccessToken(client, tokenRequest);

}
```



### AbstractTokenGranter#getAccessToken

getAccessToken这里就比较明显了，使用了tokenService的createAccessToken方法。tokenService默认情况下为DefaultTokenServices

```java
protected OAuth2AccessToken getAccessToken(ClientDetails client, TokenRequest tokenRequest) {
    return tokenServices.createAccessToken(getOAuth2Authentication(client, tokenRequest));
}
```



## 9.DefaultTokenServices#createAccessToken(OAuth2Authentication authentication)

这里需要注意createAccessToken里有一个tokenStore对象，这个东东就是用来存储token的，默认情况下oauth2会使用内存存储token，不过也有很多其他的方法，如jdbc、redis等。这里暂时不介绍token存储的细节。

```java
private TokenStore tokenStore;

@Transactional
public OAuth2AccessToken createAccessToken(OAuth2Authentication authentication) throws AuthenticationException {
	// 检查是否存在旧的token，如果是第一次访问，那么这个existingToken就是空的了
    OAuth2AccessToken existingAccessToken = tokenStore.getAccessToken(authentication);
    OAuth2RefreshToken refreshToken = null;
    if (existingAccessToken != null) {
        // 如果过期了，就要把原来tokenstore的token移除
        if (existingAccessToken.isExpired()) {
            if (existingAccessToken.getRefreshToken() != null) {
                refreshToken = existingAccessToken.getRefreshToken();
                // The token store could remove the refresh token when the
                // access token is removed, but we want to
                // be sure...
                tokenStore.removeRefreshToken(refreshToken);
            }
            tokenStore.removeAccessToken(existingAccessToken);
        }
        // 没有过期的话就返回了
        else {
            // Re-store the access token in case the authentication has changed
            tokenStore.storeAccessToken(existingAccessToken, authentication);
            return existingAccessToken;
        }
    }

    // 创建一个refreshToken
    if (refreshToken == null) {
        refreshToken = createRefreshToken(authentication);
    }
    // 如果refreshToken过期了，也需要新建
    else if (refreshToken instanceof ExpiringOAuth2RefreshToken) {
        ExpiringOAuth2RefreshToken expiring = (ExpiringOAuth2RefreshToken) refreshToken;
        if (System.currentTimeMillis() > expiring.getExpiration().getTime()) {
            refreshToken = createRefreshToken(authentication);
        }
    }

    // 创建accessToken，这里注意有两个参数，是调用了一个重载的方法
    OAuth2AccessToken accessToken = createAccessToken(authentication, refreshToken);
    // 调用tokenStore存储创建的token
    tokenStore.storeAccessToken(accessToken, authentication);
    // 防止refreshToken被修改了，把token的refreshToken重新赋值
    refreshToken = accessToken.getRefreshToken();
    // 存储refreshToken
    if (refreshToken != null) {
        tokenStore.storeRefreshToken(refreshToken, authentication);
    }
    return accessToken;
}
```



## 10.DefaultTokenServices#createAccessToken(OAuth2Authentication authentication, OAuth2RefreshToken refreshToken)

这里其实就是创建token的最终步骤了，可以发现其实token就是一个随机的uuid，同时携带了过期时间、refreshtoken、scope等信息。

```java
private OAuth2AccessToken createAccessToken(OAuth2Authentication authentication, OAuth2RefreshToken refreshToken) {
    DefaultOAuth2AccessToken token = new DefaultOAuth2AccessToken(UUID.randomUUID().toString());
    int validitySeconds = getAccessTokenValiditySeconds(authentication.getOAuth2Request());
    if (validitySeconds > 0) {
        token.setExpiration(new Date(System.currentTimeMillis() + (validitySeconds * 1000L)));
    }
    token.setRefreshToken(refreshToken);
    token.setScope(authentication.getOAuth2Request().getScope());

    return accessTokenEnhancer != null ? accessTokenEnhancer.enhance(token, authentication) : token;
}
```



## 11.小结

到第十步为止，token已经正常生成，程序的调用栈已经走到了尽头，在最后第六步中，token信息将会跟随ModelAndView返回到前端页面，完成地址的跳转。页面的跳转后路径如下：

```xml
https://www.baidu.com/#access_token=979a5982-8ae3-4525-9cc4-f36e4389a199&token_type=bearer&expires_in=86399&scope=demo
```

到此，整个认证的过程就结束了，整个流程的大致调用多达20步左右，用2篇文章的篇幅其实还是有点紧凑了，导致一些细节没有分析，之后如果碰到相关问题的话再做补充~

