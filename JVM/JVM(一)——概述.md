# JVM(一)——概述

## 一、Java从编码到执行

![img](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/javaCode.png)



## 二、常见的JVM实现

可以在cmd中输入java -version查看jvm版本。

### HotSpot

orcale官方

### Jcrokit

BEA，曾经号称世界上最快的JVM，后被oracle收购

### J9

IBM

### Microsoft VM

### Taobao VM

hotspot深度定制版

### LiquidVM

直接针对硬件的vm



## 三、常见概念

### JDK

JDK=JRE+delevopment kit

### JRE

JRE=jvm+core lib



## 四、Class文件

.java文件经过javac编译之后会变成.class文件，class文件是二进制数据流，可以通过sublime，IDEA BinEd等软件或插件以十六进制形式打开查看。class中的数据类型可以分为u1、u2、u4、u8(分别表示1、2、4、8个字节)

### Class文件格式

#### Magic Number

4字节。数据固定，由16进制数**CAFEBABE**组成



#### Minor Version

2字节。java版本的小版本号



#### Major Version

2字节。java版本的大版本号



#### constant_pool_count

2字节。常量池中的常量个数



#### constant_pool

常量池存储的常量内容。是长度为constant_pool_count-1的表。有18个常量类型，主要有以下：

```
CONSTANT_Utf8_info
CONSTANT_Integer_info
CONSTANT_Float_info
CONSTANT_Long_info
CONSTANT_Double_info
CONSTANT_Class_info
CONSTANT_String_info
CONSTANT_Fieldref_info
CONSTANT_Methodref_info
CONSTANT_InterfaceMethodref_info
CONSTANT_NameAndType_info
CONSTANT_MethodHandle_info
CONSTANT_MethodType_info
CONSTANT_InvokeDynamic_info
```



#### access_flags

类访问标记，2个字节，用于做位运算。主要有以下属性：

```
ACC_PUBLIC 0x0001——是否为public类型
ACC_FINAL 0x0010——是否有final修饰
ACC_SUPER 0x0020——标志位，必须为真
ACC_INTERFACE 0x0200——是否为接口
ACC_ENUM 0x0400——是否为抽象类
ACC_SYNTHETIC 0x1000——编译器自动生成
ACC_ANNOTATION 0x2000——是否有注解
ACC_ENUM 0x4000——是否为枚举类
```



#### this_class

2个字节，当前class，指向常量池表中的class引用



#### super_class

指向常量池中父类class引用



#### interface_count

实现的接口个数



#### interfaces

实现的具体接口，存储指向常量池的引用



#### fields_count

字段个数



#### fields

具体的字段存储指向常量池的引用



#### methods_count

方法数量



#### methods

具体的方法



#### attributes_count

属性计数器，attributes_count的值表示当前Class文件attributes表的成员个数。



#### attributes

属性表，attributes表的每个项的值必须是attribute_info结构。