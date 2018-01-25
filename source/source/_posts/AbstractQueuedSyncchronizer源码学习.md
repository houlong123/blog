---
title: AbstractQueuedSyncchronizer源码学习
date: 2017-09-11 14:19:41
tags: java并发锁
---

#### 前言
在日常编程中，我们经常使用到的锁：`ReentrantLock`，`CountDownLatch`，`ReentrantReadWriteLock`等，他们的内部都一个名为`Sync`的静态抽象内部类，该类都实现了同一个名为`AbstractQueuedSyncchronizer`的接口。该接口为Java并发包提供的一个同步基础机制。

```java
abstract static class Sync extends AbstractQueuedSynchronizer{
    ....
}
```
AbstractQueuedSynchronizer在JDK1.8中还有如下图所示的众多子类:

![](http://thumbsnap.com/i/cViZmOEn.jpg)

为了方便，通常使用AQS代替AbstractQueuedSynchronizer。

#### AQS源码解析

<!-- more -->

##### AQS类内部数据结构

```java
public abstract class AbstractQueuedSynchronizer
    extends AbstractOwnableSynchronizer
    implements java.io.Serializable {
    //等待队列的头节点。初始化或者setHead()方法可以进行设置。
    private transient volatile Node head;
    
    //等待队列的尾节点。只能通过enq()方法添加新的等待节点
    private transient volatile Node tail;
    
    //同步状态。在不同的场景下，代表不同的含义，比如在ReentrantLock中，表示加锁的次数，在CountDownLatch中，则表示CountDownLatch的计数器的初始大小。在ReentrantReadWriteLock中，高16位表示读锁，低16位表示写锁
    private volatile int state;
    
    //原子量的一些较底层的操作都是来自sun.misc.Unsafe类，所以内部有一个Unsafe的静态引用。
    private static final Unsafe unsafe = Unsafe.getUnsafe();

    
    
     //等待队列节点类(双向链表)。等待队列是CLH队列的一种。
     //CLH队列是维护一组线程的严格按照FIFO的队列。通常用于自旋锁
    static final class Node {
        //表示节点在共享模式下等待的常量
        static final Node SHARED = new Node();
        
        //表示节点在独占模式下等待的常量
        static final Node EXCLUSIVE = null;
        
        //当前节点的线程被取消
        static final int CANCELLED =  1;
        
        //后继节点的线程需要被唤醒
        static final int SIGNAL    = -1;
        
        //表示当前节点的线程正在等待某个条件
        static final int CONDITION = -2;
        
        //表示接下来的一个共享模式请求set(acquireShared)要无条件的传递(往后继节点方向)下去
        static final int PROPAGATE = -3;
        
        //等待状态。用0初始化为正常同步节点，用CONDITION初始化为条件节点。通过CAS操作进行修改
        volatile int waitStatus;
        
        //链接到当前节点/线程依靠用于检查waitStatus的前导节点
        volatile Node prev;
        
        //链接到当前节点/线程要唤醒的后继节点
        volatile Node next;
        
        //排队节点对应的线程，创建时初始化，使用后置为null
        volatile Thread thread;
    }
    
    //支持独占模式下的(锁)条件，这个条件的功能与Object的wait和notify/notifyAll的功能类似，但更加明确和易用。
    public class ConditionObject implements Condition, java.io.Serializable {
        private static final long serialVersionUID = 1173984872572414699L;
        /** First node of condition queue. */
        private transient Node firstWaiter;
        /** Last node of condition queue. */
        private transient Node lastWaiter;
        
    }
    
}
```
上面列出了AQS类的内部数据结构，从源码中可以得知，该类主要有：
+ **内部类Node**：等待队列节点类。可知内部的同步等待队列是由一系列节点组成的一个双向链表。如果一个线程竞争失败，只需将这个线程及相关信息组成一个节点，拼接到队列链表尾部(尾节点)即可；如果要将一个线程竞争成功，只需重新设置新的队列首部(头节点)即可。
+ **head**：等待队列头结点。
+ **tail**：等待队列尾结点。
+ **state**：同步状态。在不同的场景下，代表不同的含义，比如在ReentrantLock中，表示加锁的次数，在CountDownLatch中，则表示CountDownLatch的计数器的初始大小。在ReentrantReadWriteLock中，高16位表示读锁，低16位表示写锁。
+ **unsafe**：底层操作类。原子量的一些较底层的操作都是来自sun.misc.Unsafe类，所以内部有一个Unsafe的静态引用。

由源码可以看到AQS内部的整体数据结构由一个同步等待队列+一个(原子的)int域构成。由节点模式可知，AQS的功能可以分为两类：独占功能和共享功能。它的所有子类中，要么实现并使用了它独占功能的API，要么使用了共享锁的功能，而不会同时使用两套API，即便是它最有名的子类ReentrantReadWriteLock，也是通过两个内部类：读锁和写锁，分别实现的两套API来实现的。

##### 独占API解析

###### acquire源码

```java
    //独占模式中获取锁，忽略中断。获取不到则创建一个waiter（当前线程）后放到队列中
public final void acquire(int arg) {
    if (!tryAcquire(arg) && acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```
从方法名字上看语义是，尝试获取锁，获取不到则创建一个waiter（当前线程）后放到队列中。

###### tryAcquire源码
```java
//获取锁机制
protected boolean tryAcquire(int arg) {
    throw new UnsupportedOperationException();
}
```
在AQS类中，tryAcquire()方法只是简单的抛出了异常。因为锁有公平锁和非公平锁两种，因此该方法有不同的实现方式。(为什么不使用抽象方法？一种版本说：原因是AQS有两种功能，面向两种使用场景，需要给子类定义的方法都是抽象方法了，会导致子类无论如何都需要实现另外一种场景的抽象方法，显然，这对子类来说是不友好的。)
+ **公平锁**：每个线程抢占锁的顺序为先后调用lock方法的顺序依次获取锁。

+ **非公平锁**：每个线程抢占锁的顺序不定，谁运气好，谁就获取到锁，和调用lock方法的先后顺序无关。


###### addWaiter源码

```java
//竞争锁失败，添加到等待队列
private Node addWaiter(Node mode) {
        //根据当前线程，创建一个node。创建好节点后，将节点加入到队列尾部，此处，在队列不为空的时候，先尝试通过cas方式修改尾节点为最新的节点，如果修改失败，意味着有并发，这个时候才会进入enq中死循环，“自旋”方式修改
        Node node = new Node(Thread.currentThread(), mode);
        // Try the fast path of enq; backup to full enq on failure
        Node pred = tail;
        if (pred != null) {
            //如果同步等待队列尾节点不为null,将当前(线程的)Node链接到尾节点。
            node.prev = pred;
            if (compareAndSetTail(pred, node)) {            //尝试将当前Node设置(原子操作)为同步等待队列的尾节点。
                //如果设置成功，完成链接(pred的next指向当前节点)。
                pred.next = node;
                return node;
            }
        }
        //如果同步等待队列尾节点为null，或者快速入队过程中设置尾节点失败，
        //进行正常的入队过程，调用enq方法。
        enq(node);
        return node;
    }
```
先使用当前线程和节点模式创建一个节点，创建好节点后，将节点加入到队列尾部。此处，在队列不为空的时候，先尝试通过cas方式修改尾节点为最新的节点，如果修改失败，意味着有并发，这个时候才会进入enq中死循环，“自旋”方式修改。

###### enq源码
```java
//添加节点到队列，如果需要的话，先进行初始化。
private Node enq(final Node node) {
        for (;;) {
            Node t = tail;
            if (t == null) { // Must initialize
                /*
                 * 如果同步等待队列尾节点为null，说明还没有任何线程进入同步等待队列，
                 * 这时要初始化同步等待队列：创建一个(dummy)节点，然后尝试将这个
                 * 节点设置(CAS)为头节点，如果设置成功，将尾节点指向头节点
                 * 也就是说，第一次有线程进入同步等待队列时，要进行初始化，初始化
                 * 的结果就是头尾节点都指向一个哑(dummy)节点。
                 */
                if (compareAndSetHead(new Node()))
                    tail = head;
            } else {
                //将当前(线程)节点的前驱节点指向同步等待队列的尾节点。
                node.prev = t;
                if (compareAndSetTail(t, node)) { //尝试将当前节点设置为同步等待队列的尾节点。
                    t.next = node;
                    return t;
                }
            }
        }
    }
```
将线程的节点接入到队里中后，当然还需要做一件事:将当前线程挂起！这个事，由acquireQueued来做。

###### acquireQueued源码
```java
final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                //找到当前节点的前驱节点p
                final Node p = node.predecessor();
                if (p == head && tryAcquire(arg)) {  //如果当前的节点是head说明他是队列中第一个“有效的”节点，因此尝试获取
                    //如果p节点是头节点且tryAcquire方法返回true。那么将
                    //当前节点设置为头节点。
                    //从这里可以看出，请求成功且已经存在队列中的节点会被设置成头节点
                    setHead(node);
                    //将p的next引用置空，帮助GC，现在p已经不再是头节点了。
                    p.next = null; // help GC
                    //设置请求标记为成功
                    failed = false;
                    //传递中断状态，并返回
                    return interrupted;
                }
                //如果p节点不是头节点，或者tryAcquire返回false，说明请求失败。
                //那么首先需要判断请求失败后node节点是否应该被阻塞，如果应该
                //被阻塞，那么阻塞node节点，并检测中断状态。
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    //如果有中断，设置中断状态。
                    interrupted = true;
            }
        } finally {
            if (failed) //最后检测一下如果请求失败(异常退出)，取消请求。
                cancelAcquire(node);  // 取消请求，对应到队列操作，就是将当前节点从队列中移除。
        }
    }
```

###### shouldParkAfterFailedAcquire源码

```java
//检查并更新无法获取的节点的状态。如果线程应该被阻塞，返回true
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
        int ws = pred.waitStatus;//获取当前节点的前驱节点的等待状态。
        /**
         *   //当前节点的线程被取消
         *   static final int CANCELLED =  1;
         *   //后继节点的线程需要被唤醒
         *   static final int SIGNAL    = -1;
         *   //表示当前节点的线程正在等待某个条件
         *   static final int CONDITION = -2;
         *   // 表示接下来的一个共享模式请求(acquireShared)要无条件的传递(往后继节点方向)下去
         *   static final int PROPAGATE = -3;
         */
        if (ws == Node.SIGNAL)  //只有当前节点的前一个节点的等待状态为SIGNAL时，当前节点才能被挂起。
            /*
             * This node has already set status asking a release
             * to signal it, so it can safely park.
             */
            /*
             * 如果当前节点的前驱节点的状态为SIGNAL，说明当前节点已经声明了需要被唤醒，
             * 所以可以阻塞当前节点了，直接返回true。
             * 一个节点在其被阻塞之前需要线程"声明"一下其需要唤醒(就是将其前驱节点
             * 的等待状态设置为SIGNAL，注意其前驱节点不能是取消状态，如果是，要跳过)
             */
            return true;
        if (ws > 0) {
            /*
             * Predecessor was cancelled. Skip over predecessors and
             * indicate retry.
             * 如果前驱节点的状态为取消，则跳过并重试
             */
            do {    //循环跳过状态为CANCELLED的节点
                node.prev = pred = pred.prev;
            } while (pred.waitStatus > 0);
            pred.next = node;
        } else {
            /*
             * waitStatus must be 0 or PROPAGATE.  Indicate that we
             * need a signal, but don't park yet.  Caller will need to
             * retry to make sure it cannot acquire before parking.
             */
            compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
        }
        return false;
    }
```
当判断当前线程应该被阻塞后，应该要唤醒下一节点，即要调用parkAndCheckInterrupt()方法唤醒线程。

###### parkAndCheckInterrupt源码
```java
private final boolean parkAndCheckInterrupt() {
    LockSupport.park(this);//阻塞当前线程。
    return Thread.interrupted();//线程被唤醒，方法返回当前线程的中断状态，并重置当前线程的中断状态(置为false)。
}
```
如果acquireQueued方法返回true，还需要调用一下selfInterrupt方法，去中断当前线程。如果acquireQueued方法异常退出，需要执行cancelAcquire方法，即把当前节点移除队列。

###### selfInterrupt源码
```java
static void selfInterrupt() {
    Thread.currentThread().interrupt();
}
```

###### cancelAcquire源码

```java
//取消持续的获取尝试
    private void cancelAcquire(Node node) {
        // Ignore if node doesn't exist
        if (node == null)
            return;

        //跳过首先将要取消的节点的thread域置空
        node.thread = null;

        //跳过状态为"取消"的前驱节点。
        Node pred = node.prev;
        while (pred.waitStatus > 0)
            node.prev = pred = pred.prev;

        //predNext节点(node节点前面的第一个非取消状态节点的后继节点)是需要"断开"的节点。
        //下面的CAS操作会达到"断开"效果，但(CAS操作)也可能会失败，因为可能存在其他"cancel"
        // 或者"singal"的竞争
        Node predNext = pred.next;

        node.waitStatus = Node.CANCELLED;

        //如果当前节点是尾节点，那么删除当前节点(将当前节点的前驱节点设置为尾节点)
        if (node == tail && compareAndSetTail(node, pred)) {
            //将前驱节点(已经设置为尾节点)的next置空。
            compareAndSetNext(pred, predNext, null);
        } else {
            int ws;

            //如果当前节点不是尾节点，说明后面有其他等待线程，需要做一些唤醒工作。
            // 如果当前节点不是头节点，那么尝试将当前节点的前驱节点
            // 的等待状态改成SIGNAL，并尝试将前驱节点的next引用指向
            // 其后继节点。否则，唤醒后继节点。
            if (pred != head &&
                ((ws = pred.waitStatus) == Node.SIGNAL ||
                 (ws <= 0 && compareAndSetWaitStatus(pred, ws, Node.SIGNAL))) &&
                pred.thread != null) {
                //如果当前节点的前驱节点不是头节点，那么需要给当前节点的后继节点一个"等待唤醒"的标记
                //将当前节点的前驱节点的后继节点设置为当前节点的后继节点。
                Node next = node.next;
                if (next != null && next.waitStatus <= 0)
                    compareAndSetNext(pred, predNext, next);
            } else {
                //唤醒当前节点的后继节点
                unparkSuccessor(node);
            }

            node.next = node; // help GC
        }
    }
```
在请求失败(异常退出)时，会把当前节点从等待队列中移除。

###### unparkSuccessor源码

```java
//如果后继节点存在，则唤醒后继节点
private void unparkSuccessor(Node node) {
        /*
         * If status is negative (i.e., possibly needing signal) try
         * to clear in anticipation of signalling.  It is OK if this
         * fails or if status is changed by waiting thread.
         */
        int ws = node.waitStatus;
        if (ws < 0)
            compareAndSetWaitStatus(node, ws, 0);

        /*
         * Thread to unpark is held in successor, which is normally
         * just the next node.  But if cancelled or apparently null,
         * traverse backwards from tail to find the actual
         * non-cancelled successor.
         *
         * 需被唤醒的线程在后继节点中，正常情况的话被唤醒的节点将是是下一个节点。如果下一个节点是取消状态或者为null的话，将从节点尾部开始寻找，知道找到未取消的节点。
         */
        Node s = node.next;
        if (s == null || s.waitStatus > 0) {
            s = null;
            for (Node t = tail; t != null && t != node; t = t.prev)  //寻找的顺序是从队列尾部开始往前去找的最前面的一个waitStatus小于0的节点。
                if (t.waitStatus <= 0)
                    s = t;
        }
        if (s != null)
            LockSupport.unpark(s.thread);   //唤醒线程
    }
```
对线程的挂起及唤醒操作是通过使用UNSAFE类调用JNI方法实现的。

到此为止，一个线程对于锁的一次竞争才告于段落，结果有两种，要么成功获取到锁（不用进入到AQS队列中），要么，获取失败，被挂起，等待下次唤醒后继续循环尝试获取锁。

---

看完了获取锁，在看看释放锁。

###### release源码
```java
//独占模式下释放锁
public final boolean release(int arg) {
        if (tryRelease(arg)) { //尝试释放锁
            Node h = head;
            if (h != null && h.waitStatus != 0)
              //唤醒后继节点
              unparkSuccessor(h);
            return true;
        }
        return false;
    }
```

###### tryRelease源码
```java
protected boolean tryRelease(int arg) {
        throw new UnsupportedOperationException();
    }
```
同现程获取锁的逻辑一样，释放锁的方法也是默认抛出异常，具体逻辑由子类实现。

至此，独占锁的获取和释放已全部解析完毕，下面看下共享API。

##### 共享API解析
###### tryAcquireShared源码

```java
protected int tryAcquireShared(int arg) {
    throw new UnsupportedOperationException();
}
```

###### tryReleaseShared源码

```java
protected boolean tryReleaseShared(int arg) {
    throw new UnsupportedOperationException();
}
```
可知，在获取或者释放共享锁的方法逻辑中，全部都是以抛出异常来实现，具体逻辑由子类实现。在以后在讲解相应子类时，在具体讲解。


#### 参考文档
[深度解析Java 8：JDK1.8 AbstractQueuedSynchronizer的实现分析（上）](http://www.infoq.com/cn/articles/jdk1.8-abstractqueuedsynchronizer)

[Jdk1.6 JUC源码解析(6)-locks-AbstractQueuedSynchronizer](http://brokendreams.iteye.com/blog/2250372)