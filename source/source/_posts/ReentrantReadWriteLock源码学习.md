---
title: ReentrantReadWriteLock源码学习
date: 2017-09-28 14:30:40
tags: java并发锁
---

#### 前言
在学习 [ReentrantLock锁](https://houlong123.github.io/2017/09/25/java%E5%B9%B6%E5%8F%91%E5%AD%A6%E4%B9%A0%E4%B9%8BReentrantlock%E5%AD%A6%E4%B9%A0/) 的时候，我们知道，ReentrantLock实现了标准的互斥重入锁，任一时刻只有一个线程能获得锁。而现实编程中，读操作是比较普遍的，写操作则相对很少发生。如果使用互斥锁，那么即使都是读操作，也只有一个线程能获得锁，其他的读都得阻塞。这样显然不利于提供系统的并发量。本文即将进行讲解的 `读写锁ReentrantReadWriteLock` 就是为了解决这种读多写少情况。<font color='red'>在读-写锁的实现加锁策略中，允许多个读操作同时进行，但每次只允许一个写操作。</font>

#### 使用

<!-- more -->

```java
class RWDictionary {
    private final Map<String, Data> m = new TreeMap<String, Data>();
    Object data;
    volatile boolean cacheValid;
    private final ReentrantReadWriteLock rwl = new ReentrantReadWriteLock();
    private final Lock r = rwl.readLock();
    private final Lock w = rwl.writeLock();
 
    //读锁使用
    public Data get(String key) {
      r.lock();
      try { 
        return m.get(key); 
      } finally { 
        r.unlock(); 
      }
    }
    
    //写锁使用
    public Data put(String key, Data value) {
      w.lock();
      try { return m.put(key, value); }
      finally { w.unlock(); }
    }
    
    //读写锁联合使用
    void processCachedData() {
      rwl.readLock().lock();
      if (!cacheValid) {
        // Must release read lock before acquiring write lock
        rwl.readLock().unlock();
        rwl.writeLock().lock();
        try {
          // Recheck state because another thread might have
          // acquired write lock and changed state before we did.
          if (!cacheValid) {
            data = ...
            cacheValid = true;
          }
          // Downgrade by acquiring read lock before releasing write lock
          rwl.readLock().lock();
        } finally {
          rwl.writeLock().unlock(); // Unlock write, still hold read
        }
      }
 
      try {
        use(data);
      } finally {
        rwl.readLock().unlock();
      }
    }
}
```

#### 源码解析

##### 内部数据结构

```java
public class ReentrantReadWriteLock
        implements ReadWriteLock, java.io.Serializable {

    //读锁对象
    private final ReadLock readerLock;
    //写锁对象
    private final WriteLock writerLock;   

    final Sync sync;
    
    private static final sun.misc.Unsafe UNSAFE;
    private static final long TID_OFFSET;
    
    abstract static class Sync extends AbstractQueuedSynchronizer {
        
        /*
         * Read vs write count extraction constants and functions.
         * Lock state is logically divided into two unsigned shorts:
         * The lower one representing the exclusive (writer) lock hold count,
         * and the upper the shared (reader) hold count.
         */
        //AQS使用一个int型来保存状态，在读写锁中，状态的高16位用作读锁，低16位用作写锁
        static final int SHARED_SHIFT   = 16;
        
        //由于读锁用高位部分，读锁个数加1，其实是状态值加 2^16
        static final int SHARED_UNIT    = (1 << SHARED_SHIFT);      //二进制表示：0000 0000 0000 0001 0000 0000 0000 0000
        static final int MAX_COUNT      = (1 << SHARED_SHIFT) - 1;
        
        /**写锁的掩码，用于状态的低16位有效值 */
        static final int EXCLUSIVE_MASK = (1 << SHARED_SHIFT) - 1;  //二进制表示：0000 0000 0000 0000 1111 1111 1111 1111
        
        //当前线程持有的可重入读锁数
        private transient ThreadLocalHoldCounter readHolds;
        
        /** 
        * 最近一个成功获取读锁的线程的计数。这省却了ThreadLocal查找， 
        * 通常情况下，下一个释放线程是最后一个获取线程。这不是 volatile 的， 
        * 因为它仅用于试探的，线程进行缓存也是可以的 
        * （因为判断是否是当前线程是通过线程id来比较的）。 
        */ 
        private transient HoldCounter cachedHoldCounter;
        
        //firstReader是第一个获得读锁的线程
        private transient Thread firstReader = null;
        
        //firstReaderHoldCount是firstReader的重入计数
        private transient int firstReaderHoldCount;
        
        //每个线程持有读锁的计数
        static final class HoldCounter {
            int count = 0;
            // Use id, not reference, to avoid garbage retention
            final long tid = getThreadId(Thread.currentThread());
        }
        
        /**
         * 采用继承是为了重写 initialValue 方法，这样就不用进行这样的处理：
         * 如果ThreadLocal没有当前线程的计数，则new一个，再放进ThreadLocal里。
         * 可以直接调用 get。
         * */
        static final class ThreadLocalHoldCounter
            extends ThreadLocal<HoldCounter> {
            public HoldCounter initialValue() {
                return new HoldCounter();
            }
        }
        
    }
    
    //非公平锁机制
    static final class NonfairSync extends Sync {}
    
    //公平锁机制
    static final class FairSync extends Sync {}
    
    //读锁。共享锁。同一时刻可以被多个线程获得
    public static class ReadLock implements Lock, java.io.Serializable {
        private final Sync sync;
    }
    
    //写锁。独占锁
    public static class WriteLock implements Lock, java.io.Serializable {
        private final Sync sync;
    }
}
```
由源码可知，该类持有一个Sync类，继承自AQS，提供所有的同步机制。这是所有AQS子类的共性。之前有提过，AQS类中的`state`字段在不同锁中，代表不同含义。在ReentrantReadWriteLock类中，`state`字段<font color='red'> `高16位用作读锁，低16位用作写锁` </font>。所以无论是读锁还是写锁最多只能被持有65535次。

由于`state`字段同时代表了读锁和写锁状态，因此可知以下情况：

+ 由于读写锁共享状态，所以状态不为0时，只能说明是有锁，可能是读锁，也可能是写锁；

+ 读锁是高16为表示的，所以读锁加1，就是状态的高16位加1，低16位不变，所以要加的不是1，而是2^16，减一同样是这样。

+ 写锁用低16位表示，要获得写锁的次数，要用状态&2^16-1，结果的高16位全为0，低16位就是写锁被持有的次数。
+ 获取读锁的次数，要用 状态 >>> 16，结果为原先的高16位变为低16位，高16位使用0填充。


读写锁是通过内部两个锁：`readerLock` 和 `writerLock` 实现的，同样分为公平锁和非公平锁。

##### 构造函数

```java
//无参构造函数
public ReentrantReadWriteLock() {
    this(false);
}

//有参构造函数
public ReentrantReadWriteLock(boolean fair) {
    sync = fair ? new FairSync() : new NonfairSync();
    readerLock = new ReadLock(this);
    writerLock = new WriteLock(this);
}
```
由源码可知，ReentrantReadWriteLock使用无参数构造函数时，使用的是非公平锁机制。

##### 读锁获取

###### lock源码
```java
//ReadLock类中获取锁
public void lock() {
    sync.acquireShared(1);
}
```
由源码可知，在ReadLock类中获取读锁的时候，内部会调用Sync类的父类AQS中的acquireShared方法。

###### acquireShared源码

```java
public final void acquireShared(int arg) {
    if (tryAcquireShared(arg) < 0)
        doAcquireShared(arg);
}
```
acquireShared()方法在讲解 [AQS](https://houlong123.github.io/2017/09/11/java%E5%B9%B6%E5%8F%91%E5%AD%A6%E4%B9%A0%E4%B9%8BAbstractQueuedSyncchronizer%E5%AD%A6%E4%B9%A0/) 的时候有介绍，内部tryAcquireShared()方法只是简单的抛出了异常，具体逻辑由子类去实现。


###### tryAcquireShared源码

```java
//ReentrantReadWriteLock类中tryAcquireShared
protected final int tryAcquireShared(int unused) {
    /*
     * Walkthrough:
     * 1. If write lock held by another thread, fail.
     * 2. Otherwise, this thread is eligible for
     *    lock wrt state, so ask if it should block
     *    because of queue policy. If not, try
     *    to grant by CASing state and updating count.
     *    Note that step does not check for reentrant
     *    acquires, which is postponed to full version
     *    to avoid having to check hold count in
     *    the more typical non-reentrant case.
     * 3. If step 2 fails either because thread
     *    apparently not eligible or CAS fails or count
     *    saturated, chain to version with full retry loop.
     */
    Thread current = Thread.currentThread();
    int c = getState();
    //持有写锁的线程可以获得读锁
    if (exclusiveCount(c) != 0 &&   //获取写锁数
        getExclusiveOwnerThread() != current)
        return -1;  //写锁被占用，且不是由当前线程持有，返回-1

    /**
      执行到这里有两种情况：
         1：有写锁且写锁由当前线程持有
         2: 没有写锁，读锁可有可无(即允许多个读操作同时进行)
     */
    //获得读锁的数量
    int r = sharedCount(c);
    if (!readerShouldBlock() &&
        r < MAX_COUNT &&
        compareAndSetState(c, c + SHARED_UNIT)) {  /** 如果不用阻塞，且没有溢出，则使用CAS修改状态，并且修改成功。因为高16位表示读锁，要修改高16位的状态，所以要加上2^16 */
        if (r == 0) { //这是第一个占有读锁的线程，设置firstReader
            firstReader = current;
            firstReaderHoldCount = 1;
        } else if (firstReader == current) { //重入计数加1
            firstReaderHoldCount++;
        } else {
            // 非 firstReader 读锁重入计数更新
            //将cachedHoldCounter设置为当前线程
            HoldCounter rh = cachedHoldCounter;  //最近一个成功获取读锁的线程的计数
            if (rh == null || rh.tid != getThreadId(current))
                cachedHoldCounter = rh = readHolds.get();
            else if (rh.count == 0)
                readHolds.set(rh);
            rh.count++;
        }
        return 1;
    }
    //获取读锁失败，放到循环里重试
    return fullTryAcquireShared(current);
}
```
ReentrantReadWriteLock类中的tryAcquireShared()方法，里面逻辑为：
1. 如果当前写锁被其他线程持有，则获取读锁失败；
2. 写锁空闲，或者写锁被当前线程持有，在公平策略下，它可能需要阻塞，那么tryAcquireShared()就可能失败，则需要进入队列等待；如果是`非公平策略，且头结点的next结点不是写操作的话(下文分析readerShouldBlock方法)`，会尝试获取锁，使用CAS修改状态，修改成功，则获得读锁，否则也会进入同步队列等待；

###### exclusiveCount源码

```java
// 写锁的计数，也就是它的重入次数,c的低16位
//EXCLUSIVE_MASK 的二进制表示：0000 0000 0000 0000 1111 1111 1111 1111

static int exclusiveCount(int c) { 
    return c & EXCLUSIVE_MASK; 
}
```
该方法的功能为获取写锁数

###### sharedCount源码

```java
// 读锁的计数，也就是它的重入次数,c的高16位
static int sharedCount(int c)    { 
    return c >>> SHARED_SHIFT;  // >>>：无符号右移，高位补0。向右移1位相当于是把该数除以2。
}
```
该方法的功能为获取读锁数

###### readerShouldBlock源码
readerShouldBlock方法有两个实现版本，一个是公平锁策略，一个为非公平锁策略。

+ 公平策略版本
 
    ```java
    final boolean readerShouldBlock() {
        return hasQueuedPredecessors();
    }
    ```
    就是判断队列中是否有正在等待获取锁的线程，有的话，说明当前线程应该放进等待队列中。

+ 非公平策略版本

    ```java
    final boolean readerShouldBlock() {
        /* As a heuristic to avoid indefinite writer starvation,
         * block if the thread that momentarily appears to be head
         * of queue, if one exists, is a waiting writer.  This is
         * only a probabilistic effect since a new reader will not
         * block if there is a waiting writer behind other enabled
         * readers that have not yet drained from the queue.
         */
        return apparentlyFirstQueuedIsExclusive();
    }
    
    /**
     * Returns {@code true} if the apparent first queued thread, if one
     * exists, is waiting in exclusive mode.  If this method returns
     * {@code true}, and the current thread is attempting to acquire in
     * shared mode (that is, this method is invoked from {@link
     * #tryAcquireShared}) then it is guaranteed that the current thread
     * is not the first queued thread.  Used only as a heuristic in
     * ReentrantReadWriteLock.
     */
    final boolean apparentlyFirstQueuedIsExclusive() {
        Node h, s;
        return (h = head) != null &&
            (s = h.next)  != null &&
            !s.isShared()         &&
            s.thread != null;
    }
    ```
    由上面源码可知，为了防止产生写锁饥饿，在获取读锁的非公平策略下，如果正在等待排队的第一个线程为独占模式(即写锁)的话，则当前正在获取读锁的线程要阻塞。

在tryAcquireShared()方法中，如果获取锁失败，则会调用fullTryAcquireShared()方法，循环获取锁。

###### fullTryAcquireShared源码

```java
final int fullTryAcquireShared(Thread current) {
    /*
     * This code is in part redundant with that in
     * tryAcquireShared but is simpler overall by not
     * complicating tryAcquireShared with interactions between
     * retries and lazily reading hold counts.
     */
    HoldCounter rh = null;
    for (;;) {
        int c = getState();
        if (exclusiveCount(c) != 0) {
            if (getExclusiveOwnerThread() != current)
                return -1;
            // else we hold the exclusive lock; blocking here
            // would cause deadlock.
        } else if (readerShouldBlock()) {
            // Make sure we're not acquiring read lock reentrantly
            if (firstReader == current) {
                // assert firstReaderHoldCount > 0;
            } else {
                if (rh == null) {
                    rh = cachedHoldCounter;
                    if (rh == null || rh.tid != getThreadId(current)) {
                        rh = readHolds.get();
                        if (rh.count == 0)
                            readHolds.remove();
                    }
                }
                if (rh.count == 0)
                    return -1;
            }
        }
        if (sharedCount(c) == MAX_COUNT)
            throw new Error("Maximum lock count exceeded");
        if (compareAndSetState(c, c + SHARED_UNIT)) {
            if (sharedCount(c) == 0) {
                firstReader = current;
                firstReaderHoldCount = 1;
            } else if (firstReader == current) {
                firstReaderHoldCount++;
            } else {
                if (rh == null)
                    rh = cachedHoldCounter;
                if (rh == null || rh.tid != getThreadId(current))
                    rh = readHolds.get();
                else if (rh.count == 0)
                    readHolds.set(rh);
                rh.count++;
                cachedHoldCounter = rh; // cache for release
            }
            return 1;
        }
    }
}
```

##### 读锁释放

###### unlock源码

```java
public void unlock() {
    sync.releaseShared(1);
}

//内部调用父类的releaseShared()方法，与获取锁逻辑一致
public final boolean releaseShared(int arg) {
    if (tryReleaseShared(arg)) {
        doReleaseShared();
        return true;
    }
    return false;
}
```

###### tryReleaseShared源码
```java
protected final boolean tryReleaseShared(int unused) {
    Thread current = Thread.currentThread();
    if (firstReader == current) {
        // assert firstReaderHoldCount > 0;
        /**
         * 当前线程是第一个获取到锁的，如果此线程要释放锁了，则firstReader置空
         * 否则，将线程持有的锁计数减1
         */
        if (firstReaderHoldCount == 1)
            firstReader = null;
        else
            firstReaderHoldCount--;
    } else {
        HoldCounter rh = cachedHoldCounter;     //The hold count of the last thread to successfully acquire readLock
        if (rh == null || rh.tid != getThreadId(current))
            rh = readHolds.get(); //readHolds表示当前线程所持有的可重入读锁数
        int count = rh.count;
        if (count <= 1) {
            readHolds.remove();
            if (count <= 0)
                throw unmatchedUnlockException(); //如果没有持有读锁，释放是非法的
        }
        --rh.count;
    }
    //有可能其他线程也在释放读锁，所以要确保释放成功
    for (;;) {
        int c = getState();
        int nextc = c - SHARED_UNIT;
        if (compareAndSetState(c, nextc))
            // Releasing the read lock has no effect on readers,
            // but it may allow waiting writers to proceed if
            // both read and write locks are now free.
            return nextc == 0;
    }
}
```
在释放读锁的时候，会先处理 `firstReader(第一个成功获取读锁的线程)` ，
`firstReaderHoldCount（firstReader的重入计数` ，
`cachedHoldCounter(最近一个成功获取读锁的线程的计数)` ，`readHolds(当前线程所持有的可重入读锁数)`
 几个对象，如果 `firstReader == current`，则处理`firstReader`和`firstReaderHoldCount`。否则处理另外两个。在处理完后，然后在去更新 `state`值，直至成功。
 
 
 ##### 写锁获取
 
 ###### lock()
 
```java
public void lock() {
    sync.acquire(1);
}

//调用父类AQS中的acquire方法
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```

###### tryAcquire源码

```java
//ReentratReadWriteLock类中方法
protected final boolean tryAcquire(int acquires) {
    /*
     * Walkthrough:
     * 1. If read count nonzero or write count nonzero
     *    and owner is a different thread, fail.
     * 2. If count would saturate, fail. (This can only
     *    happen if count is already nonzero.)
     * 3. Otherwise, this thread is eligible for lock if
     *    it is either a reentrant acquire or
     *    queue policy allows it. If so, update state
     *    and set owner.
     */
    Thread current = Thread.currentThread();
    int c = getState();
    int w = exclusiveCount(c); //写锁被持有的次数，通过与低16位做与操作得到
    if (c != 0) { //c!=0，说明存在锁，可能是读锁，也可能是写锁
        // (Note: if c != 0 and w == 0 then shared count != 0)
        // c!=0,w==0,说明读锁存在
        //w != 0 && current != getExclusiveOwnerThread() 表示其他线程获取了写锁。
        if (w == 0 || current != getExclusiveOwnerThread())  //当w==0，表示读锁存在，则返回false，说明在线程正在读的时候，是不能进行写操作的
            return false;
        if (w + exclusiveCount(acquires) > MAX_COUNT)
            throw new Error("Maximum lock count exceeded");
        // Reentrant acquire
        //执行到这里，说明存在写锁，且由当前线程持有
        // 重入计数
        setState(c + acquires);
        return true;
    }

    //执行到这里，说明不存在任何锁
    if (writerShouldBlock() ||
        !compareAndSetState(c, c + acquires))
        return false;
    setExclusiveOwnerThread(current);
    return true;
}
```
ReentrantReadWriteLock类中的tryAcquire()方法，里面逻辑为：
1. 在获取写锁时，当存在锁时（既可能是写锁，也可能是读锁），如果是读锁，获取锁失败（读不能写）。如果是写锁，且当前线程非写锁持有线程，则获取锁失败。否则，获取锁成功。
2. 如果不存在任何锁，判断当前线程是否要阻塞，是，获取锁失败，否则CAS更新`state`值，操作成功，则成功获取锁。

###### writerShouldBlock源码
同readerShouldBlock()方法一样，分为公平策略版本和非公平策略版本。
+ 公平策略版本

    ```java
    final boolean writerShouldBlock() {
        return hasQueuedPredecessors();
    }
    ```
    就是判断队列中是否有正在等待获取锁的线程，有的话，说明当前线程应该放进等待队列中。
    
+ 非公平策略版本

    ```java
    final boolean writerShouldBlock() {
        return false; // writers can always barge
    }
    ```
    非公平策略下，获取写锁不会阻塞，直接尝试获取锁。


##### 写锁释放
 
 ###### unlock()
 
```java
public void unlock() {
    sync.release(1);
}

//父类AQS方法
public final boolean release(int arg) {
    if (tryRelease(arg)) {
        Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    return false;
}
```

###### tryRelease源码

```java
protected final boolean tryRelease(int releases) {
    if (!isHeldExclusively())
        throw new IllegalMonitorStateException();
    //状态的低16位减1，如果为0，说明写锁可用，返回true，如果不为0，说明当前线程仍然持有写锁，返回false
    int nextc = getState() - releases;
    boolean free = exclusiveCount(nextc) == 0;
    if (free)
        setExclusiveOwnerThread(null);
    setState(nextc);
    return free;
}
```
释放写锁很简单，就是状态的低16为减1，如果为0，说明写锁可用，返回true，如果不为0，说明当前线程仍然持有写锁，返回false;

#### 参考文档
[ReentrantReadWriteLock源码分析](http://blog.csdn.net/yuhongye111/article/details/39055531)