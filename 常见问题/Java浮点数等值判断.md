# Java浮点数等值判断

阅读阿里Java开发手册时注意到一项：**浮点数之间的等值判断，基本数据类型不能用==来比较，包装数据类型不能用equals 来判断。**

由于浮点数采用“尾数+阶码”的编码方式，类似于科学计数法的“有效数字+指数”的表示方式，因此二进制数无法精确表示大部分十进制小数，如：

```java
float a = 1.0f - 0.9f;
float b = 0.9f - 0.8f;

if(a==b){
    //预期进入此代码块
    //但事实上a==b的结果为false
}

Float c = 1.0f - 0.9f;
Float d = 0.9f - 0.8f;

if(c.equals(d)){
    //预期进入此代码块
    //但事实上a==b的结果为false
}
```

正确的解决方法应该如下：

**1.指定一个误差范围，两个浮点数的差值再次范围之内，则认为是相等的**

```
float a = 1.0f - 0.9f;
float b = 0.9f - 0.8f;

float diff = 1e-6f;

if(Math.abs(a-b)<diff){
     System.out.println("true");
}
```

**2.使用BigDecimal来定义值，再进行浮点数的运算操作**

```
BigDecimal a = new BigDecimal("1.0");
BigDecimal b = new BigDecimal("0.9");
BigDecimal c = new BigDecimal("0.8");

BigDecimal x = a.substract(b);
BigDecimal y = b.substract(c);

if(x.equals(y)){
    System.out.println("true");
}
```

