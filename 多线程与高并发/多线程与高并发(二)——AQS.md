# 多线程与高并发(二)——AQS

## 源码阅读规则

跑不起来不读

解决问题就好(带有目的性)

一条线索到底

无关细节略过

一般不读静态

一般动态读法



## AQS(CLH)

AbstractQueuedSynchronizer类，所有锁的核心。在研究源码之前，先了解AQS的数据结构,之前的笔记中曾经讲过，公平锁与非公平锁区别在于，线程在抢占锁时是否会进入等待队列，这里的等待队列在AQS实现

```java
	/**
     * Head of the wait queue, lazily initialized.  Except for
     * initialization, it is modified only via method setHead.  Note:
     * If head exists, its waitStatus is guaranteed not to be
     * CANCELLED.
     */
    private transient volatile Node head;

    /**
     * Tail of the wait queue, lazily initialized.  Modified only via
     * method enq to add new wait node.
     */
    private transient volatile Node tail;

    /**
     * The synchronization state.
     */
    private volatile int state;
```

上图——

![img](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/aqs.jpg)

每一个Node节点即代表一个线程，head和tail表示头节点和尾节点，state在不同的锁实现中有不同的值，用来表示线程持有锁的状态值。简单了解了AQS的数据结构，那么来看一下代码把，从ReentantLock开始分析。



## ReentrantLock

关注ReentrantLock的lock方法，发现调用了sync. Lock()，sync是**ReentrantLock**中的内部类**Sync**的实例，而Sync的lock方法是abstract的，考虑到是模板方法。通过debug模式下对lock的测试，发现Sync类lock方法的默认实现是在**NonfairSync**类中，该类也是ReentrantLock的一个内部类，代码如下：

```java
static final class NonfairSync extends Sync {
        private static final long serialVersionUID = 7316153563782823691L;

        /**
         * Performs lock.  Try immediate barge, backing up to normal
         * acquire on failure.
         */
        final void lock() {
            if (compareAndSetState(0, 1))
                setExclusiveOwnerThread(Thread.currentThread());
            else
                acquire(1);
        }
```

不难发现，NonfairSync的lock方法中，通过了compareAndSetState方法，也就是CAS操作来获取锁，CAS操作一般由unsafe来执行操作，由于unsafe的实现为本地方法，通过C来实现，所以不讨论，这里暂时了解到CAS这一步。回过头到上面if/else语句，若获取锁，则判定当前线程获得锁。若没有获取到锁，则调用acquire方法，接下来主要讨论acquire做了什么。



## acquire

```java
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```

acquire方法位于AbstractQueuedSynchronizer中，点进tryAcquire()



## tryAquire

```java
protected boolean tryAcquire(int arg) {
    throw new UnsupportedOperationException();
}
```

又是一个模板方法，看来又是debug的时候了，公平与非公平锁在这个时候就需要区分了，第一次先使用非公平锁：

#### 非公平锁

**ReentrantLock lock = new ReentrantLock();**

在lock.lock方法处进行debug，进入tryAcquire后，进入到了NonfairSync的tryAcquire()中

```java
protected final boolean tryAcquire(int acquires) {
            return nonfairTryAcquire(acquires);
}
```

再查看nonfairTryAcquire(acquires)

```java
final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        if (compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0) // overflow
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```

不难发现，当getState()=0时，表示当前没有线程占用锁，因此调用前面讲到过的compareAndSetState方法，若锁已被占用，且占用者为当前线程自身，则更新state。若state!=0且当前线程不持有锁，则说明获取锁失败。

### 公平锁

换成**ReentrantLock lock = new ReentrantLock();**再试试，发现进入了**FairSync类**的tryAquire方法，FairSync也是ReentrantLock的内部类

```java
protected final boolean tryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        if (!hasQueuedPredecessors() &&
            compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0)
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```

可以看到公平锁与非公平锁的区别在于当state为0时，当前线程将有机会获取锁，那么如何获取公平锁呢？可以看到代码里使用了hasQueuedPredecessors方法，意味队列中是否有先于当前线程的其他线程存在，代码如下：

```java
public final boolean hasQueuedPredecessors() {
    // The correctness of this depends on head being initialized
    // before tail and on head.next being accurate if the current
    // thread is first in queue.
    Node t = tail; // Read fields in reverse initialization order
    Node h = head;
    Node s;
    return h != t &&
        ((s = h.next) == null || s.thread != Thread.currentThread());
}
```

判断首节点是否不等于尾节点，若成立，则判断首节点的下一个节点是否为空，若不为空，则判断首节点的下一个节点是否为当前线程，若不是当前线程，则返回true。

h！=t不成立，即h==t，主要存在两种情况：1. 队列为空(比较好理解)；2. 队列中存在一个节点(首节点），首节点是获取到锁的节点，由于state==0，首节点应处于持有锁和释放锁的中间态。

那么回到公平锁的tryAcquire方法中——

```java
if (!hasQueuedPredecessors() &&
    compareAndSetState(0, acquires)) {
    setExclusiveOwnerThread(current);
    return true;
}
```

若队列中不存在其他线程，则当前线程就会尝试获取锁。



## 再回到acquire

回头看看刚才的acquire方法

```java
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```

讨论了tryAcquire()方法用来获取锁，那么获取失败了，就要调用**acquireQueued**进入队列中排队等待获取锁，上述代码中的意思为将当前线程作为排他(与共享对立)形式的节点插入队列，首先看一下**addWaiter**在**java8**中是如何实现的



## addWaiter

```java
/**
     * Creates and enqueues node for current thread and given mode.
     *
     * @param mode Node.EXCLUSIVE for exclusive, Node.SHARED for shared
     * @return the new node
     */
private Node addWaiter(Node mode) {
    Node node = new Node(Thread.currentThread(), mode);
    // Try the fast path of enq; backup to full enq on failure
    Node pred = tail;
    if (pred != null) {
        node.prev = pred;
        if (compareAndSetTail(pred, node)) {
            pred.next = node;
            return node;
        }
    }
    enq(node);
    return node;
}
```

通过CAS操作将当前线程作为节点插入到AQS的队列末尾中，看一下addWaiter在**java11**中的实现——

```java
private AbstractQueuedSynchronizer.Node addWaiter(AbstractQueuedSynchronizer.Node mode) {
    AbstractQueuedSynchronizer.Node node = new AbstractQueuedSynchronizer.Node(mode);

    AbstractQueuedSynchronizer.Node oldTail;
    do {
        while(true) {
            oldTail = this.tail;
            if (oldTail != null) {
                node.setPrevRelaxed(oldTail);
                break;
            }

            this.initializeSyncQueue();
        }
    } while(!this.compareAndSetTail(oldTail, node));

    oldTail.next = node;
    return node;
}
```

可以看到实现原理基本相同，但是jdk11中使用了do-while循环来使线程不断尝试获取锁，且node.setPrevRelaxed(oldTail)方法中使用了**VarHandle**类型来实现，这是在java9之后才出新的新类，用于指向某个对象的引用，即指向一个对象的内存，可以不通过对象的引用而通过varHandle实例进行原子性操作(CAS操作)



## 再再回到acquire

```java
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```

addWaiter将当前线程的节点加入到队列中并发回给acquireQueued方法作为参数，现在进入acquireQueued瞧瞧



## acquireQueued

```java
final boolean acquireQueued(AbstractQueuedSynchronizer.Node node, int arg) {
    boolean interrupted = false;

    try {
        while(true) {
            AbstractQueuedSynchronizer.Node p = node.predecessor();
            if (p == this.head && this.tryAcquire(arg)) {
                this.setHead(node);
                p.next = null;
                return interrupted;
            }

            if (shouldParkAfterFailedAcquire(p, node)) {
                interrupted |= this.parkAndCheckInterrupt();
            }
        }
    } catch (Throwable var5) {
        this.cancelAcquire(node);
        if (interrupted) {
            selfInterrupt();
        }

        throw var5;
    }
}
```

当前线程节点进入队列后，则排队等待获取锁，即代码中的while循环。由于AQS的等待队列为双向的链表，因此获取到当前节点的前一个结点p，若p为头节点，则tryAcquire，获取锁成功，那么将头节点置为当前线程节点，并将p.next置空帮助gc。若p为头节点但是获取锁失败，说明头节点还未释放锁，则在获取失败后重新等待(park)。需要注意acquireQueued的返回值类型为boolean，意思是"是否被中断"，默认为false。若acquireQueued方法catch到中断，则会调用**selfInterrupt**()，或者在**parkAndCheckInterrupt**方法中发现中断从而返回给**acquire**方法执行**selfInterrupt**()。



## 总结

历尽千辛万苦，线程终于拿到了锁，并且没有被打断(interrupted)，可以开始执行自己的任务了，以下省略一万行任务代码………获取锁之后依然还有很多操作都会和AQS有关，由于篇幅和时间关系之后应该会额外写一篇文章来记录。

另外，虽然文章主要是写给自己看的，但可能也有读者把，对于像我这样的小白，建议跟着debug自己调试以下，干啃上面这些东西我想效果不会太好。同时若有描述不当或错误的地方也希望给予指正。