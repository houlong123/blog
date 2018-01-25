---
title: DelayQueue源码学习
date: 2017-11-30 15:07:26
tags: java并发集合
---

#### 前言
<font color=red>DelayQueue是一个支持延迟获取元素的无界阻塞队列。</font>队列内部使用priorityQueue实现相应操作。存储的元素必须要继承 `Delayed接口`。PriorityQueue 和PriorityBlockingQueue队列一样，都是一种优先级的队列，内部实现原理也是使用的二叉堆。

#### 数据结构

```java
public class DelayQueue<E extends Delayed> extends AbstractQueue<E>
    implements BlockingQueue<E> {
    //可重入锁
    private final transient ReentrantLock lock = new ReentrantLock();
    //存储元素的优先级队列
    private final PriorityQueue<E> q = new PriorityQueue<E>();
    //等待队列头部元素的指定线程
    private Thread leader = null;
    //条件控制，表示是否可以从队列中取数据
    private final Condition available = lock.newCondition();
}
```
由DelayQueue类的内部数据结构可知，其内部存储的元素要实现 `Delayed接口`。在DelayQueue类中，还维持了一个 `PriorityQueue类` 的对象，用于实现DelayQueue队列的相应操作。

从`Delayed接口`的数据结构可以看出，它继承了 `Comparable接口`，这为优先级队列提供了一种排序机制。`Delayed接口`内部还有一个 `getDelay()` 方法，用于返回剩余的延迟时间：`零值或负值表示这个延迟时间已经过去了`。DelayQueue队列就是根据这个条件来控制延时获取元素的。

#### Delayed接口

<!-- more -->


```java
public interface Delayed extends Comparable<Delayed> {

    /**
     * Returns the remaining delay associated with this object, in the
     * given time unit.
     *
     * @param unit the time unit
     * @return the remaining delay; zero or negative values indicate
     * that the delay has already elapsed
     */
    long getDelay(TimeUnit unit);
}
```

#### 构造函数

```java
public DelayQueue() {}

public DelayQueue(Collection<? extends E> c) {
    this.addAll(c);
}
```
DelayQueue 内部组合PriorityQueue，对元素的操作都是通过PriorityQueue 来实现的，DelayQueue 的构造方法很简单，对于PriorityQueue 都是使用的默认参数，不能通过DelayQueue 来指定PriorityQueue的初始大小，也不能使用指定的Comparator，元素本身就需要实现Comparable ，因此不需要指定的Comparator。


#### 源码解析
##### 入队
###### add源码

```java
public boolean add(E e) {
    return offer(e);
}
```

##### offer源码

```java
public boolean offer(E e) {
    final ReentrantLock lock = this.lock;
    lock.lock();    //获取锁
    try {
        //通过PriorityQueue 来将元素入队
        q.offer(e);

        //peek 是获取的队头元素，如果队头元素为当前添加元素，则说明当前元素的优先级最小也就即将过期。这时候激活avaliable变量条件队列里面的一个线程，通知他们队列里面有元素了。
        if (q.peek() == e) {
            leader = null;
            available.signal();
        }
        return true;
    } finally {
        lock.unlock();
    }
}
```
代码逻辑：
1. 获取锁。成功，执行第2步；否则，循环等待
2. 通过PriorityQueue对象将元素入队。
3. 在入队成功后，如果队列头元素与被插入元素相同，则available唤醒，并将leader置空
4. 释放锁

##### put源码

```java
public void put(E e) {
    offer(e);
}
```

##### 超时offer源码

```java
public boolean offer(E e, long timeout, TimeUnit unit) {
    return offer(e);
}
```
与PriorityBlockingQueue队列一样，由于是无界队列，所以没有元素满的情况，但实际当到达队列最大值后，就抛oom异常。所以put和超时offer方法都不会阻塞或者返回FALSE。

##### 出队
###### poll源码

```java
public E poll() {
    final ReentrantLock lock = this.lock;
    lock.lock();     //获取同步锁
    try {
        //获取队头
        E first = q.peek();

        //如果队头为null 或者 延时还没有到，则返回null
        if (first == null || first.getDelay(NANOSECONDS) > 0)
            return null;
        else
            return q.poll();    //元素出队
    } finally {
        lock.unlock();
    }
}
```
代码逻辑：
1. 获取锁。成功，执行第2步；否则，循环获取锁
2. 获取队列头元素。
3. 如果队列为空或者延时还没到，则返回null；否则，执行第4步
4. 元素出队
5. 释放锁


##### take源码

```java
public E take() throws InterruptedException {
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();   // 获取可中断锁
    try {
        for (;;) {  //无限循环
            E first = q.peek(); //获取队列头元素
            if (first == null)  //如果队列为空，则等待
                available.await();
            else {
                //获取元素延迟时间
                long delay = first.getDelay(NANOSECONDS);
                if (delay <= 0)      //延迟时间到期，返回队列头元素
                    return q.poll();

                first = null; // don't retain ref while waiting

                // //如果有其它线程在等待获取元素，则当前线程不用去竞争，直接等待
                if (leader != null)
                    available.await();
                else {
                    Thread thisThread = Thread.currentThread();
                    leader = thisThread;
                    try {
                        //等待延迟时间到期
                        available.awaitNanos(delay);
                    } finally {
                        if (leader == thisThread)
                            leader = null;
                    }
                }
            }
        }
    } finally {
        if (leader == null && q.peek() != null)
            available.signal();
        lock.unlock();
    }
}
```
代码逻辑：
1. 获取可中断锁
2. 进入无限循环。
    1. 获取队列头元素
    2. 如果头元素为null，`available` 等待；否则执行第下步
    3. 获取头元素的延迟时间。如果延迟时间到期，返回头元素；否则，执行第下步
    4. 如果有其它线程在等待获取元素，则当前线程不用去竞争，直接等待；否则，执行第下步
    5. 获取当前线程，并设置`leader`，然后等待
    
3. 如果没有线程在等待获取元素并且队列头元素不为null，则 `available` 唤醒，并释放锁

出队方法 `take()` 与 `poll()` 方法不同之处在于：在队列为空或者延迟未到，`poll()`是直接返回null；而 `take()` 则是阻塞等待。

##### 超时poll 源码

```java
public E poll(long timeout, TimeUnit unit) throws InterruptedException {
    //超时等待时间
    long nanos = unit.toNanos(timeout);

    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();   //可中断的获取锁
    try {
        for (;;) {  //无限循环
            E first = q.peek();     //获取队头元素

            //队头为空，即队列为空
            if (first == null) {
                if (nanos <= 0) //达到超时指定时间，返回null
                    return null;
                else
                    // 如果还没有超时，那么再available条件上进行等待nanos时间
                    nanos = available.awaitNanos(nanos);
            } else {
                //获取元素延迟时间
                long delay = first.getDelay(NANOSECONDS);

                //延迟时间到期，返回队列头元素
                if (delay <= 0)
                    return q.poll();

                //延迟时间未到期，超时到期，返回null
                if (nanos <= 0)
                    return null;

                first = null; //在等待的时候，不需要持有引用。用于GC。 don't retain ref while waiting

                // 超时等待时间 < 延迟时间 或者有其它线程再取数据
                if (nanos < delay || leader != null)
                    nanos = available.awaitNanos(nanos);     //在available 条件上进行等待nanos 时间
                else {   //超时等待时间 > 延迟时间 并且没有其它线程在等待，那么当前线程成为leader，表示leader 线程 正在等待获取元素
                    Thread thisThread = Thread.currentThread();
                    leader = thisThread;
                    try {
                        long timeLeft = available.awaitNanos(delay);    //等待 延迟时间 还剩余多少
                        nanos -= delay - timeLeft;  //还需要继续等待 nanos
                    } finally {
                        //清除 leader
                        if (leader == thisThread)
                            leader = null;
                    }
                }
            }
        }
    } finally {
        //唤醒阻塞在available 的一个线程，表示可以取数据了
        if (leader == null && q.peek() != null)
            available.signal();
        lock.unlock();
    }
}
```
超时 `poll()` 方法的大致逻辑与 `take()` 方法一致。只是有细微区别。

DelayQueue队列的其他方法都是直接使用PriorityQueue来进行操作的。没什么好说的了


#### 参考文章
[Java 并发 --- 阻塞队列之DelayQueue源码分析](http://www.voidcn.com/article/p-bhnkndmy-bqr.html)

[java 之DelayQueue实际运用示例](https://www.cnblogs.com/sunzhenchao/p/3515085.html)