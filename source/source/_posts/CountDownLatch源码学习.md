---
title: CountDownLatch源码学习
date: 2017-09-26 14:26:42
tags: java并发锁
---

#### 前言
在上文 [java并发学习之Reentrantlock学习](https://houlong123.github.io/2017/09/25/java%E5%B9%B6%E5%8F%91%E5%AD%A6%E4%B9%A0%E4%B9%8BReentrantlock%E5%AD%A6%E4%B9%A0/) 中，讲解了AQS的子类独占锁`Reentrantlock`，本节讲解一下AQS的子类共享锁实现。`CountDownLatch`是共享锁的一种实现。

<!-- more -->

#### 使用

```java
/**
 * 工人类
 * @author xxx
 *
 */
class Worker {
    private String name;        // 名字
    private long workDuration;    // 工作持续时间

    /**
     * 构造器
     */
    public Worker(String name, long workDuration) {
        this.name = name;
        this.workDuration = workDuration;
    }

    /**
     * 完成工作
     */
    public void doWork() {
        System.out.println(name + " begins to work...");
        try {
            Thread.sleep(workDuration);    // 用休眠模拟工作执行的时间
        } catch(InterruptedException ex) {
            ex.printStackTrace();
        }
        System.out.println(name + " has finished the job...");
    }
}

/**
 * 测试线程
 * @author xxx
 *
 */
class WorkerTestThread implements Runnable {
    private Worker worker;
    private CountDownLatch cdLatch;

    public WorkerTestThread(Worker worker, CountDownLatch cdLatch) {
        this.worker = worker;
        this.cdLatch = cdLatch;
    }

    @Override
    public void run() {
        worker.doWork();        // 让工人开始工作
        cdLatch.countDown();    // 工作完成后倒计时次数减1
    }
}

class CountDownLatchTest {

    private static final int MAX_WORK_DURATION = 5000;    // 最大工作时间
    private static final int MIN_WORK_DURATION = 1000;    // 最小工作时间

    // 产生随机的工作时间
    private static long getRandomWorkDuration(long min, long max) {
        return (long) (Math.random() * (max - min) + min);
    }

    public static void main(String[] args) {
        CountDownLatch latch = new CountDownLatch(2);    // 创建倒计时闩并指定倒计时次数为2
        Worker w1 = new Worker("xxx", getRandomWorkDuration(MIN_WORK_DURATION, MAX_WORK_DURATION));
        Worker w2 = new Worker("王大锤", getRandomWorkDuration(MIN_WORK_DURATION, MAX_WORK_DURATION));

        new Thread(new WorkerTestThread(w1, latch)).start();
        new Thread(new WorkerTestThread(w2, latch)).start();

        try {
            latch.await();    // 等待倒计时闩减到0
            System.out.println("All jobs have been finished!");
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}

```


#### CountDownLatch源码学习
##### 内部数据结构

```java
public class CountDownLatch {

    private final Sync sync;

    /**
     * Synchronization control For CountDownLatch.
     * Uses AQS state to represent count.
     */
    private static final class Sync extends AbstractQueuedSynchronizer {}
    
}
```
由类实现可知，该类持有一个`Sync`对象，提供所有的同步机制。


##### 构造函数

```java
public CountDownLatch(int count) {
    if (count < 0) throw new IllegalArgumentException("count < 0");
    this.sync = new Sync(count);
}
```

##### Sync构造函数

```java
Sync(int count) {
    setState(count);
}
```
由Sync构造函数可知，内部会调用父类AQS的`setState`方法，设置`state`值。在前文中，有提过`state`在不同子类中代表不同的意义，在CountDownLatch中，则表示CountDownLatch的计数器的初始大小。

##### await源码

```java
public void await() throws InterruptedException {
    sync.acquireSharedInterruptibly(1);
}
```
调用了Sync的acquireSharedInterruptibly方法，因为Sync是AQS子类的原因，这里其实是直接调用了AQS的acquireSharedInterruptibly方法。

##### AQS中acquireSharedInterruptibly源码

```java
public final void acquireSharedInterruptibly(int arg)
        throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    if (tryAcquireShared(arg) < 0)
        doAcquireSharedInterruptibly(arg);
}
```
从方法名上看，这个方法的调用是响应线程的打断的，所以在前两行会检查下线程是否被打断。接着，尝试着获取共享锁，小于0，表示获取失败，通过本系列的上半部分的解读， 我们知道AQS在获取锁的思路是，先尝试直接获取锁，如果失败会将当前线程放在队列中，按照FIFO的原则等待锁。而对于共享锁也是这个思路。tryAcquireShared由子类实现。

##### CountDownLatch中tryAcquireShared源码

```java
protected int tryAcquireShared(int acquires) {
    return (getState() == 0) ? 1 : -1;
}
```
如果state变成0了，则返回1，表示获取成功，否则返回-1则表示获取失败。在获取锁失败后，应该是要将当前线程放入到队列中去。

##### doAcquireSharedInterruptibly源码

```java
private void doAcquireSharedInterruptibly(int arg)
        throws InterruptedException {
        //将当前线程包装为类型为Node.SHARED的节点，标示这是一个共享节点。
        final Node node = addWaiter(Node.SHARED);
        boolean failed = true;
        try {
            for (;;) {
                final Node p = node.predecessor();
                //如果新建节点的前一个节点，就是Head，说明当前节点是AQS队列中等待获取锁的第一个节点，
                //按照FIFO的原则，可以直接尝试获取锁。
                if (p == head) {
                    int r = tryAcquireShared(arg);
                    if (r >= 0) {//获取成功，需要将当前节点设置为AQS队列中的第一个节点，这是AQS的规则//队列的头节点表示正在获取锁的节点
                        setHeadAndPropagate(node, r);
                        p.next = null; // help GC
                        failed = false;
                        return;
                    }
                }
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt()) //检查下是否需要将当前节点挂起
                    throw new InterruptedException();
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```

##### setHeadAndPropagate源码

```java
private void setHeadAndPropagate(Node node, int propagate) {
        Node h = head; // Record old head for check below
        setHead(node);
        /*
         * Try to signal next queued node if:
         *   Propagation was indicated by caller,
         *     or was recorded (as h.waitStatus either before
         *     or after setHead) by a previous operation
         *     (note: this uses sign-check of waitStatus because
         *      PROPAGATE status may transition to SIGNAL.)
         * and
         *   The next node is waiting in shared mode,
         *     or we don't know, because it appears null
         *
         * The conservatism in both of these checks may cause
         * unnecessary wake-ups, but only when there are multiple
         * racing acquires/releases, so most need signals now or soon
         * anyway.
         */
        if (propagate > 0 || h == null || h.waitStatus < 0 ||
            (h = head) == null || h.waitStatus < 0) {
            Node s = node.next;
            if (s == null || s.isShared())
                doReleaseShared();
        }
    }
```
首先，更换了头节点，然后，将当前节点的下一个节点取出来，如果同样是“shared”类型的，再做一个"releaseShared"操作。

##### doReleaseShared源码

```java
private void doReleaseShared() {
        /*
         * Ensure that a release propagates, even if there are other
         * in-progress acquires/releases.  This proceeds in the usual
         * way of trying to unparkSuccessor of head if it needs
         * signal. But if it does not, status is set to PROPAGATE to
         * ensure that upon release, propagation continues.
         * Additionally, we must loop in case a new node is added
         * while we are doing this. Also, unlike other uses of
         * unparkSuccessor, we need to know if CAS to reset status
         * fails, if so rechecking.
         */
        for (;;) {
            Node h = head;
            if (h != null && h != tail) {
                int ws = h.waitStatus;
                //如果当前节点是SIGNAL意味着，它正在等待一个信号，  
                //或者说，它在等待被唤醒，因此做两件事，1:重置waitStatus标志位，2:重置成功后,唤醒下一个节点。
              
                if (ws == Node.SIGNAL) {
                    if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                        continue;            // loop to recheck cases
                    unparkSuccessor(h);
                }
                //如果本身头节点的waitStatus是出于重置状态（waitStatus==0）的，将其设置为“传播”状态。
                //意味着需要将状态向后一个节点传播。
                else if (ws == 0 &&
                         !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                    continue;                // loop on failed CAS
            }
            if (h == head)                   // loop if head changed
                break;
        }
    }
```

##### countDown源码

```java
public void countDown() {
    sync.releaseShared(1);
}
```
调用了AQS的releaseShared方法，并传参1.

##### AQS中releaseShared源码
```java
public final boolean releaseShared(int arg) {
    if (tryReleaseShared(arg)) {
        doReleaseShared();
        return true;
    }
    return false;
}
```
同样先尝试去释放锁，tryReleaseShared同样为空方法，留给子类自己去实现，以下是CountDownLatch的内部类Sync的实现：


##### CountDownLatch中tryReleaseShared源码

```java
protected boolean tryReleaseShared(int releases) {
    // Decrement count; signal when transition to zero
    for (;;) { //死循环更新state的值，实现state的减1操作，之所以用死循环是为了确保state值的更新成功。
        int c = getState();
        if (c == 0)
            return false;
        int nextc = c-1;
        if (compareAndSetState(c, nextc))
            return nextc == 0;
    }
}
```
如果state的值为0，在CountDownLatch中意味：所有的子线程已经执行完毕，这个时候可以唤醒调用await()方法的线程了，而这些线程正在AQS的队列中，并被挂起的，所以下一步应该去唤醒AQS队列中的头节点了（AQS的队列为FIFO队列），然后由头节点去依次唤醒AQS队列中的其他共享节点。如果tryReleaseShared返回true,进入doReleaseShared()方法。

当线程被唤醒后，会重新尝试获取共享锁，而对于CountDownLatch线程获取共享锁判断依据是state是否为0，而这个时候显然state已经变成了0，因此可以顺利获取共享锁并且依次唤醒AQS队里中后面的节点及对应的线程。

#### 参考文章
[深度解析Java 8：JDK1.8 AbstractQueuedSynchronizer的实现分析（下）](http://www.infoq.com/cn/articles/java8-abstractqueuedsynchronizer)
