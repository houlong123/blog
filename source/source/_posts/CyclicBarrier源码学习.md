---
title: CyclicBarrier源码学习
date: 2018-01-08 15:24:43
tags: java并发锁
---

#### 前言
CyclicBarrier的字面意思是：可循环使用（cyclic）的屏障（barrier）。通过它可以实现让一组线程等待至某个状态之后再全部同时执行。

#### 使用

<!-- more -->

```java
/**
 * CyclicBarrier类使用
 *
 * CyclicBarrier为可循环使用的屏障。让一组线程到达屏障时被阻塞，直到最后一个线程到达屏障时，所有被屏障拦截的线程才会继续运行。
 */
public class CyclicBarrierTest implements Runnable {
    // CyclicBarrir应用场景
    private CyclicBarrier cyclicBarrier = new CyclicBarrier(4, this);
    /**
     * 保存每个sheet计算出的银流结果
     */
    private ConcurrentHashMap<String, Integer> sheetBankWaterMap = new ConcurrentHashMap<>(4);
    /**
     * 假如有4个sheet，启动4个线程
     */
    private Executor executor = Executors.newFixedThreadPool(4);

    private void count() {
        for (int i = 0; i < 4; i++) {
            executor.execute(new Runnable() {
                @Override
                public void run() {
                    sheetBankWaterMap.put(Thread.currentThread().getName(), 1);     //模拟银流计算结果，并保存

                    //银流计算完成，插入一个屏障
                    try {
                        cyclicBarrier.await();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    } catch (BrokenBarrierException e) {
                        e.printStackTrace();
                    }
                }
            });
        }
    }

    @Override
    public void run() {
        int sheetCount = 0;
        for (Map.Entry<String, Integer> entry : sheetBankWaterMap.entrySet()) {
            sheetCount += entry.getValue();
        }

        System.out.println("银流数目：" + sheetCount);
    }

    public static void main(String[] args) {
        CyclicBarrierTest test = new CyclicBarrierTest();
        test.count();
    }

}
```

#### 源码分析
##### 构造函数

```java
public CyclicBarrier(int parties) {
    this(parties, null);
}

public CyclicBarrier(int parties, Runnable barrierAction) {
    if (parties <= 0) throw new IllegalArgumentException();
    this.parties = parties;
    this.count = parties;
    //当给定数量的线程处于等待时，会触发屏障CyclicBarrier，当屏障被触发时执行该命令
    this.barrierCommand = barrierAction;
}
```
从 CyclicBarrier 的源码中，我们可以看到该类有两个构造函数，最终是使用`CyclicBarrier(int parties, Runnable barrierAction)`。其中的 Runnable 类型的参数是在：`当给定数量的线程处于等待时，会触发屏障CyclicBarrier，当屏障被触发时执行该命令`。


##### await源码

```java
public int await() throws InterruptedException, BrokenBarrierException {
    try {
        return dowait(false, 0L);
    } catch (TimeoutException toe) {
        throw new Error(toe); // cannot happen
    }
}

//超时等待await
public int await(long timeout, TimeUnit unit)
    throws InterruptedException,
           BrokenBarrierException,
           TimeoutException {
    return dowait(true, unit.toNanos(timeout));
}
```
在 CyclicBarrier 中，无论是带有超时的await还是没有带超时的await方法，最终调用的都是 dowait() 方法。
 
 
 ##### dowait源码
 
```java
private int dowait(boolean timed, long nanos)
    throws InterruptedException, BrokenBarrierException,
           TimeoutException {
    final ReentrantLock lock = this.lock;
    // 1. 获取 ReentrantLock
    lock.lock();
    try {
        final Generation g = generation;

        // 2. 判断 generation 是否已经 broken
        if (g.broken)
            throw new BrokenBarrierException();

        // 3. 判断线程是否中断, 中断后就 breakBarrier
        if (Thread.interrupted()) {
            breakBarrier();
            throw new InterruptedException();
        }

        //更新已经到达 barrier 的线程数
        int index = --count;
        if (index == 0) {  // tripped   当所有线程到达了 barrier时，触发屏障
            boolean ranAction = false;
            try {
                final Runnable command = barrierCommand;
                if (command != null)    //最后一个线程到达 barrier, 执行 command
                    command.run();
                ranAction = true;
                nextGeneration();   //更新 generation
                return 0;
            } finally {
                if (!ranAction)
                    breakBarrier();
            }
        }

        // loop until tripped, broken, interrupted, or timed out
        for (;;) {
            try {
                if (!timed)
                    trip.await();   //没有进行 timeout 的 await，其中，在Condition.await()的时候，会释放已持有的锁
                else if (nanos > 0L)
                    nanos = trip.awaitNanos(nanos); //进行 timeout 方式的等待
            } catch (InterruptedException ie) {
                if (g == generation && ! g.broken) {
                    breakBarrier();
                    throw ie;
                } else {
                    // We're about to finish waiting even if we had not
                    // been interrupted, so this interrupt is deemed to
                    // "belong" to subsequent execution.
                    Thread.currentThread().interrupt();
                }
            }

            if (g.broken)
                throw new BrokenBarrierException();

            //所有线程到达 barrier 直接返回
            if (g != generation)
                return index;

            if (timed && nanos <= 0L) {
                breakBarrier();
                throw new TimeoutException();
            }
        }
    } finally {
        lock.unlock();
    }
}
```
该方法是 CyclicBarrier 类的核心。逻辑为：
1. 获取可重入锁。成功获取，执行下一步；获取失败，等待
2. 判断屏障是否已经`broken`。是，抛出`BrokenBarrierException`异常；否则，执行下一步。
3. 判断当前线程是否被中断。已中断，则调用breakBarrier()方法将屏障`broken`，然后抛出`InterruptedException`异常；否则，执行下一步。
4. 更新已经到达屏障（barrier）的线程数`index = --count`。
5. 如果`index==0`，则说明所以线程都已到达屏障，则执行`barrierCommand`任务，并更新generation，然后返回，方法结束。
6. 如果`index!=0`，则进入for循环。
    1. 调用Condition类的相关方法将线程进去等待状态
    2. 判断generation的状态及是否等待已超时。

##### breakBarrier源码

```java
// 当某个线程被中断 / 等待超时 则将 broken = true, 并且唤醒所有等待中的线程
private void breakBarrier() {
    generation.broken = true;
    count = parties;
    trip.signalAll();
}
```

##### nextGeneration源码

```java
//生成下一个 generation
private void nextGeneration() {
    // signal completion of last generation
    // 唤醒所有等待的线程来获取 AQS 的state的值
    trip.signalAll();
    // set up next generation
    // 重新赋值计算器
    count = parties;
    // 重新初始化 generation
    generation = new Generation();
}
```


##### isBroken源码

```java
//判断 barrier 是否 broken = true
public boolean isBroken() {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        return generation.broken;
    } finally {
        lock.unlock();
    }
}
```
该方法用来判断阻塞的线程是否被中断

##### reset源码

```java
//将屏障重置为初始状态。
public void reset() {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        breakBarrier();   // break the current generation
        nextGeneration(); // start a new generation
    } finally {
        lock.unlock();
    }
}
```

##### getNumberWaiting源码

```java
//Returns the number of parties currently waiting at the barrier.
public int getNumberWaiting() {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        return parties - count;
    } finally {
        lock.unlock();
    }
}
```
该方法用于获取在屏障处阻塞的线程。

#### 参考文章
[CyclicBarrier 源码分析 (基于Java 8)](https://www.jianshu.com/p/98afdcfdb37c)