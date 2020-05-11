# String(一)——构造方法

String可以说是开发中最常用的一个类，由于这个类的代码实在有点多，光构造方法就十几个，因此分多次说明把，第一篇先从构造方法开始讲



### 1.String()

```
public String() {
    this.value = "".value;
}
```

这是一个很有趣的方法，如果你在一个外部的类尝试使用“”.value，会发现根本没这API，因为value属性在String类中是一个final关键字修饰的字符数组，因此只能在内部调用，因此，该构造函数其实就是创建了一个String类型的对象，同时其内部数据为一个空的字符数组。



### 2.String(String original)

```java
public String(String original) {
    this.value = original.value;
    this.hash = original.hash;
}
```

传入字符串，并创建一个新的对象，value、hash与参数一致。也就是说，这个新创建的字符串是参数字符串的副本，意思是，两个字符串的value值属于同一个字符串常量，因此hashCode相同。



### 3.String(char value[])

```java
public String(char value[]) {
    this.value = Arrays.copyOf(value, value.length);
}
```

拷贝一个字符数组



### 4.String(char value[], int offset, int count)

**value[]**表示一个字符数组，**offset**表示偏移量，**int**为从offset开始的字符数

```java
public String(char value[], int offset, int count) {
	//偏移量异常
    if (offset < 0) {
        throw new StringIndexOutOfBoundsException(offset);
    }
    if (count <= 0) {
        //count<0，抛异常
        if (count < 0) {
            throw new StringIndexOutOfBoundsException(count);
        }
        //count=0，且offset范围正常，返回空字符数组
        if (offset <= value.length) {
            this.value = "".value;
            return;
        }
    }
    //偏移量异常
    if (offset > value.length - count) {
        throw new StringIndexOutOfBoundsException(offset + count);
    }
    //复制字符数组
    this.value = Arrays.copyOfRange(value, offset, offset+count);
}
```



### 5.String(int[] codePoints, int offset, int count)

```java
public String(int[] codePoints, int offset, int count) {
    if (offset < 0) {
        throw new StringIndexOutOfBoundsException(offset);
    }
    if (count <= 0) {
        if (count < 0) {
            throw new StringIndexOutOfBoundsException(count);
        }
        if (offset <= codePoints.length) {
            this.value = "".value;
            return;
        }
    }
    if (offset > codePoints.length - count) {
        throw new StringIndexOutOfBoundsException(offset + count);
    }
	//上面的内容就不赘述了
    //end表示为最后一个数的偏移量
    final int end = offset + count;
    int n = count;
    //计算数组精确长度
    for (int i = offset; i < end; i++) {
        int c = codePoints[i];
        //c属于一个BMP，则跳过进行下一个
        if (Character.isBmpCodePoint(c))
            continue;
        //c不为BMP且属于unicode
        else if (Character.isValidCodePoint(c))
            n++;
        //c超过unicode范围，抛异常
        else throw new IllegalArgumentException(Integer.toString(c));
    }

    // 填充到新的字节数组中
    final char[] v = new char[n];

    for (int i = offset, j = 0; i < end; i++, j++) {
        int c = codePoints[i];
        //BMP填充
        if (Character.isBmpCodePoint(c))
            v[j] = (char)c;
        //非BMP填充
        else
            Character.toSurrogates(c, v, j++);
    }

    this.value = v;
}
```

由于unicode的合理取值范围是0x0000-0x10ffff，同时0x0000-0xffff这一段被称为BMP(Basic Multilingual Plane，基本多文种平面)，关于BMP可以参考这篇文章：

```
https://www.fontke.com/article/1071/
```

另外，由于char的大小为2字节，所以能表示的最大值为0xffff，也就是说char字符类型只能表示BMP。因此，如果传入的int字节数组中存在容量大于oxffff的值，则在创建字节数组时需要额外分配一个字节(只分配一个是因为传入的int值只允许在unicode范围内，即3个字节)，这些对应的int数据会使用n++做记录。

上面的非BMP填充比较复杂，拎出来单独看一看：

Character#5149

```java
static void toSurrogates(int codePoint, char[] dst, int index) {
        // We write elements "backwards" to guarantee all-or-nothing
        dst[index+1] = lowSurrogate(codePoint);
        dst[index] = highSurrogate(codePoint);
}
```

Surrogate这个概念，不是来自Java语言，而是来自Unicode编码方式之一:UTF-16。 简而言之，Java语言內部的字符信息是使用UTF-16编码。因为char这个类型是16-bit的，它可以有65536种取值，即65536个编号，每个编号可以代表1种字符。但是Unicode 包含的字符已经远远超过65536个。那编号大于65536的，还要用16-bit编码，该怎么办？ 于是Unicode标准制定组想出的办法就是，从这65536个编号里，拿出2048个，规定它们是「Surrogates」， 让它们两个为一组，来代表编号大于65536的那些字符。更具体地，编号从U+D800至U+DBFF的规定为「High Surrogates」，共1024个。编号为 U+DC00至U+DFFF的规定为「Low Surrogates」，也是1024个，它们两两组合出现，就又可以多表示1048576种字符。

依次处理高位和低位的数据：

**lowSurrogate(codePoint)**

```java
public static char lowSurrogate(int codePoint) {
        return (char) ((codePoint & 0x3ff) + MIN_LOW_SURROGATE);
}
```

**highSurrogate(codePoint)**

```java
public static char highSurrogate(int codePoint) {
        return (char) ((codePoint >>> 10)
            + (MIN_HIGH_SURROGATE - (MIN_SUPPLEMENTARY_CODE_POINT >>> 10)));
}
```



### 6.String(byte bytes[], int offset, int length, String charsetName)

```java
public String(byte bytes[], int offset, int length, String charsetName)
        throws UnsupportedEncodingException {
    //参数检查
    if (charsetName == null)
        throw new NullPointerException("charsetName");
    //这里也是参数检查
    checkBounds(bytes, offset, length);
    //对字节数组进行指定的字符集解码
    this.value = StringCoding.decode(charsetName, bytes, offset, length);
}
```



### 7.String(byte bytes[], int offset, int length, Charset charset)

```java
public String(byte bytes[], int offset, int length, Charset charset) {
    if (charset == null)
        throw new NullPointerException("charset");
    checkBounds(bytes, offset, length);
    this.value =  StringCoding.decode(charset, bytes, offset, length);
}
```

功能和5号那个相同，但是传入的参数为Charset



### 8.String(byte bytes[], String charsetName)

```java
public String(byte bytes[], String charsetName)
        throws UnsupportedEncodingException {
    this(bytes, 0, bytes.length, charsetName);
}
```

很简单，调用了6号构造方法



### 9.String(byte bytes[], Charset charset)

```java
public String(byte bytes[], Charset charset) {
    this(bytes, 0, bytes.length, charset);
}
```

很简单，调用了7号构造方法



### 10.String(byte bytes[], int offset, int length)

```java
public String(byte bytes[], int offset, int length) {
    checkBounds(bytes, offset, length);
    this.value = StringCoding.decode(bytes, offset, length);
}
```

使用java默认的字符集解码



### 11.String(byte bytes[])

```java
public String(byte bytes[]) {
    this(bytes, 0, bytes.length);
}
```

调了10号构造方法



### 12.String(StringBuffer buffer)

```java
public String(StringBuffer buffer) {
    synchronized(buffer) {
        this.value = Arrays.copyOf(buffer.getValue(), buffer.length());
    }
}
```

拷贝一个Stringbuffer中的值，这里比较有意思的是，数据复制时加上了一把锁。众所周知，Stringbuffer是线程安全的，也就是说，当一个线程需要操作一个Stringbuffer对象，其需要获取到该对象的锁，在这里也是一个意思。



### 13.String(StringBuilder builder)

```java
public String(StringBuilder builder) {
    this.value = Arrays.copyOf(builder.getValue(), builder.length());
}
```

拷贝一个Stringbuilder对象中的值。



### 14.String(char[] value, boolean share)

```java
String(char[] value, boolean share) {
    // assert share : "unshared not supported";
    this.value = value;
}
```

String唯一一个包访问权限的构造方法，其实实现方式和3号没啥差别，官方注释写的是共享值数组以提高速度，暂时还没找到使用这个构造方法的地方，以后看到在讨论把