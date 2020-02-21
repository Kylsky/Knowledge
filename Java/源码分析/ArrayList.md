# ArrayList源码分析

## 属性

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

## 构造方法

ArrayList含有3个构造函数，分别为

### **ArrayList(int)**

接收一个整型数字，若大于0，则创建长度为该值的Object数组

### **ArrayList()**

创建一个新的ArrayList，默认为DEFAULTCAPACITY_EMPTY_ELEMENTDATA，即一个空的Object数组

### **ArrayList(Collection<? extends E> )**

接收一个集合对象，若传来的并非Object对象，则使用Arrays.copyOf将其转换为Object对象

## 方法

增删改查等方法不一一介绍了，由于ArrayList是一个动态的数组，因此主要关注它是如何进行动态容量改变的，看以下的方法

### **ensureCapacity(int minCapacity)**

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

### **ensureExplicitCapacity(int minCapacity)**

判断minCpacity是否大于elementData.length,若大于0，则使用grow进行扩容

### **grow(int minCpacity)**

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

### **trimToSize()**

扩容后elementData会存在某些下标对应元素为空，浪费空间，当size<小于elementData.length，则使elementData更新

### **calculateCapacity(Object[] elementData,int minCapacity)**

功能与ensureCapacity相似，判断elementData是否为空，若为空，比较10与minCapacity哪个大并返回。若不为空，则返回minCapacity。

**问**：既然功能相似，为什么还要分成两个方法？

**答**：功能相似但不相同，ensureCapacity是public方法，最终有可能调用ensureExplicitCapacity实现扩容，而calculateCapacity是private的，只是返回了一个整形，并不会进行扩容。

### subList(int head,int tail)

生成一个当前List的从head到tail的开闭区间的视图，对视图进行增删操作会出现fast-fail（快速失败），同时使用set方法会更新源列表