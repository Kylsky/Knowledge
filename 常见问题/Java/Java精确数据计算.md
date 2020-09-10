# Java精确数据计算

今天在看公司项目的时候看到了Java对数据的精确计算的部分代码，比较感兴趣，于是网上查了一些资料，来做一下记录与整理

## double的问题

在java中，double是无法完成精确的数据计算的，可以看如下代码：

```java
public class DoubleTest {
    public static void main(String[] args) {
        double number1 = 1.01 + 2.13;
        double number2 = 4.12 - 2.01;
        double number3 = 151.3/100;
        System.out.println(number1);
        System.out.println(number2);
        System.out.println(number3);
    }
}
```

打印的结果值为：

```
3.1399999999999997
2.1100000000000003
1.5130000000000001
```

虽然试了一下其他的加法值是有返回正确的值得，但是只要存在某一种情况得到了类似上述的结果，那么在遇到需要精确值结果的情况下double便不能直接被使用。

## BigDecimal

bigdecimal被用来解决double的计算数值不精确情况，这里先通过代码来作比较：

```
public class BigDecimalTest {
    public static void main(String[] args) {
        BigDecimal number1 = new BigDecimal(1.01);
        BigDecimal number2 = new BigDecimal(2.13);
        BigDecimal number3 = new BigDecimal(4.12);
        BigDecimal number4 = new BigDecimal(2.01);
        BigDecimal number5 = new BigDecimal(151.3);
        BigDecimal number6 = new BigDecimal(100);

        //加法
        System.out.println(number1.add(number2).doubleValue());
        //减法
        System.out.println(number3.subtract(number4).doubleValue());
        //除法
        System.out.println(number5.divide(number6).doubleValue());
    }
}
```

来看看结果吧

```
3.1399999999999997
2.1100000000000003
1.5130000000000001
```

？？？怎么翻车了？？？不是说好的精确值吗？？？痛定思痛，我决定再仔细看看，发现BigDecimal还有一个相似的构造函数，但是传入的参数是String类型的，那么再来试试：

```java
public class BigDecimalTest {
    public static void main(String[] args) {
        BigDecimal number1 = new BigDecimal("1.01");
        BigDecimal number2 = new BigDecimal("2.13");
        BigDecimal number3 = new BigDecimal("4.12");
        BigDecimal number4 = new BigDecimal("2.01");
        BigDecimal number5 = new BigDecimal("151.3");
        BigDecimal number6 = new BigDecimal("100");

        //加法
        System.out.println(number1.add(number2).doubleValue());
        //减法
        System.out.println(number3.subtract(number4).doubleValue());
        //除法
        System.out.println(number5.divide(number6).doubleValue());
    }
}
```

看看结果：

```
3.14
2.11
1.513
```

奥力给！！！成功了哈哈，另外发现使用BigDecimal的valueOf方法也可以达到相同的效果。不过光是这样的简单操作还是不够的，有时候对一个数据的计算会要求精确到几位小数，并且对末尾的舍去机制也有不同的要求(如四舍五入，或直接截断等)，这时就要使用setScale方法来对数据做截断,在了解setScale方法前，先来看看java中默认的几种截取模式：

### ROUND_CEILING

将数据往正半轴方向舍入

### ROUND_FLOOR

将数据往负半轴方向舍入

### ROUND_DOWN

将数据往靠近0的方向舍入

### ROUND_UP

将数据往远离0的方向舍入

### ROUND_UNNECESSARY

不使用舍入模式，如果使用此模式得到了不精确的计算结果，会抛出ArithmeticException异常

### ROUND_HALF_UP

数学意义上的四舍五入，向距离近的一边舍入，若距离相等，则向远离0的方向舍入

### ROUND_HALF_DOWN

向距离近的一边舍入，若距离相等，则向靠近0的方向舍入

### ROUND_HALF_EVEN

向距离最近的一边舍入，如果两边距离相等且小数点后保留的位数是奇数，就使用**ROUND_HALF_UP**模式；如果两边距离相等且小数点后保留的位数是偶数，就使用**ROUND_HALF_DOWN**



那么，举个例子，当需要对一个数字保留两位小数并进行四舍五入时，可以做如下操作：

```java
public class BigDecimalTest {
    public static void main(String[] args) {
        BigDecimal number1 = new BigDecimal("1.01");
        BigDecimal number2 = new BigDecimal("2.13");
        System.out.println(number1.multiply(number2).setScale(2,BigDecimal.ROUND_HALF_UP));
    }
}
```

结果如下：

```
2.15
```

其他模式就不一一列举了。本篇对java精确数值的计算就简单介绍到这里了，以后若有碰到其他情况再做补充。