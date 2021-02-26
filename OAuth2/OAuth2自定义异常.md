# 1.概述

项目在调用OAuth2默认的RemoteTokenServices进行check_token校验时，发现返回格式如下：

```json
{
    "error": "invalid_token",
    "error_description": "Invalid access token: XXXXXXXXXXXXXXXXXXXXXXX"
}
```

由于前端进行响应处理时需要的返回格式为：

```json
{
	"success":false,
   	"code":401,
    "desc":"invalid_token"
}
```

因此需要对异常进行处理



# 2.源码分析

RemoteTokenServices#loadAuthentication

```java
@Override
public OAuth2Authentication loadAuthentication(String accessToken) throws AuthenticationException, InvalidTokenException {

    MultiValueMap<String, String> formData = new LinkedMultiValueMap<String, String>();
    formData.add(tokenName, accessToken);
    HttpHeaders headers = new HttpHeaders();
    headers.set("Authorization", getAuthorizationHeader(clientId, clientSecret));
    Map<String, Object> map = postForMap(checkTokenEndpointUrl, formData, headers);

    // 一般出现错误之后，map中存在error字段，因此会进入这个条件，并抛出异常
    if (map.containsKey("error")) {
        if (logger.isDebugEnabled()) {
            logger.debug("check_token returned error: " + map.get("error"));
        }
        throw new InvalidTokenException(accessToken);
    }

    if (!Boolean.TRUE.equals(map.get("active"))) {
        logger.debug("check_token returned active attribute: " + map.get("active"));
        throw new InvalidTokenException(accessToken);
    }

    return tokenConverter.extractAuthentication(map);
}
```



# 3.解决思路

考虑到loadAuthentication抛出的异常会被OAuth2AuthenticationProcessingFilter捕获，因此不考虑在重写的checkToken接口中改造loadAuthentication方法接收的map参数并尝试在自定义的tokenConverter#extractAuthentication中抛出自定义的异常（实践证明确实不行，因为自定义的异常无法被ControllerAdvice接收）。

另外考虑到重写一个新的CustomRemoteTokenServices，不过实践也以失败告终。

最后不得不进行面向百度编程了，这里必须吐槽csdn的盲目抄袭和对文章质量完全不负责的开发者~~！！大致的思路就是继承OAuth2Exception，并创建异常转换器，以及创建自定义的异常序列化，下面是具体的实现。



# 4.解决方案

## 4.1 CustomOauthException

自定义一个异常，继承OAuth2Exception。

```json
import com.fasterxml.jackson.databind.annotation.JsonSerialize;
import org.springframework.security.oauth2.common.exceptions.OAuth2Exception;

//这里配置了序列化，不要漏掉
@JsonSerialize(using = CustomOauthExceptionSerializer.class)
public class CustomOauthException extends OAuth2Exception {
    // 自定义的两个参数，可以根据个人需求设置
    private String oAuth2ErrorCode;
    private int httpErrorCode;

    public CustomOauthException(String msg, String oAuth2ErrorCode, int httpErrorCode) {
        super(msg);
        this.oAuth2ErrorCode = oAuth2ErrorCode;
        this.httpErrorCode = httpErrorCode;
    }

    public String getoAuth2ErrorCode() {
        return oAuth2ErrorCode;
    }

    @Override
    public int getHttpErrorCode() {
        return httpErrorCode;
    }
}
```



## 4.2 CustomOauthExceptionSerializer

这个就是需要配置在CustomOauthException上的序列化工具。

```java
import com.fasterxml.jackson.core.JsonGenerator;
import com.fasterxml.jackson.databind.SerializerProvider;
import com.fasterxml.jackson.databind.ser.std.StdSerializer;
import org.apache.ibatis.executor.ResultExtractor;

import java.io.IOException;

public class CustomOauthExceptionSerializer extends StdSerializer<CustomOauthException> {

    public CustomOauthExceptionSerializer() {
        super(CustomOauthException.class);
    }

    @Override
    public void serialize(CustomOauthException value, JsonGenerator gen, SerializerProvider provider) throws IOException {
        gen.writeStartObject();
        // 在这里自定义需要返回的字段
        gen.writeObjectField("success",false);
        gen.writeObjectField("code", "401");
        gen.writeObjectField("desc","invalid_token");
        gen.writeEndObject();
    }
}
```



## 4.3 CustomExceptionTranslator

定义一个异常转换器，有了这个转换器后就能将抛出的异常转换为我们自定义的异常了。

```java
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.security.oauth2.common.exceptions.OAuth2Exception;
import org.springframework.security.oauth2.provider.error.DefaultWebResponseExceptionTranslator;

public class CustomExceptionTranslator extends DefaultWebResponseExceptionTranslator {

    @Override
    public ResponseEntity<OAuth2Exception> translate(Exception e) throws Exception {
        ResponseEntity<OAuth2Exception> translate = super.translate(e);
        OAuth2Exception body = translate.getBody();
        CustomOauthException customOauthException = new CustomOauthException(body.getMessage(),body.getOAuth2ErrorCode(),
                                                                             body.getHttpErrorCode());
        ResponseEntity<OAuth2Exception> response = new ResponseEntity<>(customOauthException, translate.getHeaders(),
                                                                        HttpStatus.OK);
        return response;
    }
}
```

别忘了把转换器配置到ResourceServerConfigureAdapter的实现类中去：

```java
@Override
public void configure(ResourceServerSecurityConfigurer resources) throws Exception {
    OAuth2AuthenticationEntryPoint authenticationEntryPoint = new OAuth2AuthenticationEntryPoint();
    //在这里配置转换器
    authenticationEntryPoint.setExceptionTranslator(new CustomExceptionTranslator());
    resources.authenticationEntryPoint(authenticationEntryPoint);
}
```



# 5.注意

需要注意的是，由于这里对异常的序列化是基于Content-Type=application/json的，因此在浏览器上访问仍然会出现非预期的错误提示。