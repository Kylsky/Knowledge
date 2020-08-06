### 1.js解析json字符串

假设json字符串格式如下：

```
{
	"data":"haha",
	"msg":"emmmm"
}
```

则可以通过如下代码获取json字符串中的内容

```
var var2 = eval('(' + response + ')');
var data = eval.data;
var msg = eval.msg;
```



### 2.js字符串转义

可以使用原生js自带的encodeURI()或者encodeURIComponent()对特殊字符进行转义，区别在于，encodeURI只会对特定的一些字符转义，但是如#之后的内容不会做处理，因此，如果需要将#作为密码传输，则必须使用后者。