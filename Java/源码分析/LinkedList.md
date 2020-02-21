# LinkedList源码分析

## 类信息

````java
public class LinkedList<E> extends AbstractSequentialList<E>
    implements List<E>, Deque<E>, Cloneable, java.io.Serializable{
......
}
````

LinkedList实现了List、Deque、Cloneable，表明该类具有列表、堆栈、双端队列、克隆等功能

## 属性

### Node

LinkedList内部由双链表实现，Node便是关键类

````java
private static class Node<E> {
        E item;
        Node<E> next;
        Node<E> prev;

        Node(Node<E> prev, E element, Node<E> next) {
            this.item = element;
            this.next = next;
            this.prev = prev;
        }
    }
````

每一个节点都有3个属性，一个是节点存储的数据item，一个是指向前一个节点的引用，一个是指向后一个节点的引用

### DescendingIterator

通过ListItr.previous实现的适配器

### LLSpliterator

LLSpliterator是JDK1.8之后LinkedList新增的内部类，大概用途是将元素分割成多份，分别交于不于的线程去遍历，以提高效率。

### size

列表的大小

### first

列表的第一个元素

### last

列表最后一个元素

## 构造方法

### LinkedList()

方法体为空，仅为了构造一个空的LinkedList

### LinkedList(Collection<? extends E> c)

传入一个集合，将元素加入到当前LinkedList对象中

## 方法

对双向链表的操作主要为对各个节点的Node引用的修改以及节点判空问题，下面列举几个涉及队列和堆栈的操作

### peek()

返回链表中的第一个节点的item

### poll()

从正向队列中返回第一个节点并移除链表中的该节点

````java
public E poll() {
        final Node<E> f = first;
        return (f == null) ? null : unlinkFirst(f);
}
````

### unlinkFirst()

````java
private E unlinkFirst(Node<E> f) {
        // assert f == first && f != null;
        final E element = f.item;
        final Node<E> next = f.next;
        f.item = null;
        f.next = null; // help GC
        first = next;
        if (next == null)
            last = null;
        else
            next.prev = null;
        size--;
        modCount++;
        return element;
    }
````

主要细节在于将first的item和节点引用置空，对next节点进行非空判断，并更新引用。

### unlinkLast()

功能与unlinkFirst相近

### remove()

会调用removeFirst，removeFirst功能与poll相似，区别在于poll方法在判断first为空时不会抛出异常。

### pollFirst()

与poll完全相同，这里的pollFirst是针对双向队列概念

### pollLast()

使最后一个元素出列，调用了unlinkLast

### push()

调用了addFirst方法，在first前插入一个节点

### pop()

调用removeFirst方法，使第一个元素出栈

### listIterator(int index)

返回一个ListItr对象

### writeObject(java.io.ObjectOutputStream s)

````java
private void writeObject(java.io.ObjectOutputStream s)
        throws java.io.IOException {
        // Write out any hidden serialization magic
        s.defaultWriteObject();

        // Write out size
        s.writeInt(size);

        // Write out all elements in the proper order.
        for (Node<E> x = first; x != null; x = x.next)
            s.writeObject(x.item);
    }
````

序列化对象，由于size、first、node是transient的，所谓size和所有item需要额外的write操作

### readObject(java.io.ObjectInputStream s)

````java
private void readObject(java.io.ObjectInputStream s)
        throws java.io.IOException, ClassNotFoundException {
        // Read in any hidden serialization magic
        s.defaultReadObject();

        // Read in size
        int size = s.readInt();

        // Read in all elements in the proper order.
        for (int i = 0; i < size; i++)
            linkLast((E)s.readObject());
    }
````

按照序列化的顺序进行反序列化，在读取item时，通过linkLast一个个加入到linkedList对象的最后一个元素上