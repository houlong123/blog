---
title: ArrayBlockingQueue源码学习
date: 2017-11-16 14:59:01
tags: java并发集合
---

#### 前言
ArrayBlockingQueue是一个
**`基于数组且有界的阻塞队列`** 。此队列按 FIFO（先进先出）原则对元素（元素不允许为null）进行排序。此队列一经创建，其容量不能再改变。

#### 源码解析
##### 内部数据结构

```java
public class ArrayBlockingQueue<E> extends AbstractQueue<E>
        implements BlockingQueue<E>, java.io.Serializable {
    /** The queued items */
    //内部数据结构
    final Object[] items;

    /** items index for next take, poll, peek or remove */
    //下次从数组中取元素的索引
    int takeIndex;

    /** items index for next put, offer, or add */
    //下次往数组中添加元素的索引
    int putIndex;

    /** Number of elements in the queue */
    //队列中的元素数
    int count;

    /*
     * Concurrency control uses the classic two-condition algorithm
     * found in any textbook.
     */

    /** Main lock guarding all access */
    final ReentrantLock lock;

    /** Condition for waiting takes */
    private final Condition notEmpty;

    /** Condition for waiting puts */
    private final Condition notFull;
       
}
```
从ArrayBlockingQueue内部数据结构中，可以看出：该并发集合其底层是使用了 [ReentrantLock](https://houlong123.github.io/2017/09/25/java并发学习之Reentrantlock学习/) 和Condition来完成并发控制的。内部是基于数组。


##### 构造函数

<!-- more -->

```java
public ArrayBlockingQueue(int capacity) {
    this(capacity, false);
}

public ArrayBlockingQueue(int capacity, boolean fair) {
    if (capacity <= 0)
        throw new IllegalArgumentException();
    this.items = new Object[capacity];  //初始化内部数组大小
    lock = new ReentrantLock(fair); //初始化锁
    notEmpty = lock.newCondition();
    notFull =  lock.newCondition();
}

public ArrayBlockingQueue(int capacity, boolean fair,
                          Collection<? extends E> c) {
    this(capacity, fair);   //调用重载方法，初始化变量

    //在加锁的条件下，将集合c中的元素放入items中
    final ReentrantLock lock = this.lock;
    lock.lock(); // Lock only for visibility, not mutual exclusion
    try {
        int i = 0;
        try {
            for (E e : c) {
                checkNotNull(e);
                items[i++] = e;
            }
        } catch (ArrayIndexOutOfBoundsException ex) {
            throw new IllegalArgumentException();
        }
        count = i;
        putIndex = (i == capacity) ? 0 : i;
    } finally {
        lock.unlock();
    }
}
```
从源码中可知，ArrayBlockingQueue类有三个构造函数，其中，都是基于 `ArrayBlockingQueue(int capacity, boolean fair)` 构造函数的。该构造函数主要是初始化了内部数组，锁及Condition两个对象。


##### 元素添加源码

###### add源码

```java
//add添加元素时，在队列满的时候不会阻塞，直接抛出异常
public boolean add(E e) {
    return super.add(e);    //调用父类AbstractQueue中的add方法，最钟调用offer方法
}
```
由源码可知，add方法内部调用了父类AbstractQueue的add方法，父类AbstractQueue的add方法为一个模板方法，最终实现元素添加是在offer方法中。


###### AbstractQueue的add源码

```java
public boolean add(E e) {
    if (offer(e))
        return true;
    else
        throw new IllegalStateException("Queue full");
}
```
由源码可知，在添加元素成功后，返回true，否则，抛出异常。


###### offer方法

```java
//此方法为ArrayBlockingQueue类中的实现
public boolean offer(E e) {
    checkNotNull(e);    //元素为空判断
    final ReentrantLock lock = this.lock;
    lock.lock();    //加锁
    try {
        if (count == items.length)  //队列满的时候，返回FALSE
            return false;
        else {
            enqueue(e); //入队
            return true;
        }
    } finally {
        lock.unlock();
    }
}
```
代码逻辑：
1. 元素为空判断。为空，抛出异常；否则，执行第2步
2. 获取可重入锁对象。获取锁失败，等待；获取成功，执行第3步
3. 如果队列已满，返回FALSE，否则，调用enqueue方法进行入队操作，并返回true
4. 释放锁


###### enqueue源码

```java
private void enqueue(E x) {
    // assert lock.getHoldCount() == 1;
    // assert items[putIndex] == null;
    final Object[] items = this.items;  //获取当前数组items
    items[putIndex] = x;    //将元素添加进数组中
    if (++putIndex == items.length) //在队列满时，重置putIndex的值，从队列头重新开始
        putIndex = 0;
    count++;    //元素数加1
    notEmpty.signal();  //队列不空通知
}
```
代码逻辑：
1. 获取当前数组
2. 将元素添加进数组中
3. 增加索引`putIndex`，如果队列满，则重置putIndex=0
4. 增加`count`值
5. 唤醒通知

###### put源码

```java
//add添加元素时，在队列满的时候会阻塞，与add方法的处理逻辑不同
public void put(E e) throws InterruptedException {
    checkNotNull(e);    //元素为空判断
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();   //获取可中断锁
    try {
        while (count == items.length)   //队列满时，无限等待
            notFull.await();
        enqueue(e); //入队
    } finally {
        lock.unlock();
    }
}
```
代码逻辑：
1. 元素为空判断。为空，抛出异常；否则，执行第2步
2. 获取可重入可中断锁对象。获取锁失败，等待；获取成功，执行第3步
3. while循环判断队列是否已满。已满，等待；不满，调用enqueue方法进行入队操作
4. 释放锁

<font color=red>备注：元素入队有add与put两种方法。二者的不同之处在于：add方法在添加元素时，如果队列已满，则抛出异常；put方法在添加元素时，如果队列已满，则是无限等待。</font>


###### 超时offer源码

```java
public boolean offer(E e, long timeout, TimeUnit unit)
    throws InterruptedException {

    checkNotNull(e);    //元素为空判断
    long nanos = unit.toNanos(timeout);
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();   //获取可中断锁
    try {
        while (count == items.length) { //队列满时
            if (nanos <= 0) //等待超时时间已过，返回
                return false;
            nanos = notFull.awaitNanos(nanos);  //等待
        }
        enqueue(e); //入队
        return true;
    } finally {
        lock.unlock();
    }
}
```
该方法与重载方法 `offer(E e)` 的不同之处在于：该方法提供了超时等待机制，在等待超时时，返回FALSE。

> 模拟put操作

+ 队列初始态
    
    <a data-flickr-embed="true"  href="https://www.flickr.com/photos/157389715@N05/37566092135/in/dateposted-public/" title="初始化状态"><img src="https://farm5.staticflickr.com/4516/37566092135_9260cb4e4d_z.jpg" width="546" height="392" alt="初始化状态"></a>

+ 插入元素10

    <a data-flickr-embed="true"  href="https://www.flickr.com/photos/157389715@N05/38420830332/in/dateposted-public/" title="插入数据"><img src="https://farm5.staticflickr.com/4542/38420830332_f9158e874a_z.jpg" width="546" height="392" alt="插入数据"></a>

##### 获取元素
###### poll源码

```java
//将队列的头元素出队
public E poll() {
    final ReentrantLock lock = this.lock;
    lock.lock();    //加锁
    try {
        return (count == 0) ? null : dequeue(); //队列为空时，返回null，否则调用dequeue方法
    } finally {
        lock.unlock();
    }
}
```
由源码可知，在出队列时，会首先获取锁，在获取锁成功后，如果队列为空，返回null；否则，调用dequeue方法进行出队操作。


###### dequeue源码

```java
private E dequeue() {
    // assert lock.getHoldCount() == 1;
    // assert items[takeIndex] != null;
    final Object[] items = this.items;  //获取当前数组items
    @SuppressWarnings("unchecked")
    E x = (E) items[takeIndex]; //获取takeIndex处的元素
    items[takeIndex] = null;    //将索引takeIndex处的元素置空
    if (++takeIndex == items.length)    //如果已取到数组尾部，则重置takeIndex为0，从数组头提取元素
        takeIndex = 0;
    count--;    //数组元素数减一
    if (itrs != null)
        itrs.elementDequeued();
    notFull.signal();   //队列非满通知
    return x;
}
```
出队dequeue方法与入队enqueue的逻辑大致一致。在出队时，首先获取索引takeIndex处的元素并将takeIndex处的值置为空，然后修改takeIndex的值，并进行通知。

###### 可超时poll源码

```java
public E poll(long timeout, TimeUnit unit) throws InterruptedException {
    long nanos = unit.toNanos(timeout);
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    try {
        while (count == 0) {    //在队列为空时，超时等待
            if (nanos <= 0)
                return null;
            nanos = notEmpty.awaitNanos(nanos);
        }
        return dequeue();
    } finally {
        lock.unlock();
    }
}
```
该方法提供了一种超时等待机制，在队列为空的情况下，如果等待超时，则直接返回null。

###### take源码

```java
public E take() throws InterruptedException {
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();   //获取非中断锁
    try {
        while (count == 0)  //队列没有元素，无限等待
            notEmpty.await();
        return dequeue();   //出队
    } finally {
        lock.unlock();
    }
}
```
该方法与poll方法不同之处在于：在队列没有元素的情况下，poll方法直接返回null；而take方法则阻塞，直到队列有元素为止。

###### peek源码

```java
public E peek() {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        return itemAt(takeIndex); // null when queue is empty
    } finally {
        lock.unlock();
    }
}

final E itemAt(int i) {
    return (E) items[i];
}
```
方法很简单，不在详述。

> 模拟take操作

+ 首次take操作
    
    <a data-flickr-embed="true"  href="https://www.flickr.com/photos/157389715@N05/24582389988/in/dateposted-public/" title="take操作"><img src="https://farm5.staticflickr.com/4577/24582389988_4819eb4dea_z.jpg" width="524" height="392" alt="take操作"></a>

##### remove源码

```java
public boolean remove(Object o) {
    if (o == null) return false;    //待移除元素为空，直接返回FALSE
    final Object[] items = this.items;  //获取当前数组
    final ReentrantLock lock = this.lock;   //获取锁
    lock.lock();
    try {
        if (count > 0) {    //队列有元素
            final int putIndex = this.putIndex;
            int i = takeIndex;
            do {    //循环遍历数组，从takeIndex开始，直到i == putIndex
                if (o.equals(items[i])) {
                    removeAt(i);    //移除指定位置的元素
                    return true;
                }
                if (++i == items.length)
                    i = 0;
            } while (i != putIndex);
        }
        return false;
    } finally {
        lock.unlock();
    }
}
```
代码逻辑：
1. 待移除元素为空判断，为空，返回FALSE
2. 获取锁
3. 队列不为空，执行第4步；为空，返回FALSE
4. 从索引 `takeIndex` 处遍历内部数组，判断与待移除元素是否相等，相等，调用removeAt方法移除指定位置处的元素。否则，返回FALSE


###### removeAt源码

```java
void removeAt(final int removeIndex) {
    // assert lock.getHoldCount() == 1;
    // assert items[removeIndex] != null;
    // assert removeIndex >= 0 && removeIndex < items.length;
    final Object[] items = this.items;
    if (removeIndex == takeIndex) { //待移除的元素为数组中开始元素
        // removing front item; just advance
        items[takeIndex] = null;    //takeIndex位置上的元素置为空
        if (++takeIndex == items.length)
            takeIndex = 0;
        count--;
        if (itrs != null)
            itrs.elementDequeued();
    } else {    //待移除的元素不是数组中开始元素
        // an "interior" remove

        // slide over all others up through putIndex.
        final int putIndex = this.putIndex;
        for (int i = removeIndex;;) {
            int next = i + 1;
            if (next == items.length)
                next = 0;
            if (next != putIndex) { //将待移除元素后的所以元素往前移动
                items[i] = items[next];
                i = next;
            } else {
                items[i] = null;
                this.putIndex = i;
                break;
            }
        }
        count--;
        if (itrs != null)
            itrs.removedAt(removeIndex);
    }
    notFull.signal();
}
```
代码逻辑：
1. 获取原数组
2. 如果待删除元素索引 `removeIndex` 等于 `takeIndex`，则直接将`takeIndex`的元素置为空，并修改`takeIndex`值即可。否则，执行第3步
3. 从待删除元素索引 `removeIndex` 开始，将其后的元素逐一向前移动，并把最末尾的元素置为空。
4. 唤醒通知。

> 模拟remove操作
+ 初始状态

    <a data-flickr-embed="true"  href="https://www.flickr.com/photos/157389715@N05/37567430565/in/dateposted-public/" title="未命名文件"><img src="https://farm5.staticflickr.com/4523/37567430565_f271740798_z.jpg" width="524" height="395" alt="未命名文件"></a>

    
+ 调用remove方法，移除元素23

    + 首次for循环
    
    <a data-flickr-embed="true"  href="https://www.flickr.com/photos/157389715@N05/38398090676/in/dateposted-public/" title="未命名文件 (1)"><img src="https://farm5.staticflickr.com/4557/38398090676_6158a52d19_z.jpg" width="524" height="395" alt="未命名文件 (1)"></a>
    
    + 二次for循环
    
    <a data-flickr-embed="true"  href="https://www.flickr.com/photos/157389715@N05/37739182084/in/dateposted-public/" title="未命名文件 (2)"><img src="https://farm5.staticflickr.com/4563/37739182084_98507671c1_z.jpg" width="524" height="395" alt="未命名文件 (2)"></a>
    
    + 三次for循环
    
    <a data-flickr-embed="true"  href="https://www.flickr.com/photos/157389715@N05/37739214404/in/dateposted-public/" title="未命名文件 (3)"><img src="https://farm5.staticflickr.com/4557/37739214404_914121d0fd_z.jpg" width="539" height="395" alt="未命名文件 (3)"></a>
    
    至此，remove操作完成。
    

###### contains源码

```java
public boolean contains(Object o) {
    if (o == null) return false;    //如果元素为空，直接返回FALSE
    final Object[] items = this.items;  //获取原数组
    final ReentrantLock lock = this.lock;
    lock.lock();    //获取锁
    try {
        if (count > 0) {    //队列不为空的情况下，从索引takeIndex处开始遍历数组，直到putIndex处，如果找到，返回true，否则，返回FALSE
            final int putIndex = this.putIndex;
            int i = takeIndex;
            do {
                if (o.equals(items[i]))
                    return true;
                if (++i == items.length)
                    i = 0;
            } while (i != putIndex);
        }
        return false;
    } finally {
        lock.unlock();
    }
}
```
很简单，遍历数组，逐个比较。


#### 参考文章
[BlockingQueue之ArrayBlockingQueue](http://blog.csdn.net/u010412719/article/details/52337471)

[ArrayBlockingQueue源码分析](http://www.jianshu.com/p/9a652250e0d1)
