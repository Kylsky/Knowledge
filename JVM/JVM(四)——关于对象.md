# JVM(四)——关于对象

## 一、查看虚拟机配置

java -XX:+PrintCommandLineFlags -version



## 二、对象的创建过程

### 1.class loading

### 2.class linking

### 3.class initializing

### 4.申请对象内存

### 5.成员变量赋默认值

### 6.调用构造方法<init>

​	①成员变量顺序赋初始值

​	②执行构造方法语句



## 三、对象的内存布局

### 普通对象

#### 1.对象头

markword，8字节

#### 2.ClassPointer指针：

指向对象所在类的class对象，若配置了-XX:+UseCompressedClassPointers参数，则为4字节，不使用则为8字节

#### 3.实例数据

即成员变量。成员变量为基本类型时，大小为指定数据类型大小。成员变量为引用数据类型时，若开启参数-XX:+UseCompressedOops则为4字节，不开启为8字节

#### 4.padding对齐

8的倍数，cpu缓存读取数据时通常取64字节，padding对齐可以提高读取对象数据的效率



### 数组对象

#### 1.对象头

#### 2.ClassPointer指针

#### 3.数组长度

比普通对象多出的一项，记录了数组的长度，4字节

#### 4.数组数据

#### 5.padding对齐



## 四、对象头包含的信息

![img](http://www.kylescloud.top/site/pic/markword.jpg)



## 五、对象定位

T t = new T()，对象定位即将t引用定位到T的class下

### 1.句柄池

### 2.直接指针



## 六、对象的分配

这里只做简单的图示，之后会有具体的介绍

![img](http://kylescloud.top/site/pic/gc.jpg)