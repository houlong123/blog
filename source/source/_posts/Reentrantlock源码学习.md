---
title: Reentrantlock源码学习
date: 2017-09-25 14:22:06
tags: java并发锁
---

#### 前言
在上文 [AbstractQueuedSyncchronizer学习](https://houlong123.github.io/2017/09/11/java%E5%B9%B6%E5%8F%91%E5%AD%A6%E4%B9%A0%E4%B9%8BAbstractQueuedSyncchronizer%E5%AD%A6%E4%B9%A0/) 中，对AQS进行了相应的学习，知道AQS是Java并发包的一个同步基础机制。该接口有几个常用的子类，本文章主要对其中最常用的子类`ReentrantLock`类进行学习，其他子类后续再讲。

#### ReentrantLock源码学习

##### 内部数据结构

```java
//可重入独占锁，实现了Lock接口
public class ReentrantLock implements Lock, java.io.Serializable {

    private final Sync sync;
    
    //锁的同步控制基础。子类分为公平锁和非公平锁，使用AQS中的state代表持有的锁数
    abstract static class Sync extends AbstractQueuedSynchronizer {}
    
    //非公平锁实现
    static final class NonfairSync extends Sync {}
    
    //公平锁实现
    static final class FairSync extends Sync {}
}
```
由类实现可知，该类持有一个`Sync`对象，提供所有的同步机制。

<!-- more -->

##### 构造函数

```java
public class ReentrantLock implements Lock, java.io.Serializable {
    
    //无参构造函数，默认提供非公平锁机制
    public ReentrantLock() {
        sync = new NonfairSync();
    }
    
    //有参构造函数，fair = true,提供公平锁机制;fair = false,提供非公平锁机制
    public ReentrantLock(boolean fair) {
        sync = fair ? new FairSync() : new NonfairSync();
    }
}
```
`ReentrantLock`类有两个构造函数，一个无参构造函数，默认提供一个非公平锁机制；一个boolean类型的构造函数，根据boolea值确实提供何种锁机制。

##### 获取锁
对于ReentrantLock，使用过的同学应该都知道，它的通常用法为：

```java
try {
    reentrantLock.lock();
    //do something
} finally {
    reentrantLock.unlock();
}
```
ReentrantLock会保证 `do something`在同一时间只有一个线程在执行这段代码。其余线程会被挂起，直到获取锁。由此可知，ReentrantLock实现的就是一个 **`独占锁`** 的功能：有且只有一个线程获取到锁，其余线程全部挂起，直到该拥有锁的线程释放锁，被挂起的线程被唤醒重新开始竞争锁。

###### lock源码

```java
/**
  * Acquires the lock.
  *
  * <p>Acquires the lock if it is not held by another thread and returns
  * immediately, setting the lock hold count to one.
  *
  * <p>If the current thread already holds the lock then the hold
  * count is incremented by one and the method returns immediately.
  *
  * <p>If the lock is held by another thread then the
  * current thread becomes disabled for thread scheduling
  * purposes and lies dormant until the lock has been acquired,
  * at which time the lock hold count is set to one.
  */
public void lock() {
    sync.lock();
}
```
由方法注释可知，`ReentrantLock`为可重入锁，内部属性`state`代表了加锁的次数。由源码可知，ReentrantLock获取锁的具体实现由其内部类`Sync`的子类提供。内部类`Sync`的子类有两种：FairSync(公平锁)和NonfairSync(非公平锁)。至于公平锁和非公平锁，唯一的区别是在获取锁的方式。(是直接去获取锁，还是进入队列排队)。下面分析ReentrantLock中公平锁的实现。

###### FairSync类中lock源码
```java
//FairSync类中的lock源码
final void lock() {
    acquire(1);
}
```

调用到了AQS的acquire方法：

###### AQS中acquire源码
```java
//AQS中acquire源码
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```
该方法的具体分析，可以查看 [AQS源码学习](https://houlong123.github.io/2017/09/11/java%E5%B9%B6%E5%8F%91%E5%AD%A6%E4%B9%A0%E4%B9%8BAbstractQueuedSyncchronizer%E5%AD%A6%E4%B9%A0/)。可知`tryAcquire`方法是要由子类去实现的。

###### FairSync类中tryAcquire源码

```java
        protected final boolean tryAcquire(int acquires) {  //返回true，表示获取到锁
            final Thread current = Thread.currentThread();
            int c = getState();//获取父类AQS中的标志位
            if (c == 0) {  //说明目前没有持有锁对象，尝试获取锁对象
            
                if (!hasQueuedPredecessors() &&     //由于是公平锁内的实现，因此在获取锁之前还需先判断队列中是否有其他已排队的线程
                        //如果队列中没有其他线程  说明没有线程正在占有锁！
                    compareAndSetState(0, acquires)) {      //修改一下状态位，注意：这里的acquires是在lock的时候传递来的，从上面的图中可以知道，这个值是写死的1
                    setExclusiveOwnerThread(current);  //如果通过CAS操作将状态为更新成功则代表当前线程获取锁，因此，将当前线程设置到AQS的一个变量中，说明这个线程拿走了锁。
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) {
                //如果不为0 意味着，锁已经被拿走了，但是，因为ReentrantLock是重入锁，
                //是可以重复lock,unlock的，只要成对出现就行。这里还要再判断一次 获取锁的线程是不是当前请求锁的线程。
                int nextc = c + acquires;
                if (nextc < 0)
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }
```
功能分析：
    
在方法内部首先获取AQS中的标志位`state`，在前文已提到过，`state`在AQS类的不同子类中，有不同的意义。ReentrantLock中，表示加锁的次数。如果`state == 0`，说明还没有线程持有锁，因此会去直接获取锁，由于是公平锁实现，因此在获取锁之前还要先判断队列中是否有其他已排队的线程，如果没有的话，则尝试修改`state`值，修改成功，表示获取锁成功，返回`true`；如果`state 不为 0`，表示已有线程持有了锁，因为 **`ReentrantLock为可重入锁`**，因此判断当前线程是否为持有锁的线程，是的话，state值+1，返回`true`。

###### hasQueuedPredecessors源码
```java
//return true if there is a queued thread preceding the
//current thread, and return false if the current thread
//is at the head of the queue or the queue is empty
public final boolean hasQueuedPredecessors() {
    Node t = tail; 
    Node h = head;
    Node s;
    return h != t &&
        ((s = h.next) == null || s.thread != Thread.currentThread());
}
```
至此，ReentrantLock获取锁的公平锁逻辑已分析完，非公平锁逻辑和公平锁逻辑差不多，只是少了排队的逻辑。下面学习一下释放锁的逻辑。

###### unlock源码
```java
public void unlock() {
    sync.release(1);
}
```
与获取锁的逻辑一样，内部实现同样由`Sync`内部类提供具体实现。调用到了AQS中的release方法

###### AQS中release源码
```java
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
`tryRelease`方法具体逻辑由子类实现。

###### Sync类tryRelease源码

```java
protected final boolean tryRelease(int releases) {
            int c = getState() - releases;
            if (Thread.currentThread() != getExclusiveOwnerThread())    //如果释放的线程和获取锁的线程不是同一个，抛出非法监视器状态异常。
                throw new IllegalMonitorStateException();
            boolean free = false;
            if (c == 0) { //因为是重入的关系，不是每次释放锁c都等于0，直到最后一次释放锁时，才通知AQS不需要再记录哪个线程正在获取锁。
                free = true;
                setExclusiveOwnerThread(null);
            }
            setState(c);
            return free;
        }
```
因为是可重入锁，因此在`state == 0`后，需要通知AQS无需再记录哪个线程正在获取锁。即把当前拥有锁的线程置为空。**无论是公平锁还是非公平锁，释放锁的逻辑都相同**。

#### 参考文章
[深度解析Java 8：JDK1.8 AbstractQueuedSynchronizer的实现分析（上）](http://www.infoq.com/cn/articles/jdk1.8-abstractqueuedsynchronizer)
