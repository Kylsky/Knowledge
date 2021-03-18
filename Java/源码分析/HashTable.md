# HashTable

在理解hashTable的源码前，先列出以下结论，将其与HashMap做一下比较，这样在阅读时更有针对性

```
1.HashTable无论是键值对都不能存放null，而HashMap可以
2.HashTable初始容量为11，而HashMap为16
3.HashTable的扩容制度为capacity*2+1，而HashMap为capacity*2
4.HashTable的index方法为(hash&0x7FFFFFFF)%table.length，而HashMap的index方法为hash&(table.length-1)
5.HashTable线程安全，而HashMap线程不安全
```

接下来看看对应源码，由于HashTable和HashMap同样使用了哈希表，因此一些相似的操作就不贴出来了，但是需要注意尽管是相同或相似的操作，HashTable中都是使用synchronized修饰过的，因此是线程相对安全的。



## 1.构造方法

### Hashtable(int initialCapacity, float loadFactor)

````java
public Hashtable(int initialCapacity, float loadFactor) {
    //判断变量是否有效
	if (initialCapacity < 0)
	    throw new IllegalArgumentException("Illegal Capacity: "+
                                               initialCapacity);
    //判断变量是否有效
    if (loadFactor <= 0 || Float.isNaN(loadFactor))
            throw new IllegalArgumentException("Illegal Load: "+loadFactor);
	//特殊值处理
    if (initialCapacity==0)
            initialCapacity = 1;
	this.loadFactor = loadFactor;
	table = new Entry[initialCapacity];
	threshold = (int)(initialCapacity * loadFactor);
    }
````



### Hashtable(int initialCapacity)

```java
public Hashtable(int initialCapacity) {
    //调用上面那个构造方法，loadFactor为默认的0.75
	this(initialCapacity, 0.75f)
}
```



### Hashtable()

```java
public Hashtable() {
    //设置默认容量为11，loadFactor为0.75
	this(11, 0.75f);
}
```



### Hashtable(Map<? extends K, ? extends V> t)

传入一个Map，并判断map容量*2与11哪个大，最后将元素传入hashTable中

```java
public Hashtable(Map<? extends K, ? extends V> t) {
	this(Math.max(2*t.size(), 11), 0.75f);
	putAll(t);
}
```



## 2.方法

### contains(Object value)

```java
public synchronized boolean contains(Object value) {
	if (value == null) {
	    throw new NullPointerException();
	}

	Entry tab[] = table;
	for (int i = tab.length ; i-- > 0 ;) {
	    for (Entry<K,V> e = tab[i] ; e != null ; e = e.next) {
			if (e.value.equals(value)) {
		    	return true;
			}
	    }
	}
	return false;
}
```

由于HashTable要求存储的对象的key和value均不能为null，所以若对象为null，则直接抛出空指针异常。若对象不为空，则通过for循环进行查找，**这里比较有趣的是**，查找是从HashTable的末尾开始的，另外，个人认为contains方法和containsValue方法在一定程度上意思是一样的，那么来看看containsValue吧



### containsValue(Object value)

```java
public boolean containsValue(Object value) {
	return contains(value);
}
```

哦，原来是调用了contains方法啊。为什么要这么做，我想是因为如果多线程条件下使用containsValue，为了保证数据一致性，那么加锁是必要的。此时通过对contains方法来间接锁定对象，而不是直接在containsValue上上锁，我认为这制造了一种非同步的假象



### containsKey(Object key)

```java
public synchronized boolean containsKey(Object key) {
	Entry tab[] = table;
	int hash = key.hashCode();
	int index = (hash & 0x7FFFFFFF) % tab.length;
	for (Entry<K,V> e = tab[index] ; e != null ; e = e.next) {
	    if ((e.hash == hash) && e.key.equals(key)) {
			return true;
	    }
	}
	return false;
}
```

这里可以去和HashMap的containsKey方法作比较，两者在映射对象的方式上的区别可见一斑，HashTable采用的映射算法为(hash & 0x7FFFFFFF) % tab.length，为什么选用这样的算法，后面会做说明。



### put(K,V)

```
public synchronized V put(K key, V value) {
	// 保证value不为空
	if (value == null) {
	    throw new NullPointerException();
	}
	Entry tab[] = table;
	int hash = key.hashCode();
	int index = (hash & 0x7FFFFFFF) % tab.length;
	for (Entry<K,V> e = tab[index] ; e != null ; e = e.next) {
	    if ((e.hash == hash) && e.key.equals(key)) {
		V old = e.value;
		e.value = value;
		return old;
	    }
	}

	modCount++;
	if (count >= threshold) {
	    // Rehash the table if the threshold is exceeded
	    rehash();

            tab = table;
            index = (hash & 0x7FFFFFFF) % tab.length;
	}

	// Creates the new entry.
	Entry<K,V> e = tab[index];
	tab[index] = new Entry<K,V>(hash, key, value, e);
	count++;
	return null;
}
```

除了前面提到的映射算法不同，这里与HashMap的操作基本相似，重要的还是看看put过程中可能会遇到的扩容(rehash)操作



### rehash()

```java
protected void rehash() {
	int oldCapacity = table.length;
	Entry[] oldMap = table;

	int newCapacity = oldCapacity * 2 + 1;
	Entry[] newMap = new Entry[newCapacity];

	modCount++;
	threshold = (int)(newCapacity * loadFactor);
	table = newMap;

	for (int i = oldCapacity ; i-- > 0 ;) {
	    for (Entry<K,V> old = oldMap[i] ; old != null ; ) {
		Entry<K,V> e = old;
		old = old.next;

		int index = (e.hash & 0x7FFFFFFF) % newCapacity;
		e.next = newMap[index];
		newMap[index] = e;
	    }
	}
}
```

可以看到**int newCapacity = oldCapacity * 2 + 1;**扩容机制将容量扩大到了原容量的两倍+1，我想这和映射算法是有关的，因此同样放到后面一起解释吧~另外，由于rehash只在put中被调用，因此不需要在方法上加上synchronized做修饰



### clear()

```java
public synchronized void clear() {
	Entry tab[] = table;
	modCount++;
	for (int index = tab.length; --index >= 0; )
    tab[index] = null;
	count = 0;
}
```

将整个HashTable清空



## 3.扩容机制

HashTable的一些属性基本与HashMap没有太大差别，而且由于HashTable不存在红黑树等操作，因此相对简单，Enumeration类的属性作为历史遗留问题也不做介绍了，下面主要讲讲扩容机制吧。网上有一篇总结的比较好的文章，贴出部分内容，侵删：

```
链接：https://blog.csdn.net/weixin_41884010/article/details/100886250

Hashtable中数组的长度尽量为素数或者奇数，同时Hashtable采用取模的方式来计算数组下标，这样减少Hash碰撞，计算出来的数组下标更加均匀。但是这样效率会比HashMap利用位运算计算数组下标低。

采用头插法的方式效率更高。如果采用尾插法需要遍历数组将元素放置到链表的末尾，而采用头插法将结点放置到链表的头部，减少了遍历数组的时间，效率更高。

Hashtable是线程相对安全的，所以Hashtable不需要考虑并发冲突问题，可以采用效率更高的头插法。
```



## 4.关于键值null的约束

```
https://juejin.cn/post/6844904023363960839
```





