---
title: Semaphore(信号量)源码学习
date: 2018-01-04 15:22:55
tags: java并发锁
---

#### 前言
Semaphore（信号量）是用来`控制同时访问特定资源的线程数量`，它通过协调各个线程，以保证合理的使用公共资源。Semaphore可以用于做`流量控制`，特别公用资源有限的应用场景，比如数据库连接。

#### 使用

<!-- more -->

```java
/**
 * 信号量Semaphore使用
 */
public class SemaphoreTest {

    private static final int THREAD_COUNT = 30;
    private static ExecutorService threadPool = new ThreadPoolExecutor(THREAD_COUNT, 200, 60,
            TimeUnit.SECONDS, new LinkedBlockingQueue<>(1024), Executors.defaultThreadFactory(), new ThreadPoolExecutor.AbortPolicy());

    /**
     * 10个数据库连接
     */
    private static Semaphore semaphore = new Semaphore(10);

    public static void main(String[] args) {
        for (int index = 0; index < THREAD_COUNT; index++) {
            threadPool.execute(() -> {
                try {
                    semaphore.acquire();
                    TimeUnit.SECONDS.sleep(2);
                    System.out.println("操作数据库");
                } catch (InterruptedException e) {
                    e.printStackTrace();
                } finally {
                    semaphore.release();
                }

            });
        }

        threadPool.shutdown();
        while (!threadPool.isTerminated()) {
            //System.out.println("任务执行中，请稍后。。。");
        }

        System.out.println("执行完成");
    }
}
```

#### 源码解析
##### 数据结构

```java
//计数信号量
public class Semaphore implements java.io.Serializable {
    
    /** All mechanics via AbstractQueuedSynchronizer subclass */
    private final Sync sync;
    
    // 内部类，继承自AQS
    abstract static class Sync extends AbstractQueuedSynchronizer {}
    
    //非公平策略获取资源许可
    static final class NonfairSync extends Sync {}
    
    //公平策略获取资源许可
    static final class FairSync extends Sync {}
}
```
Semaphore 的数据结构很简单，和之前分析的 [ReentrantLock源码](https://houlong123.github.io/2017/09/25/java并发学习之Reentrantlock学习/) 的内部数据结构一样，类内部总共存在Sync、NonfairSync、FairSync三个类。其底层是基于[AbstractQueuedSynchronizer](https://houlong123.github.io/2017/09/11/java并发学习之AbstractQueuedSyncchronizer学习/)来实现的。


###### Sync类

```java
// 内部类，继承自AQS
abstract static class Sync extends AbstractQueuedSynchronizer {
    private static final long serialVersionUID = 1192457210091910933L;

    Sync(int permits) {
        // 设置状态数
        setState(permits);
    }

    // 获取许可
    final int getPermits() {
        return getState();
    }

    // 共享模式下非公平策略获取
    final int nonfairTryAcquireShared(int acquires) {
        //无限循环
        for (;;) {
            // 获取许可数
            int available = getState();
            //剩余的许可
            int remaining = available - acquires;

            // 许可小于0或者比较并且设置状态成功
            if (remaining < 0 ||
                compareAndSetState(available, remaining))
                return remaining;
        }
    }

    // 共享模式下进行释放
    protected final boolean tryReleaseShared(int releases) {
        //无限循环
        for (;;) {
            //释放许可
            int current = getState();
            // 可用的许可
            int next = current + releases;
            if (next < current) // overflow
                throw new Error("Maximum permit count exceeded");
            //CAS设置许可数
            if (compareAndSetState(current, next))
                return true;
        }
    }

    // 根据指定的缩减量减小可用许可的数目
    final void reducePermits(int reductions) {
        for (;;) {
            int current = getState();
            int next = current - reductions;
            if (next > current) // underflow
                throw new Error("Permit count underflow");
            if (compareAndSetState(current, next))
                return;
        }
    }

    // 获取并返回立即可用的所有许可
    final int drainPermits() {
        for (;;) {
            int current = getState();
            if (current == 0 || compareAndSetState(current, 0))
                return current;
        }
    }
}
```
在 `Sync内部抽象类` 内部实现很简单，只有一些代码。并实现了父类AQS中的`tryReleaseShared()`方法，该方法的功能是：释放共享模式下的锁。由此可知，Semaphore内部无论是公平锁还是非公平锁，都使用了`共享锁`机制。

###### NonfairSync源码

```java
//非公平策略获取资源许可
static final class NonfairSync extends Sync {
    private static final long serialVersionUID = -2694183684443567898L;

    NonfairSync(int permits) {
        super(permits);
    }

    // 共享模式下获取
    protected int tryAcquireShared(int acquires) {
        return nonfairTryAcquireShared(acquires);
    }
}
```
NonfairSync 为非公平锁。其获取共享锁方法`tryAcquireShared()` 方法，内部调用了父类`Sync` 中的`nonfairTryAcquireShared()`方法。


###### FairSync源码

```java
//公平策略获取资源许可
static final class FairSync extends Sync {
    private static final long serialVersionUID = 2014338818796000944L;

    FairSync(int permits) {
        super(permits);
    }

    protected int tryAcquireShared(int acquires) {
        for (;;) {
            // 同步队列中存在其他节点，获取许可失败
            if (hasQueuedPredecessors())
                return -1;
            int available = getState();
            int remaining = available - acquires;
            if (remaining < 0 ||
                compareAndSetState(available, remaining))
                return remaining;
        }
    }
}
```
FairSync 为公平锁。该锁与 非公平锁`NonfairSync`的不同之处在于：在获取锁之前，公平锁会判断同步队列中是否有其他节点也在等待获锁，如果有，则表明获取锁失败，返回失败信息；否则和非公平锁一样，直接去获取锁。

##### 构造函数

```java
public Semaphore(int permits) {
    //默认使用非公平策略锁
    sync = new NonfairSync(permits);
}

public Semaphore(int permits, boolean fair) {
    sync = fair ? new FairSync(permits) : new NonfairSync(permits);
}
```
Semaphore 类有两个构造函数，默认情况下，使用非公平锁机制。其中的参数`permits`回去初始化父类ASQ中的`state`值，表示可访问资源数。

##### 获取锁

###### acquire源码

```java
public void acquire() throws InterruptedException {
    sync.acquireSharedInterruptibly(1);
}
```
该方法功能是：可中断的获取锁。内部实际是由内部类`Sync` 对象完成相应逻辑。在之前讲解AQS源码的时候，有分析过，`acquireSharedInterruptibly()` 方法是一个模板方法，内部会去调用由AQS的子类实现的`tryAcquireShared()`方法。所以在Semaphore类中，会调用内部的锁实现机制的相应逻辑。

###### acquireUninterruptibly源码

```java
public void acquireUninterruptibly() {
    sync.acquireShared(1);
}
```
该方法的功能：不可中断的获取锁。内部实现逻辑与上面的`acquire()`方法大致一样。最终都是调用Semaphore类中的内部锁实现机制的相应逻辑。

##### 释放锁

```java
public void release() {
    sync.releaseShared(1);
}
```
该方法功能：释放锁。最终的实现逻辑是调用Semaphore类中的内部锁实现机制的相应逻辑。


Semaphore 类的内部实现逻辑非常简单。其数据结构依托于AQS的数据结构。因此在之前分析了AQS的相关知识后，学习Semaphore的内部实现就非常容易了。