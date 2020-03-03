# 多线程&高并发(三)——ThreadLocal

## ThreadLocal

ThreadLocal修饰的变量对于每一个线程来说都是独有的，因此不会产生并发的问题。ThreadLocal通过**set**方法为变量赋值，看一下代码

```java
public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
}
```

可以看到在set中通过一个ThreadLocalMap类保存了value值，这个map是由**getMap**方法获取到的，同时可以发现getMap传入了当前线程的引用，那么来看一下代码：

```java
ThreadLocalMap getMap(Thread t) {
    return t.threadLocals;
}
```

发现getMap方法返回的是t.threadLocals，t是Thread类型，也就是说ThreadLocalMap保存的是当前传入的线程的数据。



## 引用的强|软|弱|虚

由于ThreadLocal会涉及弱引用的相关问题，这里简单介绍一下java的四种引用

### 强引用

```java
Object o = new Object();
System.out.println(o);
System.gc();
System.out.println(o);
```

若始终有引用指向一个对象，那么这个对象就不会被gc回收，因此称为强引用

### 软引用

```java
SoftReference<byte[]> m = new SoftReference<>(new byte[1024]);
M m1 = m.get();
System.out.println(m1);
System.gc();
System.out.println(m1);
```

若内存不够，则gc会直接回收软引用，否则不会回收。软引用一般可以用来做**缓存**，如存放图片对象等。

### 弱引用

```java
WeakReference<M> weakReference = WeakReference(new M());
System.out.println(weakReference.get());
System.gc();
System.out.println(weakReference.get());
```

弱引用一旦遇到gc就会被回收。一般会用在容器中，如**WeakHashMap**，**ThreadLocal**

### 虚引用

```java
public class TestPhantomReference {
    private static final List<Object> LIST = new LinkedList<>();
    private static final ReferenceQueue<Object> QUEUE = new ReferenceQueue<>();

    public static void main(String[] args) {
        PhantomReference<Object> phantomReference = new PhantomReference<>(new Object(),QUEUE);

        new Thread(()->{
            while (true){
                LIST.add(new byte[1024*1024]);
                try {
                    TimeUnit.SECONDS.sleep(1);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println(phantomReference.get());
            }
        }).start();
        
        new Thread(()->{
            while (true){
                Reference<? extends Object> o = QUEUE.poll();
                if (o!=null){
                    System.out.println("---------phantom reference appear");
                }
            }
        }).start();
    }
}
```

在运行代码之前，将jvm的最大内存和最小内存设置问20M。PhantomReference通过虚引用指向一个对象和一个引用队列，当引用被回收，则将引用放入一个队列中。虚引用的值通过get方法是无法获取的。

使用场景，NIO使用DirectByteBuffer直接操作对外内存，是JVM无法回收的，因此可以通过虚引用监测**DirectByteBuffer**，若需要回收则创建虚引用并回收，通过Queue(队列)通知操作系统需要回收的内存。



##  ThreadLocalMap

简述了java中的四种引用，那么回到ThreadLocalMap中，由于Map类是以键值对形式存放对象的，那么ThreadLocalMap作为一个Map，应该也是如此。Map用Entry来存放ThreadLocal对象，因此看到ThreadLocalMap中的**Entry**类：

```java
static class Entry extends WeakReference<ThreadLocal<?>> {
    /** The value associated with this ThreadLocal. */
    Object value;

    Entry(ThreadLocal<?> k, Object v) {
        super(k);
        value = v;
    }
}
```

可以发现Entry的构造方法中使用了super(k)，等价于new WeakReference(k)，意为作为键的ThreadLocal对象k是一个弱引用，之所以对该对象使用弱引用，是因为即使令一个线程中的ThreadLocal对象为空，但是Entry中的key仍会指向这个ThreadLocal对象，若k为强引用，那么ThreadLocal对象就不会被回收，导致内存泄漏，因此需要使用weakReference。

不过即使令k为弱引用，也会出现内存泄漏问题，因为当ThreadLocal对象被回收后，key变为null，但是该键值对仍然存在，并可能无法被访问，也会存在内存泄漏情况。

因此考虑到上面key为null的情况，则需要通过ThreadLocal的remove方法将其从ThreadLocalMap中移除

