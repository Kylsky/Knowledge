# ArrayList源码分析

## 一、属性

### **modCount**

该字段表示list结构上被修改的次数。结构上的修改指的是那些改变了list的长度大小或者使得遍历过程中产生不正确的结果的其它方式。该属性位于ArrayList的父类AbstractList中，被iterator(迭代器)使用，如果modCount在预期之外的情况下改变，就会回应在next、remove、previous等方法时抛出ConcurrentModificationException

### **DEFAULT_CAPACITY**

默认的容量，值为10

### **EMPTY_ELEMENTDATA**

空数据集，是一个Object数组，默认为{}

### **DEFAULTCAPACITY_EMPTY_ELEMENTDATA**

默认容量下的空元素集，值为{}

### **elementData**

当前实例存放数据的地方，是一个Object数组

### **size**

当前实例的所有数据的长度

### **MAX_ARRAY_SIZE**

最大的数组长度，为Integer的最大值-8

## 二、构造方法

ArrayList含有3个构造函数，分别为

### **ArrayList(int)**

接收一个整型数字，若大于0，则创建长度为该值的Object数组

```java
public ArrayList(int initialCapacity) {
        if (initialCapacity > 0) {
            this.elementData = new Object[initialCapacity];
        } else if (initialCapacity == 0) {
            this.elementData = EMPTY_ELEMENTDATA;
        } else {
            throw new IllegalArgumentException("Illegal Capacity: "+
                                               initialCapacity);
        }
    }
```

### ArrayList()

创建一个新的ArrayList，默认为DEFAULTCAPACITY_EMPTY_ELEMENTDATA，即一个空的Object数组

```java
public ArrayList() {
        this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
    }
```

### ArrayList(Collection<? extends E> )

接收一个集合对象，若传来的并非Object对象，则使用Arrays.copyOf将其转换为Object对象

```java
public ArrayList(Collection<? extends E> c) {
        elementData = c.toArray();
        if ((size = elementData.length) != 0) {
            // c.toArray might (incorrectly) not return Object[] (see 6260652)
            if (elementData.getClass() != Object[].class)
                elementData = Arrays.copyOf(elementData, size, Object[].class);
        } else {
            // replace with empty array.
            this.elementData = EMPTY_ELEMENTDATA;
        }
    }
```

## 三、方法

### get(int index)

```java
public E get(int index) {
    //检查index是否合法
    rangeCheck(index);
    return elementData(index);
}
```



### set(int index, E element)

```java
public E set(int index, E element) {
    //检查index是否合法
    rangeCheck(index);
    //获取老的值
    E oldValue = elementData(index);
    //设置新的值
    elementData[index] = element;
    //返回旧值
    return oldValue;
}
```



### *add(E e)

```java
public boolean add(E e) {
    //先判断是否需要扩容
    ensureCapacityInternal(size + 1);  // Increments modCount!!
    //添加数据
    elementData[size++] = e;
    return true;
}
```



### *add(int index, E element)

```java
public void add(int index, E element) {
    //参数检查
    rangeCheckForAdd(index);
	//判断是否需要扩容
    ensureCapacityInternal(size + 1);  // Increments modCount!!
    //调用本地方法拷贝数组并在index留出空间
    System.arraycopy(elementData, index, elementData, index + 1,
                     size - index);
    //为index处的元素赋值
    elementData[index] = element;
    size++;
}
```



### *remove(int index)

```java
public E remove(int index) {
    rangeCheck(index);

    modCount++;
    E oldValue = elementData(index);
	//需要移动的数据
    int numMoved = size - index - 1;
    if (numMoved > 0)
        System.arraycopy(elementData, index+1, elementData, index,
                         numMoved);
    //用于GC，防止内存泄漏
    elementData[--size] = null; 

    return oldValue;
}
```



### remove(Object o)

```java
public boolean remove(Object o) {
    if (o == null) {
        for (int index = 0; index < size; index++)
            if (elementData[index] == null) {
                fastRemove(index);
                return true;
            }
    } else {
        for (int index = 0; index < size; index++)
            if (o.equals(elementData[index])) {
                fastRemove(index);
                return true;
            }
    }
    return false;
}
```



### *batchRemove(Collection<?> c, boolean complement)

```java
private boolean batchRemove(Collection<?> c, boolean complement) {
    final Object[] elementData = this.elementData;
    int r = 0, w = 0;
    boolean modified = false;
    try {
        for (; r < size; r++)
            //假设complement为false，只要elementData中的元素在c中不存在，就将覆写到elementData中，这种情况一般用于removeAll方法
            //假设complement为true，只要elementData中的元素在c中存在，就将其覆写到elementData中，这种情况一般用于retain方法
            if (c.contains(elementData[r]) == complement)
                elementData[w++] = elementData[r];
    } finally {
        //c.contains使用iterator操作数组对象，因此可能抛出异常导致r!=size,此时将r位置后没有操作过的数据复制到elementData的w位置之后
        if (r != size) {
            System.arraycopy(elementData, r,
                             elementData, w,
                             size - r);
            //更新w
            w += size - r;
        }
        //清除w之后多余的数据，更新modcount，设置modified为true
        if (w != size) {
            // clear to let GC do its work
            for (int i = w; i < size; i++)
                elementData[i] = null;
            modCount += size - w;
            size = w;
            modified = true;
        }
    }
    return modified;
}
```



### *fastRemove(int index)

功能和remove(int index)一样，不过这个方法不会返回旧值

```java
private void fastRemove(int index) {
    modCount++;
    int numMoved = size - index - 1;
    if (numMoved > 0)
        System.arraycopy(elementData, index+1, elementData, index,
                         numMoved);
    elementData[--size] = null; 
}
```



### *ensureCapacity(int 

### minCapacity)

````java
public void ensureCapacity(int minCapacity) {
        int minExpand = (elementData != DEFAULTCAPACITY_EMPTY_ELEMENTDATA)
            // any size if not default element table
            ? 0
            // larger than default for default empty table. It's already
            // supposed to be at default size.
            : DEFAULT_CAPACITY;

        if (minCapacity > minExpand) {
            ensureExplicitCapacity(minCapacity);
        }
}
````

判断elementData是否为空，若是空的，则minCapacity只有大于默认容量（10)才需扩容。若不是空的,则可以扩容



### ensureExplicitCapacity(int minCapacity)

判断minCpacity是否大于elementData.length,若大于0，则使用grow进行扩容

```java
private void ensureExplicitCapacity(int minCapacity) {
        modCount++;

        // overflow-conscious code
        if (minCapacity - elementData.length > 0)
            grow(minCapacity);
    }
```



### *grow(int minCpacity)

````java
private void grow(int minCapacity) {
        // overflow-conscious code
        int oldCapacity = elementData.length;
        int newCapacity = oldCapacity + (oldCapacity >> 1);
        if (newCapacity - minCapacity < 0)
            newCapacity = minCapacity;
        if (newCapacity - MAX_ARRAY_SIZE > 0)
            newCapacity = hugeCapacity(minCapacity);
        // minCapacity is usually close to size, so this is a win:
        elementData = Arrays.copyOf(elementData, newCapacity);
    }
````

扩容后的容量为原容量的1.5倍，若扩容后的容量仍小于minCapacity，则直接将新的容量置为minCapacity,随后复制数组，更新elementData



### trimToSize()

扩容后elementData会存在某些下标对应元素为空，浪费空间，当size<小于elementData.length，则使elementData更新



### calculateCapacity(Object[] elementData,int minCapacity)

功能与ensureCapacity相似，判断elementData是否为空，若为空，比较10与minCapacity哪个大并返回。若不为空，则返回minCapacity。

```java
private static int calculateCapacity(Object[] elementData, int minCapacity) {
        if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
            return Math.max(DEFAULT_CAPACITY, minCapacity);
        }
        return minCapacity;
    }
```



**问**：既然功能相似，为什么还要分成两个方法？

**答**：功能相似但不相同，ensureCapacity是public方法，最终有可能调用ensureExplicitCapacity实现扩容，而calculateCapacity是private的，只是返回了一个整形，并不会进行扩容。



### *subList(int head,int tail)

生成一个当前List的从head到tail的开闭区间的视图，对视图进行增删操作会出现fast-fail（快速失败），同时使用set方法会更新源列表



### rangeCheck(int index)

```java
private void rangeCheck(int index) {
    if (index >= size)
        throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
}
```



### hugeCapacity(int minCapacity)

```
private static int hugeCapacity(int minCapacity) {
    if (minCapacity < 0) // overflow
        throw new OutOfMemoryError();
    return (minCapacity > MAX_ARRAY_SIZE) ?
        Integer.MAX_VALUE :
        MAX_ARRAY_SIZE;
}
```



### *System.arraycopy

```
public static native void arraycopy(Object src,  int  srcPos,
                                    Object dest, int destPos,
                                    int length);
```

参考链接：

```
https://blog.csdn.net/wenzhi20102321/article/details/78444158?utm_medium=distribute.pc_relevant.none-task-blog-BlogCommendFromBaidu-1.nonecase&depth_1-utm_source=distribute.pc_relevant.none-task-blog-BlogCommendFromBaidu-1.nonecase
```

**src**

源数组

**srcPos**

源数组要复制的起始位置

**dest**

目标数组

**destPos**

目标数组要复制的起始位置

**length**

复制的长度



## 四、Iterator

Iterator值迭代器，其能让开发者免去对数组的直接遍历操作，以下是该接口的内容：

```java
public interface Iterator<E> {
    //当前元素是否是最后一个元素
    boolean hasNext();
    //当前元素的下一个元素
    E next();
    //移除当前元素
    default void remove() {
        throw new UnsupportedOperationException("remove");
    }
    //接收Consumer函数式接口，通过实现accept方法对数据进行处理
	default void forEachRemaining(Consumer<? super E> action) {
        Objects.requireNonNull(action);
        while (hasNext())
            action.accept(next());
    }
}
```

ArrayList中有很多Iterator的实现类，看一下Itr：

```java
private class Itr implements Iterator<E> {
    	//游标，指向下一个需要返回的元素，默认为0
        int cursor;       // index of next element to return
        //上一次操作的数组下标
    	int lastRet = -1; // index of last element returned; -1 if no such
        int expectedModCount = modCount;

        Itr() {}
		//若cursor=size，则是最后一个元素
        public boolean hasNext() {
            return cursor != size;
        }

        @SuppressWarnings("unchecked")
        public E next() {
            checkForComodification();
            //创建cursor 副本，检查cursor范围
            int i = cursor;
            if (i >= size)
                throw new NoSuchElementException();
            Object[] elementData = ArrayList.this.elementData;
            //failfast
            if (i >= elementData.length)
                throw new ConcurrentModificationException();
            //游标指向下一个元素了
            cursor = i + 1;
            //更新lastRet并返回
            return (E) elementData[lastRet = i];
        }

        public void remove() {
            if (lastRet < 0)
                throw new IllegalStateException();
            checkForComodification();

            try {
                //删除lastRet下标的元素，同时更新cursor到lastRet，并使lastRet=-1，更新expectedModCount
                ArrayList.this.remove(lastRet);
                cursor = lastRet;
                lastRet = -1;
                expectedModCount = modCount;
            } catch (IndexOutOfBoundsException ex) {
                //failfast
                throw new ConcurrentModificationException();
            }
        }

        @Override
        @SuppressWarnings("unchecked")
        public void forEachRemaining(Consumer<? super E> consumer) {
            Objects.requireNonNull(consumer);
            final int size = ArrayList.this.size;
            int i = cursor;
            if (i >= size) {
                return;
            }
            final Object[] elementData = ArrayList.this.elementData;
            if (i >= elementData.length) {
                throw new ConcurrentModificationException();
            }
            
            
            while (i != size && modCount == expectedModCount) {
                consumer.accept((E) elementData[i++]);
            }
            
            
            // 更新cursor和lastRet
            cursor = i;
            lastRet = i - 1;
            checkForComodification();
        }

    	//检查expectedModCount
        final void checkForComodification() {
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
        }
}
```

ListIterator的实现方式其实也是相似的，具体可以看链接：

```
https://blog.csdn.net/anlian523/article/details/103059566
```



## 五、Fail-Fast

当modCount != expectedModCount时，就会抛出该异常。但是在一开始的时候，expectedModCount初始值默认等于modCount，为什么会出现modCount != expectedModCount，很明显expectedModCount在整个迭代过程除了一开始赋予初始值modCount外，并没有再发生改变，所以可能发生改变的就只有modCount，在前面关于ArrayList扩容机制的分析中，可以知道在ArrayList进行add，remove，clear等涉及到修改集合中的元素个数的操作时，modCount就会发生改变(modCount ++),所以当另一个线程(并发修改)或者同一个线程遍历过程中，调用相关方法使集合的个数发生改变，就会使modCount发生变化，这样在checkForComodification方法中就会抛出ConcurrentModificationException异常。