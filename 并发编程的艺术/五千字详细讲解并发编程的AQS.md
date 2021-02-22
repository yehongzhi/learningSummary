# 思维导图

![](https://static.lovebilibili.com/aqs_swdt.jpg)

> **文章已收录Github精选，欢迎Star**：https://github.com/yehongzhi/learningSummary

# 一、什么是AQS

谈到并发编程，不得不说AQS(AbstractQueuedSynchronizer)，这可谓是Doug Lea老爷子的大作之一。AQS即是抽象队列同步器，是用来构建Lock锁和同步组件的基础框架，很多我们熟知的锁和同步组件都是基于AQS构建，比如ReentrantLock、ReentrantReadWriteLock、CountDownLatch、Semaphore。

实际上AQS是一个抽象类，我们不妨先看一下源码：

```java
public abstract class AbstractQueuedSynchronizer extends AbstractOwnableSynchronizer implements java.io.Serializable {
	//头结点
    private transient volatile Node head;
    //尾节点
    private transient volatile Node tail;
    //共享状态
    private volatile int state;
    
    //内部类，构建链表的Node节点
	static final class Node {
        volatile Node prev;
        volatile Node next;
        volatile Thread thread;
    }
}
//AbstractQueuedSynchronizer的父类
public abstract class AbstractOwnableSynchronizer implements java.io.Serializable {
    //占用锁的线程
    private transient Thread exclusiveOwnerThread;
}
```

由源码可以看出AQS是有以下几个部分组成的：

![](https://static.lovebilibili.com/aqs_01.png)

## 1.1 state共享变量

AQS中里一个很重要的字段state，表示同步状态，是由`volatile`修饰的，用于展示当前临界资源的获锁情况。通过getState()，setState()，compareAndSetState()三个方法进行维护。

```java
private volatile int state;

protected final int getState() {
    return state;
}
protected final void setState(int newState) {
    state = newState;
}
//CAS操作
protected final boolean compareAndSetState(int expect, int update) {
    return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
}
```

关于state的几个要点：

- 使用volatile修饰，保证多线程间的可见性。
- getState()、setState()、compareAndSetState()使用final修饰，限制子类不能对其重写。
- compareAndSetState()采用乐观锁思想的CAS算法，保证原子性操作。

## 1.2 CLH队列

AQS里另一个重要的概念就是CLH队列，它是一个双向链表队列，其内部由head和tail分别记录头结点和尾结点，队列的元素类型是Node。

简单讲一下这个队列的作用，就是当一个线程获取同步状态(state)失败时，AQS会将此线程以及等待的状态等信息封装成Node加入到队列中，同时阻塞该线程，等待后续的被唤醒。

队列的元素就是一个个的Node节点，下面讲一下Node节点的组成：

```java
static final class Node {
	//共享模式下的等待标记
    static final Node SHARED = new Node();
	//独占模式下的等待标记
    static final Node EXCLUSIVE = null;
    //表示当前节点的线程因为超时或者中断被取消
    static final int CANCELLED =  1;
	//表示当前节点的后续节点的线程需要运行，也就是通过unpark操作
    static final int SIGNAL    = -1;
	//表示当前节点在condition队列中
    static final int CONDITION = -2;
	//共享模式下起作用，表示后续的节点会传播唤醒的操作
    static final int PROPAGATE = -3;
	//状态，包括上面的四种状态值，初始值为0，一般是节点的初始状态
    volatile int waitStatus;
	//上一个节点的引用
    volatile Node prev;
	//下一个节点的引用
    volatile Node next;
	//保存在当前节点的线程引用
    volatile Thread thread;
	//condition队列的后续节点
    Node nextWaiter;
}
```

## 1.3 exclusiveOwnerThread

AQS通过继承AbstractOwnableSynchronizer类，拥有的属性。表示独占模式下同步器的持有者。

# 二、AQS的实现原理

AQS有两种模式，分别是独占式和共享式。

## 2.1 独占式

同一时刻仅有一个线程持有同步状态，也就是其他线程只有在占有的线程释放后才能竞争，比如**ReentrantLock**。下面从源码切入，梳理独占式的实现思路。

首先看acquire()方法，这是AQS在独占模式下获取同步状态的方法。

```java
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```

先讲这个方法的总体思路：

- tryAcquire()尝试直接去获取资源，如果成功则直接返回。
- 如果失败则调用addWaiter()方法把当前线程包装成Node(状态为EXCLUSIVE，标记为独占模式)插入到CLH队列末尾。
- 然后acquireQueued()方法使线程阻塞在等待队列中获取资源，一直获取到资源后才返回，如果在整个等待过程中被中断过，则返回true，否则返回false。
- 线程在等待过程中被中断过，它是不响应的。只是获取资源后才再进行自我中断selfInterrupt()，将中断补上。

我们展开来分析，看tryAcquire()方法，尝试获取资源，成功返回true，失败返回false。

```java
//直接抛出异常，这是由子类进行实现的方法，体现了模板模式的思想
protected boolean tryAcquire(int arg) {
    throw new UnsupportedOperationException();
}
```

为什么没有具体实现呢，其实这是模板模式的思想。这个方法是尝试获取资源，但是获取资源的方式有很多种实现，比如**公平锁有公平锁的获取方式，非公平锁有非公平锁的获取方式**(后面会讲，别急)。所以这里是一个没有具体实现的方法，需要由子类去实现。

接着看addWaiter()方法，这个方法的作用是把当前线程包装成Node添加到队列中。

```java
private Node addWaiter(Node mode) {
    //把当前线程包装成Node节点
    Node node = new Node(Thread.currentThread(), mode);
    //获取到尾结点
    Node pred = tail;
    //判断尾结点是否为null，如果不为空，那就证明队列已经初始化了
    if (pred != null) {
        //已经初始化了，就直接把Node节点添加到队列的末尾
        node.prev = pred;
        if (compareAndSetTail(pred, node)) {
            pred.next = node;
            //返回包含当前线程的节点Node
            return node;
        }
    }
    //如果队列没有初始化，那就调用enq()方法
    enq(node);
    return node;
}
```

接着我们看enq()方法，就是一个自旋的操作，把传进来的node添加到队列最后，如果队列没有初始化则进行初始化。

```java
private Node enq(final Node node) {
    //自旋操作，也就是死循环，只有加入队列成功才会return
    for (;;) {
        //把尾结点赋值给t
        Node t = tail;
        //如果为空，证明没有初始化，进行初始化
        if (t == null) { // Must initialize
            //创建一个空的Node节点，并且设置为头结点
            if (compareAndSetHead(new Node()))
                //然后把头结点赋值给尾结点
                tail = head;
        } else {
            //如果是第一次循环为空，就已经创建了一个一个Node，那么第二次循环就不会为空了
            //如果尾结点不为空，就把传进来的node节点的前驱节点指向尾结点
            node.prev = t;
            //cas原子性操作，把传进来的node节点设置为尾结点
            if (compareAndSetTail(t, node)) {
                //把原来的尾结点的后驱节点指向传进来的node节点
                t.next = node;
                return t;
            }
        }
    }
}
```

接着我们再把思路跳回去顶层的方法，看acquireQueued()方法。

```java
//在队列中的节点node通过acquireQueued()方法获取资源，忽略中断。
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        //自旋的操作，一个死循环
        for (;;) {
            //获取传进来的node节点的前驱节点，赋值给p
            final Node p = node.predecessor();
            //如果p是头结点，node节点就是第二个节点，则再次去尝试获取资源
            if (p == head && tryAcquire(arg)) {
                //tryAcquire(arg)获取资源成功的话，则把node节点设置为头结点
                setHead(node);
                //把原来的头结点p的后驱节点设置为null，等待GC垃圾回收
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }
            //如果p不是头结点，或者tryAcquire()获取资源失败，判断是否可以被park，也就是把线程阻塞起来
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())//&&前面如果返回true，将当前线程阻塞并检查是否被中断
                //如果阻塞过程中被中断，则置interrupted标志位为true。
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

最后是selfInterrupt()方法，自我中断。

```java
static void selfInterrupt() {
    Thread.currentThread().interrupt();
}
```

过程记不住没关系，下面画张图来总结一下，其实很简单。

![](https://static.lovebilibili.com/aqs_02.jpg)

## 2.2 共享式

即共享资源可以被多个线程同时占有，直到共享资源被占用完毕。比如ReadWriteLock和CountdownLatch。下面我们从源码去分析其实现原理。

首先还是看最顶层的acquireShared()方法。

```java
public final void acquireShared(int arg) {
    if (tryAcquireShared(arg) < 0)
        doAcquireShared(arg);
}
```

这段代码很简单，首先调用tryAcquireShared()方法，tryAcquireShared返回是一个int数值，当返回值大于等于0的时候，说明获得成功获取锁，方法结束，否则返回负数，表示获取同步状态失败，执行doAcquireShared方法。

tryAcquireShared()方法是一个模板方法由子类去重写，意思是需要如何获取同步资源由实现类去定义，AQS只是一个框架。

那么就看如果获取资源失败，执行的doAcquireShared()方法。

```java
private void doAcquireShared(int arg) {
    //调用addWaiter()方法，把当前线程包装成Node，标志为共享式，插入到队列中
    final Node node = addWaiter(Node.SHARED);
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            //获取当前节点node的前驱节点
            final Node p = node.predecessor();
            //前驱节点是否是头结点
            if (p == head) {
                //如果前驱节点是头结点，则调用tryAcquireShared()获取同步资源
                int r = tryAcquireShared(arg);
                //r>=0表示获取同步资源成功，只有获取成功，才会执行到return退出for循环
                if (r >= 0) {
                    //设置node为头结点
                    setHeadAndPropagate(node, r);
                    p.next = null; // help GC
                    if (interrupted)
                        selfInterrupt();
                    failed = false;
                    return;
                }
            }
            //判断是否可以被park，跟独占式的逻辑一样返回true，则进行park操作，阻塞线程
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

这段逻辑基本上跟独占式的逻辑差不多，不同的地方在于入队的Node是标志为SHARED共享式的，获取同步资源的方式是tryAcquireShared()方法。

# 三、AQS的模板模式

模板模式在AQS中的应用可谓是一大精髓，在上文中有提到的tryAcquireShared()和tryAcquire()都是很重要的模板方法。一般使用AQS往往都是使用一个内部类继承AQS，然后重写相应的模板方法。

AQS已经把一些常用的，比如入队，出队，CAS操作等等构建了一个框架，使用者只需要实现获取资源，释放资源的，因为很多锁，还有同步器，其实就是获取资源和释放资源的方式有比较大的区别。

那么我们看一下模板方法有哪些。

## 3.1 tryAcquire()

tryAcquire()方法，独占式获取同步资源，返回true表示获取同步资源成功，false表示获取失败。

```java
protected boolean tryAcquire(int arg) {
    throw new UnsupportedOperationException();
}
```

## 3.2 tryRelease()

tryRelease()方法，独占式使用，tryRelease()的返回值来判断该线程是否已经完成释放资源，子类来决定是否能成功释放锁。

```java
protected boolean tryRelease(int arg) {
    throw new UnsupportedOperationException();
}
```

## 3.3 tryAcquireShared()

tryAcquireShared()方法，共享式获取同步资源，返回大于等于0表示获取资源成功，返回小于0表示失败。

```java&#39;
protected int tryAcquireShared(int arg) {
    throw new UnsupportedOperationException();
}
```

## 3.4 tryReleaseShared()

tryReleaseShared()方法，共享式尝试释放资源，如果释放后允许唤醒后续等待结点返回true，否则返回false。

```java
protected boolean tryReleaseShared(int arg) {
	throw new UnsupportedOperationException();
}
```

## 3.5 isHeldExclusively()

isHeldExclusively()方法，该线程是否正在独占资源。只有用到condition才需要去实现它。

```java
protected boolean isHeldExclusively() {
    throw new UnsupportedOperationException();
}
```

# 四、认识ReentrantLock

ReentrantLock是一个很经典的使用AQS的案例，不妨以此为切入点来继续深入。ReentrantLock的特性有很多，首先它是一个悲观锁，其次有两种模式分别是公平锁和非公平锁，最后它是重入锁，也就是能够对共享资源重复加锁。

AQS通常是使用内部类实现，所以不难想象在ReentrantLock类里有两个内部类，我们看一张类图。

![](https://static.lovebilibili.com/aqs_03.png)

FairSync是公平锁的实现，NonfairSync则是非公平锁的实现。通过构造器传入的boolean值进行判断。

```java
public ReentrantLock(boolean fair) {
    //true则使用公平锁，false则使用非公平锁
    sync = fair ? new FairSync() : new NonfairSync();
}
//默认是非公平锁
public ReentrantLock() {
    sync = new NonfairSync();
}
```

公平锁是遵循**FIFO**（先进先出）原则的，先到的线程会优先获取资源，后到的线程会进行排队等待，能保证每个线程都能拿到锁，不会存在有线程饿死的情况。

非公平锁是则不遵守先进先出的原则，会出现有线程插队的情况，不能保证每个线程都能拿到锁，会存在有线程饿死的情况。

下面我们从源码分析去找出这两种锁的区别。

# 五、源码分析ReentrantLock

## 5.1 上锁

ReentrantLock是通过lock()方法上锁，所以看lock()方法。

```java
public void lock() {
    sync.lock();
}
```

sync就是NonfairSync或者FairSync。

```java
//这里就是调用AQS的acquire()方法，获取同步资源
final void lock() {
    acquire(1);
}
```

acquire()方法前面已经解析过了，主要看FairSync的tryAcquire()方法。

```java
protected final boolean tryAcquire(int acquires) {
    //获取当前线程
    final Thread current = Thread.currentThread();
    //获取同步状态
    int c = getState();
    //判断同步状态是否为0
    if (c == 0) {
        //关键在这里，公平锁会判断是否需要排队
        if (!hasQueuedPredecessors() &&
            //如果不需要排队，则直接cas操作更新同步状态为1
            compareAndSetState(0, acquires)) {
            //设置占用锁的线程为当前线程
            setExclusiveOwnerThread(current);
            //返回true，表示上锁成功
            return true;
        }
    }
    //判断当前线程是否是拥有锁的线程，主要是可重入锁的逻辑
    else if (current == getExclusiveOwnerThread()) {
        //如果是当前线程，则同步状态+1
        int nextc = c + acquires;
        if (nextc < 0)
            throw new Error("Maximum lock count exceeded");
        //设置同步状态
        setState(nextc);
        return true;
    }
    //以上情况都不是，则返回false，表示上锁失败。上锁失败根据AQS的框架设计，会入队排队
    return false;
}
```

如果是非公平锁NonfairSync的tryAcquire()，我们继续分析。

```java
protected final boolean tryAcquire(int acquires) {
    return nonfairTryAcquire(acquires);
}
//非公平式获取锁
final boolean nonfairTryAcquire(int acquires) {
    //这段跟公平锁是一样的操作
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        //关键在这里，不再判断是否需要排队，而是直接去更新同步状态，通俗点讲就是插队
        if (compareAndSetState(0, acquires)) {
            //如果获取同步状态成功，则设置占用锁的线程为当前线程
            setExclusiveOwnerThread(current);
            //返回true表示获取锁成功
            return true;
        }
    }
    //以下逻辑跟公平锁的逻辑一样
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

其实很明显了，关键的区别就在于尝试获取锁的时候，公平锁会判断是否需要排队再去更新同步状态，非公平锁是直接就更新同步，不判断是否需要排队。

从性能上来说，公平锁的性能是比非公平锁要差的，因为**公平锁要遵守FIFO(先进先出)的原则，这就会增加了上下文切换与等待线程的状态变换时间**。

非公平锁的缺点也是很明显的，因为允许插队，这就会存在有线程饿死的情况。

## 5.2 解锁

解锁对应的方法就是unlock()。

```java
public void unlock() {
    //调用AQS中的release()方法
    sync.release(1);
}
//这是AQS框架定义的release()方法
public final boolean release(int arg) {
    //当前锁是不是没有被线程持有,返回true表示该锁没有被任何线程持有
    if (tryRelease(arg)) {
        //获取头结点h
        Node h = head;
        //判断头结点是否为null并且waitStatus不是初始化节点状态，解除线程挂起状态
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    return false;
}
```

关键在于tryRelease()，这就不需要分公平锁和非公平锁的情况，只需要考虑可重入的逻辑。

```java
protected final boolean tryRelease(int releases) {
    //减少可重入的次数
    int c = getState() - releases;
    //如果当前线程不是持有锁的线程，抛出异常
    if (Thread.currentThread() != getExclusiveOwnerThread())
        throw new IllegalMonitorStateException();
    boolean free = false;
    // 如果持有线程全部释放，将当前独占锁所有线程设置为null，并更新state
    if (c == 0) {
        //状态为0，表示持有线程被全部释放，设置为true
        free = true;
        setExclusiveOwnerThread(null);
    }
    setState(c);
    return free;
}
```

# 总结

JUC可谓是学习java的一个难点，而学习AQS其实关键在于并发的思维，因为需要考虑的情况很多，其次需要理解模板模式的思想，这才能理解为什么AQS作为一个框架的作用。ReentrantLock这个类我觉得是理解AQS一个很好的切入点，看懂了之后再去看AQS的其他应用类应该会轻松很多。

那么这篇文章就讲到这里了，希望看完能有所收获，感谢你的阅读。

**觉得有用就点个赞吧，你的点赞是我创作的最大动力**~

**我是一个努力让大家记住的程序员。我们下期再见！！！**

![](https://static.lovebilibili.com/dashacha.png)

> 能力有限，如果有什么错误或者不当之处，请大家批评指正，一起学习交流！