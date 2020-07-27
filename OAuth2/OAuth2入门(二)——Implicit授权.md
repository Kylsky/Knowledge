# OAuth2入门(二)——Implicit授权模式

## 1.前言

前面的文章讲到，oauth支持四种授权模式：

- 简化模式（implicit）

- 授权码模式（authorization code）
- 密码模式（resource owner password credentials）
- 客户端模式（client credentials）
- 扩展模式（Extension）

这篇先来讲讲最简单的简化模式



## 2.流程

用户使用oauth简化模式进行第三方登录的流程主要如下：

![image-20200710102459231](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20200710102459231.png)

**ResourceOwner**：资源所有者，即为用户
**User-Agent**：浏览器
**AuthorizationServer**：认证服务器，可以理解为用户资源托管方，比如企业微信服务端
**Client**：第三方服务



1.上图中，用户在访问第三方应用(User-Agent)时需要通过oauth系统进行登录，因此，当前应用(假设之前从未登录)发现本地token不存在，于是根据oauth分发的**client_id**和**redirect_uri**，通过指定接口尝试获取token,接口如下:

```
//host指向oauth认证系统所在地址
http://${host}/oauth/authorize?response_type=token&client_id=${client_id}&redirect_uri=${redirect_uri}
```



2.若oauth服务器发现当前用户没有登录，则会将请求转发至登录接口，等待用户登陆完成后，oauth继续完成/oauth/authorize相应操作，并将连接重定向至redirect_uri，同时带上access_token等参数，至此，用户完成登录操作



## 3.代码实现

由于/oauth/authorize接口是由oauth2自身实现的，因此贴一下源码，来看看实现细节

**AuthorizationEndpoint.authorize#116**

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
                    //一般而言，会默认进入到这个方法中,下面会展示
					return getImplicitGrantResponse(authorizationRequest);
				}
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



**AuthorizarionEndPoint#252**

```java
private ModelAndView getImplicitGrantResponse(AuthorizationRequest authorizationRequest) {
		try {
            //创建一个获取token的请求和一个oauth2请求，用于获取access_token
			TokenRequest tokenRequest = getOAuth2RequestFactory().createTokenRequest(authorizationRequest, "implicit");
			OAuth2Request storedOAuth2Request = getOAuth2RequestFactory().createOAuth2Request(authorizationRequest);
			OAuth2AccessToken accessToken = getAccessTokenForImplicitGrant(tokenRequest, storedOAuth2Request);
			if (accessToken == null) {
				throw new UnsupportedResponseTypeException("Unsupported response type: token");
			}
            //若token创建成功，则重定向并返回token
			return new ModelAndView(new RedirectView(appendAccessToken(authorizationRequest, accessToken), false, true,
					false));
		}
		catch (OAuth2Exception e) {
			return new ModelAndView(new RedirectView(getUnsuccessfulRedirect(authorizationRequest, e, true), false,
					true, false));
		}
	}
```

