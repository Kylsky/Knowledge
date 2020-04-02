# PostMan+OAuth2

今天在写公司接口的时候要进行测试，可是接口测试需要通过权限校验，当前项目使用的是OAuth2，因为打算用PostMan进行相关测试，因此去百度了一下相关的操作，以下是我的相关操作

**通过正常流程进行网页登录，随后找到Http Request Header中的Cookie**

![img](http://kylescloud.top/site/pic/postman+oauth.jpg)

![img](http://kylescloud.top/site/pic/postman+oauth2.jpg)

**将3的内容对应到postman的Header中，如下：**

![img](http://kylescloud.top/site/pic/postman+oauth3.jpg)

此时即可执行相关的请求了，其他的请求参数与请求类型按照接口要求进行操作即可。