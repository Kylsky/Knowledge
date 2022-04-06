# HashMap源码分析

## 一、前言

### 什么是Hash？

在谈论HashMap前，先讨论一下Hash。Hash，就是把任意长度的输入通过散列算法，变换成固定长度的输出，该输出就是散列值。这种转换是一种压缩映射，也就是，散列值的空间通常远小于输入的空间，不同的输入可能会散列成相同的输出，而不可能从散列值来唯一的确定输入值。简单的说就是一种将任意长度的消息压缩到某一固定长度的消息摘要的函数。

### HashMap

哈希表是根据设定的哈希函数H（key）和处理冲突方法将一组关键字映射到一个有限的地址区间上，并以关键字在地址区间中的象作为记录在表中的存储位置，这种表称为哈希表或散列，所得存储位置称为哈希地址或散列地址。作为线性数据结构与表格和队列等相比，哈希表是查找速度比较快的一种。

HashMap以键值对的形式存储数据，且键可以为null。这里会关键讨论到hash表在Java中的实现以及解决冲突的方法。

![img](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/5679451-c6e80d30b6653324.png)



## 二、属性

### *1.DEFAULT_INITIAL_CAPACITY

默认容量，16

### 2.MAXIMUM_CAPACITY

最大容量，2^30

### *3.DEFAULT_LOAD_FACTOR

默认装填因子，0.75

### *4.TREEIFY_THRESHOLD

默认值为8，在hashmap中，不同的key可能经过hash函数会映射成相同的哈希值，这些哈希值相同的数据将会放在一条链表中或一棵红黑树中存储。在初始时，这些数据会形成链表，当链表长度大于8时，链表会转换成红黑树

### *5.TREEIFY_THRESHOLD

默认值为6，当上述的数据长度小于6时，红黑树又会转回链表以达到性能均衡

### 6.MIN_TREEIFY_CAPACITY

默认值为64，最小树形化容量阈值。当哈希表中的容量大于该值，则将链表树形化。为了避免进行扩容、树形化选择的冲突，这个值不能小于 4 * **TREEIFY_THRESHOLD**

### 7.Node

Node是Entry的实现类，HashMap中每一个K，V键值对数据都存放在一个Entry(Node)中。Node主要有以下属性及方法

**hash**

用来记录节点的hash值

**key**

用来记录一个key

**value**

用来记录一个value

**next**

Node类型，用来指向下一个相同hash值的节点

**Node(int hash,K key ,V value,Node<K,V>)**

构造函数

### 8.table

table是一个Node<K,V>数组，用来表示一个HashMap对象的所有的初始节点的节点集。每一个table的项被称为桶

### 9.entrySet

是一个Set<Map.Entry<K,V>>，存放了HashMap中的所有节点

### 10.size

HashMap中包含的键值对的数量

### 11.modCount

HashMap结构修改次数

### 12.threshold

threshold=capacity*loadFactor

当HashMap的size大于threshold时会执行resize操作

### 13.loadFactor

装载因子。用于hashMap的扩容。

### 14.KeySet

用于存放key

### 15.Value

用来存放Values

### 16.EntrySet

用于存放Entry

### 17.TreeNode

红黑树的实现



## 三、构造函数

### HashMap(int initialCapacity,float loadFactor)

传入初始化容量和装填因子

````java
public HashMap(int initialCapacity, float loadFactor) {
        if (initialCapacity < 0)
            throw new IllegalArgumentException("Illegal initial capacity: " +
                                               initialCapacity);
        if (initialCapacity > MAXIMUM_CAPACITY)
            initialCapacity = MAXIMUM_CAPACITY;
        if (loadFactor <= 0 || Float.isNaN(loadFactor))
            throw new IllegalArgumentException("Illegal load factor: " +
                                               loadFactor);
        this.loadFactor = loadFactor;
        this.threshold = tableSizeFor(initialCapacity);
    }
````

这里需要留心最后的tableSizeFor方法，这里会对传入的initialCapacity做判断并进行优化后再初始化Hash Map的初始容量

### Hash(int initialCapacity)

````java
public HashMap(int initialCapacity) {
        this(initialCapacity, DEFAULT_LOAD_FACTOR);
    }
````

这里不做过多说明，下同

### HashMap()

````java
public HashMap() {
        this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
    }
````

### HashMap(Map<? extends K,? extends V> m)

````java
public HashMap(Map<? extends K, ? extends V> m) {
        this.loadFactor = DEFAULT_LOAD_FACTOR;
        putMapEntries(m, false);
}
````

将参数作为新的hashMap的entry，putMapEntries将在下面介绍



## 四、方法

### putMapEntries(Map<? extends K, ? extends V> m, boolean evict) 

````java
final void putMapEntries(Map<? extends K, ? extends V> m, boolean evict) {
        int s = m.size();
        if (s > 0) {
            if (table == null) { // pre-size
                float ft = ((float)s / loadFactor) + 1.0F;
                int t = ((ft < (float)MAXIMUM_CAPACITY) ?
                         (int)ft : MAXIMUM_CAPACITY);
                if (t > threshold)
                    threshold = tableSizeFor(t);
            }
            else if (s > threshold)
                resize();
            for (Map.Entry<? extends K, ? extends V> e : m.entrySet()) {
                K key = e.getKey();
                V value = e.getValue();
                putVal(hash(key), key, value, false, evict);
            }
        }
    }
````

用于实现putAll()以及构造方法，主要功能为将参数中的Map传递到当前实例中。

传入Map时先判断是否为初始化HashMap的情况，即table是否为空。

若table为空，需要考虑Map的对应的threshold值是否会超出当前HashMap的threshold，因为threshold=capacity*loadFactor，所以传入的threshold为m.size()/loadFactor，考虑计算结果可能为小数，因此为了保守计算容量加上1。随后将计算出的的threshold与当前实例进行比较，随后通过tableSizeFor优化容量并初始化。

若table不为空，则判断s>threshold，若成立，则resize。

最后一步，就是通过for循环将数据存入实例中

### tableSizeFor(int cap)

````java
static final int tableSizeFor(int cap) {
        int n = cap - 1;
        n |= n >>> 1;
        n |= n >>> 2;
        n |= n >>> 4;
        n |= n >>> 8;
        n |= n >>> 16;
        return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
}
````

举个例子能更好的看懂代码，因为涉及到移位操作，故用二进制数作为参数举例：

令cap= 00100001(33)

则n=cap-1=00100000(32)

10

00001010

00000101

00001111



经过一系列移位后

n=00111111

再通过return中语句的执行，n最终等于01000000(32)

**问：**不难发现，其实上述操作是将传入的参数转换为最靠近的2的n次方，这是为什么呢?

**答：**HashMap将不同的键放入不同的桶中，主要采用了hash&capacity-1的方法，在计算机中，直接除以capacity运算的效率低于位运算，且sun公司的大牛们发现，当容量为2的n次方时，hash & (capacity - 1) == hash % capacity，前提就是容量必须为2的n次方。

### hash(Object key)

````java
static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }
````

jdk1.8的hash算法是将key的hashCode与其右移16位的或结果。

**问：**为什么不直接用hashCode？

**答：**上面讨论到hashmap将key放入到桶中使用的是hash&capacity-1，一般情况下capacity的大小都不超过低16位（65535），举16（默认容量）作为例子，在做与运算时，若使用hashCode，则hash的有效位只有低4位，如2^10,2^20,2^30在运算时就会产生冲突。因此通过抑或将hashCode的高16位与hashCode进行计算，可以减少产生冲突的几率

### get(Object key)

````java
public V get(Object key) {
        Node<K,V> e;
        return (e = getNode(hash(key), key)) == null ? null : e.value;
}
````

通过getNode方法获取到键值对

### getNode(int hash, Object key)

````java
final Node<K,V> getNode(int hash, Object key) {
        Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (first = tab[(n - 1) & hash]) != null) {
            if (first.hash == hash && // always check first node
                ((k = first.key) == key || (key != null && key.equals(k))))
                return first;
            if ((e = first.next) != null) {
                if (first instanceof TreeNode)
                    return ((TreeNode<K,V>)first).getTreeNode(hash, key);
                do {
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        return e;
                } while ((e = e.next) != null);
            }
        }
        return null;
    }
````

判断table是否为null，table长度是否大于0，桶是否存在

若存在，则first指向桶的第一个元素，且先判断first节点的key是否与传入的key相同。

若不相同，则判断first是否有下一个节点，若有下一个节点，则可能是链表，也可能是树形，若是树形，调用getTreeNode并返回结果。

若是链表，则调用while循环完成搜索。

### containsKey(Object o)

调用了getNode方法判断获取到的node是否为null

### containsValue(Object  value)

比较好理解，不贴代码了，时间复杂度为O(n^2)

### put(K,V)

````java
public V put(K key, V value) {
        return putVal(hash(key), key, value, false, true);
}
````

将键值对关联到当前map对象中，若此前map中已有该key，则会被覆盖。

### pubIfAbsent(K key,V value)

调用了putVal(hash(key), key, value, true, true)，若碰到key已存在的情况，若key对应value为空，则覆盖，不为空，则不覆盖。

### putVal(K key,V value,boolean onlyIfAbcent,boolean evict)

````java
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
    	//若table为空，则初始化threshOld
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;
    	//若桶为空，新建元素并直接插入
        if ((p = tab[i = (n - 1) & hash]) == null)
            tab[i] = newNode(hash, key, value, null);
    	//桶不为空
        else {
            Node<K,V> e; K k;
            //若hash值和key都和桶首节点相同，则可能会覆盖原有节点
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;
            //桶为树形结构
            else if (p instanceof TreeNode)
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            //桶中不存在树形结构，且key值不与首元素相同
            else {
                for (int binCount = 0; ; ++binCount) {
                    //若后续元素为空，则插入并判断是否需要树形化
                    if ((e = p.next) == null) {
                        p.next = newNode(hash, key, value, null);
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            treeifyBin(tab, hash);
                        break;
                    }
                    //后续元素不为空则执行以下语句，判断是否有相同的key
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    p = e;
                }
            }
            //若e不为空，说明有相同key，根据onlyIfAbsent判断是否需要覆盖
            if (e != null) { // existing mapping for key
                V oldValue = e.value;
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                //LinkedHashMap相关操作，暂时不讨论
                afterNodeAccess(e);
                //返回被覆盖的值
                return oldValue;
            }
        }
        ++modCount;	
    	//判断是否需要扩容
        if (++size > threshold)
            resize();
    	//LinkedHashMap相关操作，暂时不讨论
        afterNodeInsertion(evict);
        return null;
}
````

代码较多，以注释形式呈现。如果线程A和线程B同时进行put操作，刚好这两条不同的数据hash值一样，并且该位置数据为null，所以这线程A、B都会进入第8行代码中。假设一种情况，线程A进入后还未进行数据插入时挂起，而线程B正常执行，从而正常插入数据，然后线程A获取CPU时间片，此时线程A不用再进行hash判断了，问题出现：线程A会把线程B插入的数据给**覆盖**，发生线程不安全。

### resize()

````java
final Node<K,V>[] resize() {
        Node<K,V>[] oldTab = table;
        int oldCap = (oldTab == null) ? 0 : oldTab.length;
        int oldThr = threshold;
        int newCap, newThr = 0;
        if (oldCap > 0) {
            //若table长度已经是最大值，则直接返回
            if (oldCap >= MAXIMUM_CAPACITY) {
                threshold = Integer.MAX_VALUE;
                return oldTab;
            }
            //若oldCap的2倍小于最大长度且oldCap大于等于16，则newCap长度为oldCap的2倍
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                     oldCap >= DEFAULT_INITIAL_CAPACITY)
                newThr = oldThr << 1;
        }
    	//table长度为0，oldThr=16*loadFactor作为当前新的容量
        else if (oldThr > 0) // initial capacity was placed in threshold
            newCap = oldThr;
    	//oldCap和threshOld都为0，则使用默认值
        else {               
            newCap = DEFAULT_INITIAL_CAPACITY;
            newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
        }
    	//newThr可能为0，若为0，则需要通过计算初始化
        if (newThr == 0) {
            float ft = (float)newCap * loadFactor;
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                      (int)ft : Integer.MAX_VALUE);
        }
    	//更新threshOld
        threshold = newThr;
        @SuppressWarnings({"rawtypes","unchecked"})
        Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
        table = newTab;
    	//旧表不为空，则需要将旧表的数据复制到新表
        if (oldTab != null) {
            //操作每一个桶中的节点元素
            for (int j = 0; j < oldCap; ++j) {
                Node<K,V> e;
                if ((e = oldTab[j]) != null) {
                    //gc
                    oldTab[j] = null;
                    //后续元素为空
                    if (e.next == null)
                        newTab[e.hash & (newCap - 1)] = e;
                    //后续元素不为空，且是树形的
                    else if (e instanceof TreeNode)
                        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                    //后续元素不为空，且是链表
                    else { // preserve order
                        Node<K,V> loHead = null, loTail = null;
                        Node<K,V> hiHead = null, hiTail = null;
                        Node<K,V> next;
                        do {
                            next = e.next;
                            //这里是e.hash&oldCap，并不是e.hash&oldCap-1，后面解释
                            //若为0，插入第一个元素时为loHead赋值e
                            //之后每一个元素都为loTail赋值，并更新loTail到loTail.next
                            if ((e.hash & oldCap) == 0) {
                                if (loTail == null)
                                    loHead = e;
                                else
                                    loTail.next = e;
                                loTail = e;
                            }
                            //若不为0，插入第一个元素时为hiHead赋值e
                            //之后每一个元素都为hiTail赋值，并更新hiTail到hiTail.next
                            else {
                                if (hiTail == null)
                                    hiHead = e;
                                else
                                    hiTail.next = e;
                                hiTail = e;
                            }
                        } while ((e = next) != null);
                        //while循环结束后，最终会形成2条链表——loHead和hiHead
                        //loHead放在原来的位置
                        if (loTail != null) {
                            loTail.next = null;
                            newTab[j] = loHead;
                        }
                        //hiHead放在新的位置
                        if (hiTail != null) {
                            hiTail.next = null;
                            newTab[j + oldCap] = hiHead;
                        }
                    }
                }
            }
        }
        return newTab;
}
````

初始化表，或者令表长度加倍。若表为空，则按照初始容量分配容量。若不为空，由于容量为2^n，所以每个桶中元素必须保持相同的索引，或者在新的table中按照2^n规则进行偏移。代码有些长，用注释代替长篇的解释了。

**e.hash & oldCap原理**

````
引用自https://blog.csdn.net/u013494765/article/details/77837338
// (e.hash & oldCap) 得到的是 元素的在数组中的位置是否需要移动,示例如下
// 示例1：
// e.hash=10 0000 1010
// oldCap=16 0001 0000
//	 &   =0	 0000 0000       比较高位的第一位 0
//结论：元素位置在扩容后数组中的位置没有发生改变

// 示例2：
// e.hash=17 0001 0001
// oldCap=16 0001 0000
//	 &   =1	 0001 0000      比较高位的第一位   1
//结论：元素位置在扩容后数组中的位置发生了改变，新的下标位置是原下标位置+原数组长度

// (e.hash & (oldCap-1)) 得到的是下标位置,示例如下
//   e.hash=10 0000 1010
// oldCap-1=15 0000 1111
//      &  =10 0000 1010

//   e.hash=17 0001 0001
// oldCap-1=15 0000 1111
//      &  =1  0000 0001

//新下标位置
//   e.hash=17 0001 0001
// newCap-1=31 0001 1111    newCap=32
//      &  =17 0001 0001    1+oldCap = 1+16

//元素在重新计算hash之后，因为n变为2倍，那么n-1的mask范围在高位多1bit(红色)，因此新的index就会发生这样的变化：
// 0000 0001->0001 0001
````

### treeifyBin(Node<K,V>[] tab,int hash)

````java
final void treeifyBin(Node<K,V>[] tab, int hash) {
        int n, index; Node<K,V> e;
        if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY)
            resize();
        else if ((e = tab[index = (n - 1) & hash]) != null) {
            TreeNode<K,V> hd = null, tl = null;
            do {
                TreeNode<K,V> p = replacementTreeNode(e, null);
                if (tl == null)
                    hd = p;
                else {
                    p.prev = tl;
                    tl.next = p;
                }
                tl = p;
            } while ((e = e.next) != null);
            if ((tab[index] = hd) != null)
                hd.treeify(tab);
        }
    }
````

除非table太小了，需要resize，否则替换所有桶中的链表元素。

判断table是否为空，或table容量小于MIN_TREEIFY_CAPACITY（6）,则resize。

若上述条件不成立，则判断桶是否存在，存在则利用replacementTreeNode进行树形化

### putAll(Map<? extends K,? extends V> m)

调用了putMapEntries(m, true)

### putMapEntries(Map<? extends K, ? extends V> m, boolean evict)

```java
final void putMapEntries(Map<? extends K, ? extends V> m, boolean evict) {
    int s = m.size();
    if (s > 0) {
        if (table == null) { // pre-size
            float ft = ((float)s / loadFactor) + 1.0F;
            int t = ((ft < (float)MAXIMUM_CAPACITY) ?
                     (int)ft : MAXIMUM_CAPACITY);
            if (t > threshold)
                threshold = tableSizeFor(t);
        }
        else if (s > threshold)
            resize();
        for (Map.Entry<? extends K, ? extends V> e : m.entrySet()) {
            K key = e.getKey();
            V value = e.getValue();
            putVal(hash(key), key, value, false, evict);
        }
    }
}
```

判断table是否为空，若为空，则调用tableSizeFor检查是否需要扩容，并更新threshOld。

若不为空，则判断s是否大于threshOld，大于则需要扩容。

通过for循环调用putVal插入元素

### remove(Object key)

通过removeNode实现

### remove(Object key,Object value)

调用了removeNode(hash(key),key,value,true,true),只有当value相同时才会删除

### removeNode(int hash, Object key, Object value,boolean matchValue, boolean movable)

````java
final Node<K,V> removeNode(int hash, Object key, Object value,
                               boolean matchValue, boolean movable) {
        Node<K,V>[] tab; Node<K,V> p; int n, index;
    	//table不为空，长度大于0，对应桶存在
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (p = tab[index = (n - 1) & hash]) != null) {
            //node是需要被删除的元素
            Node<K,V> node = null, e; K k; V v;
            //桶首元素正好是要删除的元素
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                node = p;
            //桶首元素与key不相等，判断next
            else if ((e = p.next) != null) {
                //树形结构
                if (p instanceof TreeNode)
                    node = ((TreeNode<K,V>)p).getTreeNode(hash, key);
                //链表结构
                else {
                    do {
                        if (e.hash == hash &&
                            ((k = e.key) == key ||
                             (key != null && key.equals(k)))) {
                            node = e;
                            break;
                        }
                        p = e;
                    } while ((e = e.next) != null);
                }
            }
            //上述操作是为了找到相应的节点
            //matchValue为true时，需要判断value值是否相等，只有相等才会删除
            if (node != null && (!matchValue || (v = node.value) == value ||
                                 (value != null && value.equals(v)))) {
                //树形结构删除
                if (node instanceof TreeNode)
                    ((TreeNode<K,V>)node).removeTreeNode(this, tab, movable);
                //node==p,说明删除元素在桶第一个，直接为tabl[index]赋值即可
                else if (node == p)
                    tab[index] = node.next;
                //若node！=p,由于p指向的是node前一个节点，因此将p.next指向node.next即可
                else
                    p.next = node.next;
                ++modCount;
                --size;
                afterNodeRemoval(node);
                return node;
            }
        }
        return null;
    }
````

### merge(K key, V value,BiFunction<? super V, ? super V, ? extends V> remappingFunction)

```java
public V merge(K key, V value,
               BiFunction<? super V, ? super V, ? extends V> remappingFunction) {
    if (value == null)
        throw new NullPointerException();
    if (remappingFunction == null)
        throw new NullPointerException();
    //获取key的hash值
    int hash = hash(key);
    Node<K,V>[] tab; Node<K,V> first; int n, i;
    //桶元素数量
    int binCount = 0;
    TreeNode<K,V> t = null;
    //old用来存储旧的value值
    Node<K,V> old = null;
    //容量不足，table为空都需要resize
    if (size > threshold || (tab = table) == null ||
        (n = tab.length) == 0)
        n = (tab = resize()).length;
    //判断hash&值对应桶存在与否
    if ((first = tab[i = (n - 1) & hash]) != null) {
        //桶为树形结构
        if (first instanceof TreeNode)
            old = (t = (TreeNode<K,V>)first).getTreeNode(hash, key);
        //链表结构
        else {
            Node<K,V> e = first; K k;
            //找到对应节点
            do {
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k)))) {
                    old = e;
                    break;
                }
                ++binCount;
            } while ((e = e.next) != null);
        }
    }
    //old不为空，进行merge操作
    if (old != null) {
        V v;
        if (old.value != null)
            v = remappingFunction.apply(old.value, value);
        else
            v = value;
        if (v != null) {
            old.value = v;
            afterNodeAccess(old);
        }
        //merge后的v为空，则将该节点删除
        else
            removeNode(hash, key, null, false, true);
        return v;
    }
    //old为空，即不存在key
    if (value != null) {
        //红黑树不为空
        if (t != null)
            t.putTreeVal(this, tab, hash, key, value);
        //红黑树为空
        else {
            //新建节点到first
            tab[i] = newNode(hash, key, value, first);
            //判断binCount，选择是否treeifyBin
            if (binCount >= TREEIFY_THRESHOLD - 1)
                treeifyBin(tab, hash);
        }
        ++modCount;
        ++size;
        afterNodeInsertion(true);
    }
    return value;
}
```



### *treeifyBin(Node<K,V>[] tab, int hash)

```java
final void treeifyBin(Node<K,V>[] tab, int hash) {
    int n, index; Node<K,V> e;
    //若桶为空，或桶大小未达到树形化的条件，则resize
    if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY)
        resize();
    //达到树形化条件，判断当前桶是否为空，不为空，则开始树形化
    else if ((e = tab[index = (n - 1) & hash]) != null) {
        //hd为头，tl为尾
        TreeNode<K,V> hd = null, tl = null;
        //将Node链表转换为treeNode链表
        do {
            //把Node转换为TreeNode
            TreeNode<K,V> p = replacementTreeNode(e, null);
            if (tl == null)
                hd = p;
            else {
                p.prev = tl;
                tl.next = p;
            }
            tl = p;
        } while ((e = e.next) != null);
        if ((tab[index] = hd) != null)
            hd.treeify(tab);
    }
}
```



### *treeify(Node<K,V>[] tab)

```java
final void treeify(Node<K,V>[] tab) {
    //定义根节点
    TreeNode<K,V> root = null;
    //定义两个变量，x和next，x表示当前树节点
    for (TreeNode<K,V> x = this, next; x != null; x = next) {
        //next赋值为x.next
        next = (TreeNode<K,V>)x.next;
        //为x左右节点赋空值
        x.left = x.right = null;
        //root为空的特殊情况，表示x为根节点
        if (root == null) {
            x.parent = null;
            //根节点为黑色
            x.red = false;
            //root变成了一颗新的树，头节点为root
            root = x;
        }
        //以下情况为当前树节点不为根节点
        else {
            //缓存节点key
            K k = x.key;
            //缓存节点hash
            int h = x.hash;
            Class<?> kc = null;
            //非根节点的操作，只需要考虑把当前节点往root树放就可以了，因此遍历root节点
            for (TreeNode<K,V> p = root;;) {
                int dir, ph;
                //p节点key
                K pk = p.key;
                //p的hash比h大
                if ((ph = p.hash) > h)
                    dir = -1;
                //p的hash比h小
                else if (ph < h)
                    dir = 1;
                //两个hash一样大，则比较key
                else if ((kc == null &&
                          (kc = comparableClassFor(k)) == null) ||
                         (dir = compareComparables(kc, k, pk)) == 0)
                    //dir的取值和上面的hash对比一样，若dir=0，则当作-1
                    dir = tieBreakOrder(k, pk);
				//缓存p节点
                TreeNode<K,V> xp = p;
                //根据dir判断插入节点为左还是右，需要插入的节点只有在满足当前遍历的节点左子树或右子树不为空
                if ((p = (dir <= 0) ? p.left : p.right) == null) {
                    x.parent = xp;
                    if (dir <= 0)
                        xp.left = x;
                    else
                        xp.right = x;
                    //树平衡操作
                    root = balanceInsertion(root, x);
                    break;
                }
            }
        }
    }
    moveRootToFront(tab, root);
}
```



## 四、Question

### 1.data是什么？

```
Map map = new HashMap();
Object data = map.put(1, 1);
System.out.println(data);

data = map.put(1, 2);
System.out.println(data);

//null报错？
data = map.put(null, 1);
System.out.println(data);
```

### 2.为什么HashMap使用数组加链表方式

提高查询效率，减少扩容

### 3.为什么HashMap需要扩容

提高查询效率

是数据分布更均匀

减少数据碰撞

### 4.为什么1.7的头插法要优化成1.8的尾插法？

https://www.processon.com/view/link/5eb90cf50791290fe056f1ce

### 5.HashMap可以怎么优化？

initialCapacity&loadFactor

### 6.Map map = new HashMap(3,0.75f); 则HashMap中的阈值是多少

4

## 五、总结

HashMap的compute*相关方法未介绍，后面有机会会补上，上述的介绍主要涉及到了HashMap关键的数据结构以及相关扩容、插入、删除等操作，有一些比较理解较为简单的地方略过了没有讲，大概就是这个样子了。