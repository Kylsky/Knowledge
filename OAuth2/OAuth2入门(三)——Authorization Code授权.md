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
E) 认证服务器检查授权码和重定向URI的有效性，通过后**颁发AccessToken



## 3.总结

授权码模式相较于简化模式相对复杂一些，因为多了获取授权码的步骤，不过总体而言理解起来也不是很困难。