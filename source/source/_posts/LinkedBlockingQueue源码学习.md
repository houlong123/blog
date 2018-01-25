---
title: LinkedBlockingQueue源码学习
date: 2017-11-21 15:01:44
tags: java并发集合
---

#### 前言
在上篇文章 [ArrayBlockingQueue源码学习](https://houlong123.github.io/2017/11/16/ArrayBlockingQueue源码学习/) 中，已对底层基于数组的ArrayBlockingQueue阻塞队列进行了解析。本文将讲解阻塞队列的另外一个重要成员：`LinkedBlockingQueue`。<font color=red>LinkedBlockingQueue的底层是基于链表的阻塞队列。</font>
    
它既可以充当有界队列，也可以充当无界队列。在手动设置了队列的大小时，它即为有界队列；在没有设置队列大小时，默认大小为`Integer.MAX_VALUE`，可以冲当成无界队列。

固定大小线程池底层所使用的阻塞队列就是LinkedBlockingQueue队列。


```java
public static ExecutorService newFixedThreadPool(int nThreads) {
    return new ThreadPoolExecutor(nThreads, nThreads,
                                  0L, TimeUnit.MILLISECONDS,
                                  new LinkedBlockingQueue<Runnable>());
}
```


#### 源码解析

<!-- more -->

##### 内部数据结构

```java
public class LinkedBlockingQueue<E> extends AbstractQueue<E>
        implements BlockingQueue<E>, java.io.Serializable {
        
    //链接列表节点类。所有的元素都通过Node这个静态内部类来进行存储，这与LinkedList的处理方式完全一样
    static class Node<E> {
        //使用item来保存元素本身
        E item;

        /**
         * One of:
         * - the real successor Node
         * - this Node, meaning the successor is head.next
         * - null, meaning there is no successor (this is the last node)
         */
        //保存当前节点的后继节点
        Node<E> next;

        Node(E x) { item = x; }
    }
    
    /**
     阻塞队列所能存储的最大容量
     用户可以在创建时手动指定最大容量,如果用户没有指定最大容量
     那么最默认的最大容量为Integer.MAX_VALUE.
     */
    private final int capacity;
    
    /**
     当前阻塞队列中的元素数量
     PS：如果了解过ArrayBlockingQueue的源码,你会发现
     ArrayBlockingQueue底层保存元素数量使用的是一个
     普通的int类型变量。其原因是在ArrayBlockingQueue底层
     对于元素的入队列和出队列使用的是同一个lock对象。而数
     量的修改都是在处于线程获取锁的情况下进行操作，因此不
     会有线程安全问题。
     而LinkedBlockingQueue却不是，它的入队列和出队列使用的是两个
     不同的lock对象,因此无论是在入队列还是出队列，都会涉及对元素数
     量的并发修改，因此这里使用了一个原子操作类
     来解决对同一个变量进行并发修改的线程安全问题。
     */
    private final AtomicInteger count = new AtomicInteger();
    
    //链接列表头结点。不变性：头部的元素总是null，即head.item == null
    transient Node<E> head;
    //链接列表尾节点。不变性：尾部的next总是null，即tail.next == null
    private transient Node<E> last;
     /** Lock held by take, poll, etc */
    private final ReentrantLock takeLock = new ReentrantLock();

    /** Wait queue for waiting takes */
    //    当队列为空时，通过该Condition让从队列中获取元素的线程处于等待状态
    private final Condition notEmpty = takeLock.newCondition();

    /** Lock held by put, offer, etc */
    private final ReentrantLock putLock = new ReentrantLock();

    /** Wait queue for waiting puts */
    //当队列的元素已经达到capactiy，通过该Condition让元素入队列的线程处于等待状态
    private final Condition notFull = putLock.newCondition();

}
```
由源码可知，LinkedBlockingQueue在入队列和出队列时使用的不是同一个Lock，这意味着它们之间的操作不会存在互斥。在多个CPU的情况下，它们可以做到真正的在同一时刻既消费、又生产，能够做到并行处理。内部有一个节点类，用来存储入队列的元素。

##### 构造函数

```java
public LinkedBlockingQueue() {
    this(Integer.MAX_VALUE);    //显示指定capacity的值，默认使用int的最大值
}

public LinkedBlockingQueue(int capacity) {
    if (capacity <= 0) throw new IllegalArgumentException();    //参数校验
    this.capacity = capacity;   //初始化链表大小
    last = head = new Node<E>(null);    //初始化链表头尾节点，指向一个dummy节点
}

public LinkedBlockingQueue(Collection<? extends E> c) {
    this(Integer.MAX_VALUE);    //调用重载构造函数，进行基础属性的初始化
    final ReentrantLock putLock = this.putLock; //获取锁
    putLock.lock(); // Never contended, but necessary for visibility
    try {
        int n = 0;
        for (E e : c) {
            if (e == null)
                throw new NullPointerException();
            if (n == capacity)
                throw new IllegalStateException("Queue full");
            enqueue(new Node<E>(e));
            ++n;
        }
        count.set(n);
    } finally {
        putLock.unlock();
    }
}
```
从源码中可知，LinkedBlockingQueue类的构造函数与ArrayBlockingQueue类的构造函数十分相似。都有三个构造函数，其中都是基于`LinkedBlockingQueue(int capacity)` 用于设置阻塞队列的相关属性。该构造函数初始化了队列的大小与头尾节点。

##### put源码

```java
public void put(E e) throws InterruptedException {
    if (e == null) throw new NullPointerException();    //待插入元素为空判断，为空， 抛异常
    // Note: convention in all put/take/etc is to preset local var
    // holding count negative to indicate failure unless set.
    int c = -1;
    Node<E> node = new Node<E>(e);  //将待插入元素组装成节点
    final ReentrantLock putLock = this.putLock; //获取锁
    final AtomicInteger count = this.count; //获取元素个数
    /*
        执行可中断的锁获取操作,即意味着如果线程由于获取
        锁而处于Blocked状态时，线程是可以被中断而不再继
        续等待，这也是一种避免死锁的一种方式，不会因为
        发现到死锁之后而由于无法中断线程最终只能重启应用。
    */
    putLock.lockInterruptibly();
    try {
        /*
         * Note that count is used in wait guard even though it is
         * not protected by lock. This works because count can
         * only decrease at this point (all other puts are shut
         * out by lock), and we (or some other waiting put) are
         * signalled if it ever changes from capacity. Similarly
         * for all other uses of count in other wait guards.
         */
        //如果链表已满，等待。使用while判断依旧是为了放置线程被"伪唤醒”而出现的情况,即当线程被唤醒时而队列的大小依旧等于capacity时，线程应该继续等待。
        while (count.get() == capacity) {
            notFull.await();
        }
        enqueue(node);  //入链表
        c = count.getAndIncrement();    //获取链表当前数量
        if (c + 1 < capacity)      //链表没有满，唤醒通知。c+1得到的结果是新元素入队列之后队列元素的总和。
            notFull.signal();
    } finally {
        putLock.unlock();
    }
    /*
    当c=0时，即意味着之前的队列是空队列,出队列的线程都处于等待状态，
    现在新添加了一个新的元素,即队列不再为空,因此它会唤醒正在等待获取元素的线程。
    */
    if (c == 0) //c == 0，表明是第一次往链表中添加数据
        signalNotEmpty();
}
```
代码逻辑：
1. 判断待插入元素是否为空，为空，抛异常；否则执行步骤2
2. 组装待插入节点，并获取目前队列的大小及中断锁。获取中断锁成功，执行步骤3；否则阻塞等待。
3. 如果队列已满，则等待；否则执行步骤4
4. 执行`enqueue()`方法，将节点入队列。
5. 增加队列大小，并判断队列是否已满，如果没有满，则 notFull 唤醒通知
6. 如果为第一次往队列中添加数据，则 notEmpty 唤醒通知。


##### enqueue源码

```java
//入链表
private void enqueue(Node<E> node) {
    // assert putLock.isHeldByCurrentThread();
    // assert last.next == null;
    last = last.next = node;
}
```
该方法功能很简单，就是改变链表的数据。

##### signalNotEmpty源码

```java
//唤醒正在等待获取元素的线程,告诉它们现在队列中有元素
private void signalNotEmpty() {
    final ReentrantLock takeLock = this.takeLock;
    takeLock.lock();
    try {
        notEmpty.signal();
    } finally {
        takeLock.unlock();
    }
}
```

##### offer源码

```java
public boolean offer(E e) {
    if (e == null) throw new NullPointerException();    //元素为空判断， 为空，抛异常
    final AtomicInteger count = this.count;
    if (count.get() == capacity)    //链表已满，返回FALSE
        return false;
    int c = -1;
    Node<E> node = new Node<E>(e);  //组装节点
    final ReentrantLock putLock = this.putLock;
    putLock.lock(); //获取锁
    try {
        /*
        当获取到锁时，需要进行二次的检查,因为可能当队列的大小为capacity-1时，
        两个线程同时去抢占锁，而只有一个线程抢占成功，那么此时
        当线程将元素入队列后，释放锁，后面的线程抢占锁之后，此时队列
        大小已经达到capacity，所以将它无法让元素入队列。
        */
        if (count.get() < capacity) {   //如果链表还未满，进行入链表操作
            enqueue(node);  //入链表
            c = count.getAndIncrement();    //增加元素总数，并返回之前的值
            if (c + 1 < capacity)   //在成功插入后，链表还没有满，则notFull唤醒通知
                notFull.signal();
        }
    } finally {
        putLock.unlock();
    }
    if (c == 0)
        signalNotEmpty();
    return c >= 0;
}
```
该方法与put方法的实现逻辑大体一致，只有部分细节不一样。offer方法有返回值，元素入队成功，返回true；入队失败，返回FALSE；put则为无限阻塞入队，直到入队成功返回。offer代码逻辑为：

1. 判断待插入元素是否为空，为空，抛异常；否则执行步骤2
2. 获取目前队列的大小，如果队列已满，则返回FALSE；否则执行步骤3
3. 组装待插入节点，并获取锁。获取锁成功，执行步骤4；否则阻塞等待。
4. 判断队列是否已满，已满，执行步骤7；否则执行步骤5
5. 执行enqueue()方法，将节点入队列。
6. 增加队列大小，并判断队列是否已满，如果没有满，则 notFull 唤醒通知。
7. 如果为第一次往队列中添加数据，则 notEmpty 唤醒通知。
8. 返回成功与否标识


##### 超时offer源码

```java
public boolean offer(E e, long timeout, TimeUnit unit)
    throws InterruptedException {

    if (e == null) throw new NullPointerException();
    long nanos = unit.toNanos(timeout);         //将指定的时间长度转换为毫秒来进行处理
    int c = -1;
    final ReentrantLock putLock = this.putLock;
    final AtomicInteger count = this.count;
    putLock.lockInterruptibly();
    try {
        while (count.get() == capacity) {   //在链表满的时候，进行超时等待状态
            if (nanos <= 0)
                return false;
            nanos = notFull.awaitNanos(nanos);
        }
        enqueue(new Node<E>(e));
        c = count.getAndIncrement();
        if (c + 1 < capacity)
            notFull.signal();
    } finally {
        putLock.unlock();
    }
    if (c == 0) //c == 0，表明是第一次往链表中添加数据
        signalNotEmpty();
    return true;
}
```
BlockingQueue还定义了一个限时等待插入操作，即在等待一定的时间内，如果队列有空间可以插入，那么就将元素入队列，然后返回true,如果在过完指定的时间后依旧没有空间可以插入，那么就返回false。超时offer的源码与put方法的源码不同之处就在于超时offer源码多了超时等待一步。


##### take源码

```java
public E take() throws InterruptedException {
    E x;
    int c = -1;
    final AtomicInteger count = this.count;
    final ReentrantLock takeLock = this.takeLock;
    takeLock.lockInterruptibly();   //通过takeLock获取锁，并且支持线程中断
    try {
        while (count.get() == 0) {  //如果链表为空，则阻塞等待
            notEmpty.await();
        }
        x = dequeue();  //获取头节点的item值
        c = count.getAndDecrement();    //将元素总数减一
        if (c > 1)  //如果take()后，元素总数不为0，则notEmpty唤醒通知，唤醒其他执行元素出队列的线程,让它们也可以执行元素的获取
            notEmpty.signal();
    } finally {
        takeLock.unlock();
    }
    if (c == capacity)  //链表不满时，notFull唤醒通知
        signalNotFull();
    return x;
}
```
代码逻辑：
1. 获取当前队列大小及可中断takeLock锁。如果成功获取锁，执行步骤2；否则等待
2. 如果队列为空，则notEmpty等待；否则执行步骤3
3. 调用`dequeue`方法，获取头结点元素
4. 当前队列大小减一
5. 如果出队后队列依然不为空，则notEmpty唤醒。
6. 如果出队后队列不满，则notFull唤醒通知
7. 返回头元素


##### dequeue源码

```java
private E dequeue() {
    // assert takeLock.isHeldByCurrentThread();
    // assert head.item == null;
    Node<E> h = head;   //获取当前头节点
    Node<E> first = h.next; //获取当前头元素的next节点
    h.next = h; // help GC  使头节点的next节点链接到本身
    head = first;   //重置头节点
    E x = first.item;   //获取当前头节点的item值
    first.item = null;  //将当前头结点的item值置为null
    return x;
}
```
在将头元素出队后，将原头结点的next执行本身，用于GC。因为LinkedBlockingQueue的头节点具有一致性:即元素为null。所以将新的head的item为null。

##### signalNotFull源码

```java
private void signalNotFull() {
    final ReentrantLock putLock = this.putLock;
    putLock.lock();
    try {
        notFull.signal();
    } finally {
        putLock.unlock();
    }
}
```

##### poll源码

```java
public E poll() {
    final AtomicInteger count = this.count;
    if (count.get() == 0)
        return null;
    E x = null;
    int c = -1;
    final ReentrantLock takeLock = this.takeLock;
    takeLock.lock();    //获取锁
    try {
        if (count.get() > 0) {  //链表不为空，从链表中取出元素
            x = dequeue();
            c = count.getAndDecrement();
            if (c > 1)
                notEmpty.signal();
        }
    } finally {
        takeLock.unlock();
    }
    if (c == capacity)
        signalNotFull();
    return x;
}

```
poll源码很简单，在队列为空时，直接返回null，在不为空时，获取头节点元素并返回。

##### remove源码

```java
public boolean remove(Object o) {
    if (o == null) return false;    //如果待移除元素为null，直接返回FALSE
    fullyLock();    //获取putLock和takeLock两个锁
    try {
        for (Node<E> trail = head, p = trail.next; p != null; trail = p, p = p.next) {   //从头结点开始比那里链表
            if (o.equals(p.item)) { //如果遍历到待删除元素，则调用unlink方法，去除指定结点
                unlink(p, trail);
                return true;
            }
        }
        return false;
    } finally {
        fullyUnlock();
    }
}
```
代码逻辑：
1. 待移除元素为空判断。为空，直接返回FALSE；否则执行步骤2.
2. 获取putLock和takeLock两个锁
3. 从head节点开始，遍历链表，如果找到待移除元素，则调用 `unlink` 方法，将待移除元素对应的节点从链表中移除（弱移除）。
4. 返回

##### unlink源码

```java
void unlink(Node<E> p, Node<E> trail) {
    // assert isFullyLocked();
    // p.next is not changed, to allow iterators that are
    // traversing p to maintain their weak-consistency guarantee.
    p.item = null;
    trail.next = p.next;
    if (last == p)
        last = trail;
    if (count.getAndDecrement() == capacity)
        notFull.signal();
}

```
该方法的功能是将节点P从前继节点trail中移除。即将trail的next节点指向P的next节点。从代码里的注释：  `p.next is not changed, to allow iterators that are traversing p to maintain their weak-consistency guarantee.` 可知，这种移除是一种弱移除。

##### contains源码

```java
public boolean contains(Object o) {
    if (o == null) return false;
    fullyLock();
    try {
        for (Node<E> p = head.next; p != null; p = p.next)  //从头结点开始遍历链表，判断元素o是否在链表中
            if (o.equals(p.item))
                return true;
        return false;
    } finally {
        fullyUnlock();
    }
}
```
代码实现逻辑就是从头结点开始遍历链表，判断元素o是否在链表中。


至此，LinkedBlockingQueue的主要方法已分析完，下面整理一下与ArrayBlockingQueue的区别


类名 | 底层数据结构 | 内部锁 | 元素数量大小类型 | 是否有界
---|---|---|---|---
LinkedBlockingQueue | 链表 | 有两个锁：takeLock锁与putLock锁。出队列使用takeLock锁；出队列使用putLock锁。 | AtomicInteger | 可以充当有界队列，也可以充当无界队列
ArrayBlockingQueue | 数组 | 只有一个lock锁。出队列和入队列使用相同锁 | int | 有界队列



#### 参考文章
[LinkedBlockingQueue源码分析](http://www.jianshu.com/p/cc2281b1a6bc)

