---
title: PriorityBlockingQueue源码学习
date: 2017-11-29 15:04:04
tags: java并发集合
---

#### 前言
PropertyBlockingQueue 是`基于数组实现的线程安全的无界队列，它是基于优先级，而不是FIFO队列`。默认情况下元素采取自然顺序升序排列，也可以自定义类实现compareTo()方法来指定元素排序规则，需要注意的是不能保证同优先级元素的顺序。虽说是无界的，但有可能会导致OutOfMemoryError异常。

#### 存储结构
![image](http://img.blog.csdn.net/20160708151139090?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)


#### 数据结构

```java
public class PriorityBlockingQueue<E> extends AbstractQueue<E>
    implements BlockingQueue<E>, java.io.Serializable {
    
    //队列默认大小
    private static final int DEFAULT_INITIAL_CAPACITY = 11;  
    //队列最大容量，超过该容量抛OutOfMemoryError异常
    private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;
    //优先级队列表示为平衡二叉堆。queue[0]为元素最小值。PriorityBlockingQueue 底层还是通过数组来实现的
    private transient Object[] queue;
    //队列元素个数 不序列化
    private transient int size;
    //比较器。在为null时使用元素的自然排序
    private transient Comparator<? super E> comparator;
    //用于队列操作的锁
    private final ReentrantLock lock;
    //队列非空条件
    private final Condition notEmpty;
    //队列扩容的 "锁" 不序列化
    private transient volatile int allocationSpinLock;
    //队列扩容的 "锁" 不序列化
    private transient volatile int allocationSpinLock;
}
```
PriorityBlockingQueue 与ArrayBlockingQueue，LinkedBlockingQueue阻塞队列不同，它的内部只维持了一个notEmpty条件，没有notFull 条件，也就是说PriorityBlockingQueue 没有队满的概念。当队列满的时候，进行扩容，当达到最大容量时，会抛出OutOfMemoryError异常。与ArrayBlockingQueue一致的是都是维持了一个锁来控制出入队行为，且底层也是通过数组实现。而LinkedBlockingQueue是通过两个锁来进行控制。


#### 构造函数

<!-- more -->


```java
public PriorityBlockingQueue() {
    this(DEFAULT_INITIAL_CAPACITY, null);
}

public PriorityBlockingQueue(int initialCapacity){
    //调用重载构造方法
    this(initialCapacity, null);
}

public PriorityBlockingQueue(int initialCapacity,
                             Comparator<? super E> comparator) {
    if (initialCapacity < 1)    //初始容量大小限制
        throw new IllegalArgumentException();

    //初始化个属性值
    this.lock = new ReentrantLock();
    this.notEmpty = lock.newCondition();
    this.comparator = comparator;
    this.queue = new Object[initialCapacity];
}

public PriorityBlockingQueue(Collection<? extends E> c) {
    //获取锁
    this.lock = new ReentrantLock();
    this.notEmpty = lock.newCondition();

    //是否需要将堆进行有序化
    boolean heapify = true; // true if not known to be in heap order
    //扫描null 值，保证队列中不会有null 元素
    boolean screen = true;  // true if must screen for nulls    //如果必须屏蔽空值，返回true

    //如果集合为SortedSet类型，则设置全局比较器comparator和heapify = false
    if (c instanceof SortedSet<?>) {
        SortedSet<? extends E> ss = (SortedSet<? extends E>) c;
        this.comparator = (Comparator<? super E>) ss.comparator();
        //SortedSet 本身是有序的，因此不用进行堆有序化
        heapify = false;
    }
    else if (c instanceof PriorityBlockingQueue<?>) { //如果集合为PriorityBlockingQueue类型，则设置全局比较器comparator和heapify = false，screen = false
        PriorityBlockingQueue<? extends E> pq =
            (PriorityBlockingQueue<? extends E>) c;
        this.comparator = (Comparator<? super E>) pq.comparator();
        //PriorityBlockingQueue 本身就不会存null 值，因此不用再次扫描
        screen = false;

        //如果已经是本身类结构，那么也无需再次堆有序化
        if (pq.getClass() == PriorityBlockingQueue.class) // exact match
            heapify = false;
    }


    Object[] a = c.toArray();   //获取集合的数组形式
    int n = a.length;
    // If c.toArray incorrectly doesn't return Object[], copy it.
    //拷贝元素
    if (a.getClass() != Object[].class) //如果c.toArray不是Object数组类型，则进行转换
        a = Arrays.copyOf(a, n, Object[].class);

    //扫描集合，不允许出现null
    if (screen && (n == 1 || this.comparator != null)) {    //如果需要屏蔽空值，则遍历判断是否有空值，有，抛出异常
        for (int i = 0; i < n; ++i)
            if (a[i] == null)
                throw new NullPointerException();
    }

    //设置全局queue与队列大小
    this.queue = a;
    this.size = n;

    //如果不知道在堆的顺序
    if (heapify)
        heapify();  //堆有序化
}
```
PriorityBlockingQueue提供了四个构造函数，但都是基于`PriorityBlockingQueue(int initialCapacity,
                                 Comparator<? super E> comparator)`的，该构造函数主要是初始化一些内部属性。
                                 

#### 源码解析
##### heapify源码

```java
//将这个数组进行 堆化
private void heapify() {
    Object[] array = queue;
    int n = size;
    int half = (n >>> 1) - 1;   //非叶子节点并且编号最大的节点
    Comparator<? super E> cmp = comparator;     // 获取 比较器, 若这里的 comparator是空, 则用元素自己实现的比较接口进行比较
    if (cmp == null) {
        for (int i = half; i >= 0; i--)
            siftDownComparable(i, (E) array[i], array, n);
    }
    else {
        for (int i = half; i >= 0; i--)
            siftDownUsingComparator(i, (E) array[i], array, n, cmp);
    }
}
```
在构造函数中，调用了heapify方法，该方法是将内部数组进行堆化，即维持堆不变性。其中涉及的两个方法下面详解。

##### 入队

###### add源码

```java
public boolean add(E e) {
    return offer(e);
}
```

###### offer源码

```java
public boolean offer(E e) {
    if (e == null)  //PriorityBlockingQueue不允许元素为null，为空，抛异常
        throw new NullPointerException();

    final ReentrantLock lock = this.lock;
    lock.lock(); //获取锁

    int n, cap;
    Object[] array;

    //如果队列满了，则进行扩容
    while ((n = size) >= (cap = (array = queue).length))
        tryGrow(array, cap);    //队列扩容

    try {
        Comparator<? super E> cmp = comparator; //获取比较器
        if (cmp == null)    //如果比较器为空，则进行默认比较
            siftUpComparable(n, e, array);
        else
            siftUpUsingComparator(n, e, array, cmp);
        size = n + 1;
        //入队后  notEmpty 条件满足，唤醒阻塞在notEmpty 条件上的一个线程
        notEmpty.signal();
    } finally {
        lock.unlock();
    }
    return true;
}
```
在添加的时候，因为队列为无界的，所以不会有阻塞的情况发生。添加的逻辑大致为：
1. 判断待插入元素是否为空。为空，抛异常；否则，执行第2步
2. 获取锁
3. 成功获取锁，while循环判断队列是否需要扩容。需要的话进行扩容操作。
4. 将待插入元素入队，并进行堆调整
5. 释放信号

###### tryGrow源码

```java
//只有在持有锁的情况下才被调用。扩容
private void tryGrow(Object[] array, int oldCap) {
    lock.unlock(); //释放锁 must release and then re-acquire main lock
    Object[] newArray = null;

    //用cas 将allocationSpinLock 设置为1，依然起到加锁功能
    if (allocationSpinLock == 0 &&
        UNSAFE.compareAndSwapInt(this, allocationSpinLockOffset,
                                 0, 1)) {
        try {
            //在队列旧容量小于64的时候，将队列容量在新增oldCap + 2；否则，新增oldCap >> 1
            int newCap = oldCap + ((oldCap < 64) ?
                                   (oldCap + 2) : // grow faster if small
                                   (oldCap >> 1));

            if (newCap - MAX_ARRAY_SIZE > 0) {    // possible overflow
                int minCap = oldCap + 1;    //超过最大容量，扩容增加1
                if (minCap < 0 || minCap > MAX_ARRAY_SIZE)
                    throw new OutOfMemoryError();

                //将容量赋值为最大值
                newCap = MAX_ARRAY_SIZE;
            }

            //queue == array 这里保证 queue还未被修改
            if (newCap > oldCap && queue == array)
                newArray = new Object[newCap];
        } finally {
            allocationSpinLock = 0;
        }
    }

    //其它线程对队列进行了改动，放弃扩容。因此才会看到在 offer 中通过while 循环来判断是否真正需要扩容
    if (newArray == null) // back off if another thread is allocating
        Thread.yield();

    //重新加锁，准备回到offer中
    lock.lock();
    if (newArray != null && queue == array) {
        //扩容，复制内容到新数组
        queue = newArray;
        System.arraycopy(array, 0, newArray, 0, oldCap);
    }
}
```
该方法只有offer方法会调用。在调用之前需要先获取到锁。在方法内部，首先释放锁，然后通过`CAS操作allocationSpinLock的值`，来控制只有一个线程可以进行扩容操作。

    为什么会在扩容的时候释放锁？
    
    个人理解：可能的原因是，在扩容期间，释放锁后，其他线程就可以操作内部
    数组了（比如出队操作），这样就实现了并发逻辑。
    这也就是为什么需要使用while循环来判断是否正真需要扩容操作。

###### siftUpComparable源码

```java
// 在位置k上插入元素x。通过上浮元素x直到它大于或者等于它的父节点或者是根节点来维持堆一致性
private static <T> void siftUpComparable(int k, T x, Object[] array) {
    //元素自身默认比较器
    Comparable<? super T> key = (Comparable<? super T>) x;

    while (k > 0) { //在首次插入元素时，直接跳过
        int parent = (k - 1) >>> 1; //元素x的父节点的位置 (k-1)/2
        Object e = array[parent];   //父节点值
        if (key.compareTo((T) e) >= 0)  //判断是否将x元素存在位置k,当前元素和父节点比较，如果当前节点大于父节点，退出循环
            break;
        array[k] = e;   //待插入元素值小于父节点元素值，则将父节点元素下沉
        k = parent; //从父节点位置开始，判断是否将x元素存在位置k
    }
    array[k] = key; //找到位置k，将元素x存放在该位置
}
```
理解这个方法之前，我们需要先了解二叉堆的相关知识。可以[参考文章](http://blog.csdn.net/u014634338/article/details/78369903)。在二叉堆中，位置i 的左右子节点的坐标为 `2*i + 1` 和 `2*i + 2`。其父节点坐标为 `(i-1)/2`。

该方法为上浮操作。大体逻辑为：将待插入元素与父节点位置的元素进行比较，并将最小值，放入父节点位置，依次循环，直到结束。
最终结果为父节点的值都大于或等于它的子节点值

> 上浮流程

<a data-flickr-embed="true"  href="https://www.flickr.com/photos/157389715@N05/26942033019/in/dateposted-public/" title="WechatIMG174"><img src="https://farm5.staticflickr.com/4561/26942033019_906da0c54d_z.jpg" width="512" height="640" alt="WechatIMG174"></a>


###### siftUpUsingComparator源码
```java 
//与方法siftUpComparable十分相似，只不过是有提供比较器
private static <T> void siftUpUsingComparator(int k, T x, Object[] array,
                                   Comparator<? super T> cmp) {
    while (k > 0) {
        int parent = (k - 1) >>> 1;
        Object e = array[parent];
        if (cmp.compare(x, (T) e) >= 0)
            break;
        array[k] = e;
        k = parent;
    }
    array[k] = x;
}
```

###### put源码

```java
public void put(E e) {
    offer(e); // never need to block
}
```

###### 超时offer源码

```java
//因为为无界队列，因为该方法不会阻塞或者返回FALSE。实际当到达队列最大值后，就抛oom异常
public boolean offer(E e, long timeout, TimeUnit unit) {
    return offer(e); // never need to block
}
```
因为为无界队列，所以put和超时offer方法都不会阻塞或者返回FALSE。实际当到达队列最大值后，就抛oom异常。

##### 出队

###### poll源码

```java
public E poll() {
    final ReentrantLock lock = this.lock;
    lock.lock();    //加锁
    try {
        return dequeue();   //返回出队元素
    } finally {
        lock.unlock();  //释放锁
    }
}
```

###### dequeue源码

```java
//只有在持有锁的情况下才被调用。总体逻辑为取出堆顶元素后，将堆最后一个元素放到堆顶位置，然后通过下沉操作，将最小元素重新放在堆顶
private E dequeue() {
    int n = size - 1;
    if (n < 0)  //队列为空，返回null
        return null;
    else {
        Object[] array = queue;
        //堆顶元素
        E result = (E) array[0];

        //堆中最后一个元素
        E x = (E) array[n];

        array[n] = null;    //将最后位置的元素置为null

        Comparator<? super E> cmp = comparator;

        //元素下沉操作
        if (cmp == null)
            siftDownComparable(0, x, array, n);
        else
            siftDownUsingComparator(0, x, array, n, cmp);
        size = n;
        return result;
    }
```
出队的主要逻辑是：将堆顶元素取出后，再将堆中最后一个元素放到堆顶的位置，然后通过下沉操作，将堆中最小元素放入堆顶。代码逻辑：
1. 如果队列为空，直接返回null
2. 获取堆顶元素及堆中最后一个元素，并将数组的最后为置为null
3. 调用方法siftDownComparable实现下沉操作，实现堆中最小元素在堆顶
4. 返回旧堆顶元素并修改堆数量。


###### siftDownComparable源码

```java
// 在位置k上插入元素x。通过下沉元素x直到它小于或者等于它的子节点或者是叶子节点来维持堆一致性
private static <T> void siftDownComparable(int k, T x, Object[] array,
                                           int n) {
    if (n > 0) {
        //元素自身默认比较器
        Comparable<? super T> key = (Comparable<? super T>)x;

        int half = n >>> 1;           // loop while a non-leaf
        while (k < half) {
            // 2*k+1 表示的k的左孩子的位置
            int child = (k << 1) + 1; // assume left child is least
            //获取左叶子节点元素
            Object c = array[child];

            //获取k的右孩子的位置
            int right = child + 1;

            //取左右孩子中元素值较小的值（这里的较小，是通过比较器来定义的较小）
            if (right < n &&
                ((Comparable<? super T>) c).compareTo((T) array[right]) > 0)
                c = array[child = right];

            //x 比左右孩子都小，那么不用继续下沉了
            if (key.compareTo((T) c) <= 0)
                break;

            // 下沉操作
            array[k] = c;
            k = child;  //k为左右叶子节点中最小值所在位置
        }
        array[k] = key;
    }
}

```
该方法为下沉操作。大体逻辑为：将待插入元素与其左右子节点的最小值进行比较。如果比左右子节点的最小值还小，则不变；否则移动位置，然后依次循环，直到结束。 最终结果为左右子节点的值都小于或等于它的父节点值

###### siftDownUsingComparator源码

```java
//同siftDownComparable方法
private static <T> void siftDownUsingComparator(int k, T x, Object[] array,
                                                int n,
                                                Comparator<? super T> cmp) {
    if (n > 0) {
        int half = n >>> 1;
        while (k < half) {
            int child = (k << 1) + 1;
            Object c = array[child];
            int right = child + 1;
            if (right < n && cmp.compare((T) c, (T) array[right]) > 0)
                c = array[child = right];
            if (cmp.compare(x, (T) c) <= 0)
                break;
            array[k] = c;
            k = child;
        }
        array[k] = x;
    }
}
```


> 下沉流程

<a data-flickr-embed="true"  href="https://www.flickr.com/photos/157389715@N05/26942122189/in/dateposted-public/" title="下沉操作"><img src="https://farm5.staticflickr.com/4560/26942122189_544418ed7c_z.jpg" width="640" height="463" alt="下沉操作"></a>


###### take源码

```java
//会阻塞
public E take() throws InterruptedException {
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    E result;
    try {
        //如果队列为空，则阻塞在notEmpty条件上
        while ( (result = dequeue()) == null)
            notEmpty.await();
    } finally {
        lock.unlock();
    }
    return result;
}
```
该方法与poll方法不同之处在于：当队列为空的时候，出队操作会阻塞。

###### 超时poll源码

```java
public E poll(long timeout, TimeUnit unit) throws InterruptedException {
    long nanos = unit.toNanos(timeout);
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();   //可中断获取锁
    E result;
    try {
        while ( (result = dequeue()) == null && nanos > 0)
            nanos = notEmpty.awaitNanos(nanos);
    } finally {
        lock.unlock();
    }
    return result;
}
```
与take方法不同之处在于：提供了一种超时等待机制，不会无限等待下去。

###### peek源码

```java
//返回队头元素，但是元素并不出队
public E peek() {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        return (size == 0) ? null : (E) queue[0];
    } finally {
        lock.unlock();
    }
}
```
该方法也是获取队列头元素，不同之处是不会有出队逻辑。

###### remove源码

```java
public boolean remove(Object o) {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        //获取元素o在队列的位置
        int i = indexOf(o);
        if (i == -1)
            return false;
        removeAt(i);
        return true;
    } finally {
        lock.unlock();
    }
}
```

###### removeAt源码

```java
private void removeAt(int i) {
    Object[] array = queue;
    int n = size - 1;
    //删除最后一个元素，直接将数组的最后位置为null
    if (n == i) // removed last element
        array[i] = null;
    else {
        E moved = (E) array[n]; //获取堆最后一个元素
        array[n] = null;
        Comparator<? super E> cmp = comparator;
        //将堆最后一个元素从位置i处进行下沉操作
        if (cmp == null)
            siftDownComparable(i, moved, array, n);
        else
            siftDownUsingComparator(i, moved, array, n, cmp);


        // 如果array[i] == moved说明未发生调整，那么则需要自下而上调整堆。如果i的位置满足：i > n >>> 1，则下沉操作不会发生，此时需要进行上浮操作
        if (array[i] == moved) {
            if (cmp == null)
                siftUpComparable(i, moved, array);
            else
                siftUpUsingComparator(i, moved, array, cmp);
        }
    }
    size = n;
}
```





#### 参考文章
[Java 并发 --- 阻塞队列之PriorityBlockingQueuey源码分析](http://blog.csdn.net/u014634338/article/details/78369903)

[JUC源码分析19-队列-PriorityBlockingQueue](http://blog.csdn.net/xiaoxufox/article/details/51860543)