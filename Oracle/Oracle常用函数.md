# Oracle常用函数

## 1.decode()

**功能：**

将字段按照条件判断做特定值返回，mysql同样使用该函数。

**形如：**

```
decode(value,if1,then1,if2,then2,if3,then3,...,else)
```

**意为：**

```
IF 条件=值1 THEN
　　　　RETURN(value 1)
ELSIF 条件=值2 THEN
　　　　RETURN(value 2)
　　　　......
ELSIF 条件=值n THEN
　　　　RETURN(value 3)
ELSE
　　　　RETURN(default)
END IF
```

**使用场景：**

例如数据库中以1、2记录性别，分别表示男和女，但是前端显示时需要显示中文，此时可以使用decode，如下

```
decode(SEX,'1','男','2','女','')SEX
```

上述代码将数据库中的SEX字段值编码根据条件判断返回“男”或“女”，再将SEX设为返回值



## 2.to_char()

**功能：**

将字符串按照指定格式返回

**形如：**

```
to_char(TIME,'yyyy-MM-dd') TIME
```

**意为：**

将数据进行格式化返回，如上述代码中2012/05/12 9:42:09将被格式化为2012-05-12

**使用场景：**

格式化时间字符串



## 3.