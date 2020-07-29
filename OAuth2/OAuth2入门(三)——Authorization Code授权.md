# OAuth2入门(三)——Authorization Code授权模式

## 1.前言

前面的文章讲到，oauth支持四种授权模式：

- 简化模式（implicit）

- 授权码模式（authorization code）
- 密码模式（resource owner password credentials）
- 客户端模式（client credentials）
- 扩展模式（Extension）

这篇来讲讲authorization code授权模式



## 2.流程

用户使用oauth简化模式进行第三方登录的流程主要如下：

![img](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/20190410151114440.png)

**ResourceOwner**：资源所有者，即为用户
**User-Agent**：浏览器
**AuthorizationServer**：认证服务器，可以理解为用户资源托管方，比如企业微信服务端
**Client**：第三方服务



调用流程为：
A) 第三方服务通过构造OAuth2链接（传参为client_id以及redirect_uri），将用户引导到认证服务器的授权页
B) 用户登录(假设从前未登录)选择是否同意授权
C) 若用户同意授权，则认证服务器将用户重向到第一步指定的**重定向URI**，同时附上一个**授权码**。
D) 第三方服务收到授权码，带上授权码来源的重定向URI，向认证服务器申请凭证。
E) 认证服务器检查授权码和重定向URI的有效性，通过后颁发AccessToken



## 3.代码实现

授权码模式通常需要进行两次请求从而获取token，第一次请求为/oauth/authorize，以获取授权码，之前在implicit模式介绍过这个接口的源码，主要的区别在于type不同。授权码模式下获取code的url示例如下：

```
http://${host}/oauth/authorize?response_type=code&client_id=${client_id}&redirect_uri=${redirect_uri}
```

### /oauth/authorize

```java
@RequestMapping(value = "/oauth/authorize")
	public ModelAndView authorize(Map<String, Object> model, @RequestParam Map<String, String> parameters,
			SessionStatus sessionStatus, Principal principal) {
		//将提交过来的请求封装成AuthorizationRequest对象
		AuthorizationRequest authorizationRequest = getOAuth2RequestFactory().createAuthorizationRequest(parameters);
		//获取响应类型，implicit模式的响应类型是token
		Set<String> responseTypes = authorizationRequest.getResponseTypes();
		//排除非token、非code的响应类型
		if (!responseTypes.contains("token") && !responseTypes.contains("code")) {
			throw new UnsupportedResponseTypeException("Unsupported response types: " + responseTypes);
		}
		//client_id参数校验
		if (authorizationRequest.getClientId() == null) {
			throw new InvalidClientException("A client id must be provided");
		}
		//判断当前访问改地址的用户是否已经授权
		try {

			if (!(principal instanceof Authentication) || !((Authentication) principal).isAuthenticated()) {
				throw new InsufficientAuthenticationException(
						"User must be authenticated with Spring Security before authorization can be completed.");
			}
			//若用户已经授权，则获取client_id
			ClientDetails client = getClientDetailsService().loadClientByClientId(authorizationRequest.getClientId());

			//获取redirect_uri
			String redirectUriParameter = authorizationRequest.getRequestParameters().get(OAuth2Utils.REDIRECT_URI);
            //检查一下格式
			String resolvedRedirect = redirectResolver.resolveRedirect(redirectUriParameter, client);
			if (!StringUtils.hasText(resolvedRedirect)) {
				throw new RedirectMismatchException(
						"A redirectUri must be either supplied or preconfigured in the ClientDetails");
			}
			authorizationRequest.setRedirectUri(resolvedRedirect);

			//判断请求的scope是否在规定的范围内
			oauth2RequestValidator.validateScope(authorizationRequest, client);

			// 不同的系统对aprroved的实现不一样
			authorizationRequest = userApprovalHandler.checkForPreApproval(authorizationRequest,
					(Authentication) principal);
			boolean approved = userApprovalHandler.isApproved(authorizationRequest, (Authentication) principal);
			authorizationRequest.setApproved(approved);
			if (authorizationRequest.isApproved()) {
				if (responseTypes.contains("token")) {
					return getImplicitGrantResponse(authorizationRequest);
				}
                //type为code，进入以下方法，若用户输入正确的请求地址和参数，则会将code通过redirect_uri传递给用户
				if (responseTypes.contains("code")) {
					return new ModelAndView(getAuthorizationCodeResponse(authorizationRequest,
							(Authentication) principal));
				}
			}
			
            //将auth请求存入model，这样就能让approveOrDeny方法通过session获取到这个信息
			model.put("authorizationRequest", authorizationRequest);

			return getUserApprovalPageResponse(model, authorizationRequest, (Authentication) principal);

		}
		catch (RuntimeException e) {
			sessionStatus.setComplete();
			throw e;
		}
}
```



### /oauth/token

用户获取到授权码之后，就可以携带code并通过/oauth/token接口来得到token，授权码模式下获取token的请求url示例如下：

```
http://${host}/oauth/token?client_id=${client_id}&client_secret=${client_cecret}&grant_type=authorization_code&code=${code}&redirect_uri=${redirect_uri}
```

下面看一下源码：

```java
//接收GET请求，其实最终处理的还是会用处理POST的方法	
@RequestMapping(value = "/oauth/token", method=RequestMethod.GET)
	public ResponseEntity<OAuth2AccessToken> getAccessToken(Principal principal, 	@RequestParam
	Map<String, String> parameters) throws HttpRequestMethodNotSupportedException {
		//若不支持GET方法，则抛出异常
        if (!allowedRequestMethods.contains(HttpMethod.GET)) {
			throw new HttpRequestMethodNotSupportedException("GET");
		}
        //调用postAccessToken方法，在下面
		return postAccessToken(principal, parameters);
	}
	
//处理POST方法
	@RequestMapping(value = "/oauth/token", method=RequestMethod.POST)
	public ResponseEntity<OAuth2AccessToken> postAccessToken(Principal principal, @RequestParam
	Map<String, String> parameters) throws HttpRequestMethodNotSupportedException {
		//参数校验
		if (!(principal instanceof Authentication)) {
			throw new InsufficientAuthenticationException(
					"There is no client authentication. Try adding an appropriate authentication filter.");
		}
		//获取clientId
		String clientId = getClientId(principal);
        //通过clientId获取一个ClientDetails对象authenticatedClient
		ClientDetails authenticatedClient = getClientDetailsService().loadClientByClientId(clientId);
		//根据authenticatedClient和请求参数创建一个获取token的请求
		TokenRequest tokenRequest = getOAuth2RequestFactory().createTokenRequest(parameters, authenticatedClient);
		//这里有对clientId的参数校验以及双重检查，用来保证需要请求的token对应准确的clientId
		if (clientId != null && !clientId.equals("")) {
			if (!clientId.equals(tokenRequest.getClientId())) {
				
				throw new InvalidClientException("Given client ID does not match authenticated client");
			}
		}
        //校验scope
		if (authenticatedClient != null) {
			oAuth2RequestValidator.validateScope(tokenRequest, authenticatedClient);
		}
        //校验grant type
		if (!StringUtils.hasText(tokenRequest.getGrantType())) {
			throw new InvalidRequestException("Missing grant type");
		}
        //若是implicit，则会抛出异常，表示不支持
		if (tokenRequest.getGrantType().equals("implicit")) {
			throw new InvalidGrantException("Implicit grant type not supported from token endpoint");
		}
        //如果是授权码模式获取token的请求，则清空scope
		if (isAuthCodeRequest(parameters)) {
			// The scope was requested or determined during the authorization step
			if (!tokenRequest.getScope().isEmpty()) {
				logger.debug("Clearing scope of incoming token request");
				tokenRequest.setScope(Collections.<String> emptySet());
			}
		}
		//是refreshtoken请求
		if (isRefreshTokenRequest(parameters)) {
            //一个refresh token请求有独有的默认scope，因此需要忽略所有被工厂添加的scope
			tokenRequest.setScope(OAuth2Utils.parseParameterList(parameters.get(OAuth2Utils.SCOPE)));
		}
        //获取token
		OAuth2AccessToken token = getTokenGranter().grant(tokenRequest.getGrantType(), tokenRequest);
		if (token == null) {
			throw new UnsupportedGrantTypeException("Unsupported grant type: " + tokenRequest.getGrantType());
		}
		return getResponse(token);
}
```

返回格式如下：

```
{

"access_token":"ef1e99f5-6da6-4bae-8fc8-f63520ce0fdf", //token

"token_type":"bearer",

"refresh_token":"fd20713f-8aac-47da-a78c-c1cf56ed5ea1",

"expires_in":86399,

"scope":"demo"

}
```



## 4.总结

授权码模式相较于简化模式相对复杂一些，因为多了获取授权码的步骤，不过总体而言理解起来也不是很困难。