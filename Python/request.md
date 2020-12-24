# requests

requests中主要包含了get、post、put、options、head、patch、delete几种方法

```
res = requests.get("htpp://www.baidu.com")
```

res代表了一个Response对象，主要包含了以下属性

```
#状态码
status_code
#响应的字符串文本
text
#编码，从head-charset中获取，若不存在，则默认为ISO8859-1
encoding
#备选编码，根据响应的文本进行分析设置的编码
apparent_encoding
#响应的二进制文本
content
```



## 异常

使用Response.raise_for_status()

![image-20201218091108234](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20201218091108234.png)



## 方法

以下为requests的一些方法，其实观察源码不难发现，几乎所有的请求都调用了request方法，\*\*kwargs表示支持输入多个参数，来看一下 \*\*kwargs包含哪些参数

// TODO kwargs

### request(method, url, **kwargs)



### get(url, params=None, **kwargs)

用于发送get请求，参数需要url和params，params默认为空，其他参数可以不传

### options(url, **kwargs)



### head(url, **kwargs)



### post(url, data=None, json=None, **kwargs)



### put(url, data=None, **kwargs)



### patch(url, data=None, **kwargs)



### delete(url, **kwargs)