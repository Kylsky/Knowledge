# JVM(三)——硬件层并发优化基础

## 回顾

回顾一下JVM对Class的加载和初始化流程

![img](http://www.kylescloud.top/site/pic/classloading.jpg)

### loading

将一个class文件装载到内存

### linking

**1.verification**

校验内存中的class文件格式

**2.preparaton**

为class的静态变量赋默认值，如static int i=8，则为i赋默认值为0

**3.resolution**

将符号引用转换为内存地址

### initializing

调用静态代码块，给静态变量赋值为初始值



### Resolution

resolution能将类、方法、属性等符号引用解析为直接引用，常量池中的各种符号引用解析为指针、偏移量等内存地址的直接引用。比方开发者创建了一个Object类的属性，这个类属性保存在常量池中java.lang.Object这个字符串(符号引用)，java.lang.object又指向了Object这个class具体所在的内存空间，resolution做的就是将java.lang.Object这个符号引用解析成直接引用。



### 指令重排问题

回顾一下双重校验的单例：

```java
public class Test {
    private static volatile Test test;
    private Test(){}

    public Test getInstance(){
        if (test==null){
            synchronized (Test.class){
                if (test==null){
                    test = new Test();
                }
            }
        }
        return test;
    }
}
```

这个问题涉及了volatile、class加载过程以及指令重排序的问题，因此适合当作典型案例，这里以问答形式做解释：

**为什么双重校验需要加上volatile？**

因为volatile能够防止指令的重排序

**什么是指令的重排序？**

拿Object t = new Object()举例，它在jvm中解析后的指令如下：

```
0 new #2 <java/lang/Object>

3 dup

4 invokespecial #1 <java/lang/Object. <init>>

7 astore_1

8 return
```

第1行意为为对象分配内存空间，第3行为调用构造方法，第4行是将t指向对象的内存空间，但是，一般情况下，CPU和编译器为了提升程序执行的效率，会按照一定的规则允许进行指令优化(重排序)，在某些情况下，这种优化会带来一些执行的逻辑问题，即第4行可能会在指令优化后排到第三行的前面，此时t指向的是一块空的内存空间，即null。

**不是已经双重校验了，为什么还要使用volatile？**

双重校验确实能保证并发情况下访问的是同一个对象，但是由于指令的重排序，当单例生成时t先被指向了刚初始化的空内存(null)，而在下一步调用构造方法之前又有其他线程访问t，这会导致返回的t是空值。因此，只有使用了volatile才能真正保证并发情况下每个线程得到的是非空的单例对象。



## 硬件层并发优化基础

### 缓存行

当读取内存数据到缓存中时，cpu会根据操作系统的配置读取定长的缓存数据到cpu的高速缓存中，目前多数实现为64字节。使用缓存行对齐能在一定程度上提高效率。

### 伪共享

![img](http://www.kylescloud.top/site/pic/falseshare.jpg)

对于一台拥有多个核的主机来说，这些cpu共享l3及以下的缓存，但是l1、l2的缓存却是独有的，因此会引发数据不一致的问题。从前该问题的解决方法是通过在l3上加上**总线锁**，即当某个l2访问l3时其他l2会阻塞。阻塞运行的方式会导致效率偏低。**新的CPU使用各种各样的缓存一致性协议，如MESI(Intel)**。



## 乱序问题

CPU为了提高指令执行效率，会在一条指令执行过程中，去同时执行另一条指令，前提是两条指令没有依赖关系。举例如下：

![img](http://kylescloud.top/site/pic/cpumisorder.jpg)



### 有序性保障——CPU内存屏障

**sfence**：在sfence指令前的写操作必须在sfence指令后的写操作前完成

**lfence**：在lfence指令前的读操作必须在lfence指令后的读操作前完成

**mfence**：在mfence指令前的操作必须在mfence指令后的操作前完成



### 有序性保障——JVM规范(JSR133)

#### LoadLoad屏障

对于指令Load1；LoadLoad；Load2，在Load2以及之后读取操作要读取的数据被访问前，保证Load1要读取的数据被读取完毕

#### StoreStore屏障

对于指令Store1；StoreStore；Store2，在Store2以及之后读取操作要读取的数据被访问前，保证Store1要读取的数据被读取完毕

#### LoadStore屏障

对于指令Load；LoadStore；Store，在Store以及之后读取操作要读取的数据被访问前，保证Load要读取的数据被读取完毕

#### SotrLoad屏障

对于指令Store；StoreLoad；Load，在Load以及之后读取操作要读取的数据被访问前，保证Store要读取的数据被读取完毕