# JVM(二)——类加载

## 一、Class加载-初始化

![classloading](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/classloading.jpg)

### loading

将一个class文件装载到内存



### linking

**1.verification**

校验内存中的class文件格式

**2.preparaton**

为class的静态变量赋默认值，如static int i=8，则为i赋默认值为0

**3.resolution**

```
关于符号引用和直接引用，可以参考：
https://www.cnblogs.com/shinubi/articles/6116993.html
```

将符号引用转换为内存地址。解析动作主要针对**类或接口、字段、类方法、接口方法、方法类型、方法句柄**和**调用点限定符**这7类符号引用进行。



### initializing

静态变量赋值为初始值



## 二、类加载器

![classloader](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/classloader.jpg)

当一个class文件被load到内存中时，内存为class开辟两个区域，一个区域用来存储class文件中的二进制数，另一个区域用于存放创建的java语义下的class对象，该对象指向内存中的class二进制文件。上图**需要注意的是**，BootStrap加载器是c++实现的模块，所以在java代码中无法找到，而下面的ExtClassLoader和AppClassLoader虽然使用java代码实现，但是这两者或这三者不存在继承关系，且ExtClassLoader和AppClassLoader属于Lanucher类的内部类，它们的类加载器由c++实现，不存在层级关系。



###  ClassLoader

类加载器的顶级类，所有的类加载器都需要继承该类。看一下核心方法loadClass，这里体现了classloader的双亲委派机制：

````java
protected Class<?> loadClass(String name, boolean resolve)
    throws ClassNotFoundException
{
    synchronized (getClassLoadingLock(name)) {
        // First, check if the class has already been loaded
        //检查class是否已经被加载到缓存中
        Class<?> c = findLoadedClass(name);
        if (c == null) {
            long t0 = System.nanoTime();
            try {
                //存在父加载器，让父加载器去找
                if (parent != null) {
                    c = parent.loadClass(name, false);
                } 
                //不存在父加载器，让bootstrap去找
                else {
                    c = findBootstrapClassOrNull(name);
                }
            } catch (ClassNotFoundException e) {
                // ClassNotFoundException thrown if class not found
                // from the non-null parent class loader
            }

            if (c == null) {
                // If still not found, then invoke findClass in order
                // to find the class.
                long t1 = System.nanoTime();
                //这里的findClass是一个模板方法，通过调用defineClass将class文件的二进制
                //流转换为Class类对象
                c = findClass(name);

                // this is the defining class loader; record the stats
                sun.misc.PerfCounter.getParentDelegationTime().addTime(t1 - t0);
                sun.misc.PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
                sun.misc.PerfCounter.getFindClasses().increment();
            }
        }
        if (resolve) {
            resolveClass(c);
        }
        return c;
    }
}
````



## 三、双亲委派机制

当一个class被需要时，jvm向当前class对应的类加载器检查class是否位于该类加载器的缓存中，若类加载器显示不存在，则jvm向当前类加载器的父类加载器检查，若存在则返回，否则依此类推直到检查到BootStrap ClassLoader，当处理到顶层类加载器BootStrap ClassLoader，BootStrap会根据自己加载的类库判断是否存在class信息，若有则加载class文件并返回，若无则向低一层类加载器查询，以此类推，直到检查到class并加载返回或最终没有检擦到class抛出ClassNotFound异常。流程图如下：

![classloading2](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/classloading2.jpg)

### 为什么需要双亲委派机制？

双亲委派机制保证了java核心类库的**安全性**。假设开发者使用自定义类加载器重写了核心类库中的类如String，且不存在双亲委派机制，新的String类使得用户在一些涉及隐私的操作如输入密码中使用了被重写的String类从而导致密码的泄露，后果会不堪设想。而当双亲委派机制存在时，java核心类会在程序最初就被load进内存，当开发者视图创建自定义的与核心类相同的类并初始化时，jvm会向上检查到已经加载了该核心类，导致重写的自定义类无效。



## 四、混合编译模式

java将class文件load到内存后通过解释器(bytecode interpreter)**解释后**执行相关操作，也可以通过JIT(just in time)编译器将class文件**编译**成本地代码执行。

java通常采用混合模式，即混合使用解释器+热点代码编译。热点代码包含多次被调用的方法(通过方法计数器)和多次被调用的循环(通过循环计数器)。

虽然将文件编译成本地代码执行速度较快，但是一般由于class文件很多，会导致编译时间过长，影响程序启动时间。在jvm中，使用**-Xmixed**表示混合模式，**-Xint**表示解释模式，**-Xcom**表示编译模式。