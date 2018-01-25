---
title: Fork/Join框架源码学习之ForkJoinPool学习
date: 2017-12-25 15:14:29
tags: java并发
---

#### 前言

&nbsp;&nbsp;&nbsp;&nbsp;在 [Fork/Join框架源码学习之ForkJoinTask学习](https://houlong123.github.io/2017/12/21/Fork-Join框架源码学习之ForkJoinTask学习/) 文章中，我们已经初步分析了Fork/Join框架中的一个重要类 [ForkJoinTask类](https://docs.oracle.com/javase/7/docs/api/java/util/concurrent/ForkJoinTask.html)。在本篇文章中，我们分析一下Fork/Join框架中的另一个重要类：[ForkJoinPool](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ForkJoinPool.html)。

&nbsp;&nbsp;&nbsp;&nbsp;在`ForkJoinPool`的介绍中有说：`ForkJoinPool是一种用于运行ForkJoinTasks的ExecutorService`（An ExecutorService for running ForkJoinTasks）。即`ForkJoinPool`也是一种`ExecutorService`。但由于<font color=red>工作窃取(work-stealing)</font>的缘故，该类与其他`ExecutorService`又不太一样。

&nbsp;&nbsp;&nbsp;&nbsp;创建`ForkJoinPool`对象的方式有多种，在后面的分析中，可以看到该类有多个重载的构造函数。但是在官方文档中，比较推荐使用静态方法 ` commonPool() ` 创建`ForkJoinPool`对象。


```
官方文档描述：

A static commonPool() is available and appropriate for most applications. 
The common pool is used by any ForkJoinTask that is not explicitly 
submitted to a specified pool. Using the common pool normally reduces resource 
usage (its threads are slowly reclaimed during periods of non-use, and reinstated upon subsequent use).


```

#### 源码分析

<!-- more -->

##### 数据结构

```java
@sun.misc.Contended
public class ForkJoinPool extends AbstractExecutorService {

    volatile long ctl;                   // main pool control
    
    //线程池运行状态。对应状态为下面各值。在为SHUTDOWN时，其值为负数
    volatile int runState;               // lockable status
    
    // runState bits: SHUTDOWN must be negative, others arbitrary powers of two
    private static final int  RSLOCK     = 1;
    private static final int  RSIGNAL    = 1 << 1;
    private static final int  STARTED    = 1 << 2;
    private static final int  STOP       = 1 << 29;
    private static final int  TERMINATED = 1 << 30;
    private static final int  SHUTDOWN   = 1 << 31; //负数


    //默认工作线程工厂
    public static final ForkJoinWorkerThreadFactory
            defaultForkJoinWorkerThreadFactory;
     
    //静态方法commonPool()会初始化该值      
    static final ForkJoinPool common;
    
    volatile WorkQueue[] workQueues;     // main registry
    final ForkJoinWorkerThreadFactory factory;
    final UncaughtExceptionHandler ueh;  // per-worker UEH
    
    //内部工作队列
    static final class WorkQueue {}
    
    //工作队列线程创建工厂类，该类用于创建工作线程
    public static interface ForkJoinWorkerThreadFactory {
        /**
         * Returns a new worker thread operating in the given pool.
         *
         * @param pool the pool this thread works in
         * @return the new worker thread
         * @throws NullPointerException if the pool is null
         */
        public ForkJoinWorkerThread newThread(ForkJoinPool pool);
    }

    /**
     * Default ForkJoinWorkerThreadFactory implementation; creates a
     * new ForkJoinWorkerThread.
     */
     //默认生产工作线程的工厂。在构造ForkJoinPool对象时，默认使用该工厂类
    static final class DefaultForkJoinWorkerThreadFactory
            implements ForkJoinWorkerThreadFactory {
        public final ForkJoinWorkerThread newThread(ForkJoinPool pool) {
            return new ForkJoinWorkerThread(pool);
        }
    }
}
```
从 ForkJoinPool 类的内部数据结构可知，该类最重要的属性有两个：`WorkQueue[] workQueues` 和 `ForkJoinWorkerThreadFactory factory`。这两个类中，一个是工作队列列表，因为池中会有多个工作队列；另一个是用于创建工作线程的工厂。在工作线程类`ForkJoinWorkerThread`中，我们可知该类中封装了一个工作队列和线程池。


##### ForkJoinWorkerThread 数据结构

```java
public class ForkJoinWorkerThread extends Thread {
/*
     * ForkJoinWorkerThreads are managed by ForkJoinPools and perform
     * ForkJoinTasks. For explanation, see the internal documentation
     * of class ForkJoinPool.
     *
     * This class just maintains links to its pool and WorkQueue.  The
     * pool field is set immediately upon construction, but the
     * workQueue field is not set until a call to registerWorker
     * completes. This leads to a visibility race, that is tolerated
     * by requiring that the workQueue field is only accessed by the
     * owning thread.
     *
     * Support for (non-public) subclass InnocuousForkJoinWorkerThread
     * requires that we break quite a lot of encapsulation (via Unsafe)
     * both here and in the subclass to access and set Thread fields.
     */
    final ForkJoinPool pool;                // the pool this thread works in
    final ForkJoinPool.WorkQueue workQueue; // work-stealing mechanics
}
```
由`ForkJoinWorkerThread`的源码可知，该类继续了`Thread`类，并且内部维持一个`ForkJoinPool`和`WorkQueue`对象。由文档注释可知：ForkJoinWorkerThread 由ForkJoinPool管理并执行ForkJoinTask。


##### ForkJoinWorkerThread内部主要方法

###### 构造函数

```java
protected ForkJoinWorkerThread(ForkJoinPool pool) {
    // Use a placeholder until a useful name can be set in registerWorker
    super("aForkJoinWorkerThread");
    this.pool = pool;
    this.workQueue = pool.registerWorker(this);
}

ForkJoinWorkerThread(ForkJoinPool pool, ThreadGroup threadGroup,
                     AccessControlContext acc) {
    super(threadGroup, null, "aForkJoinWorkerThread");
    U.putOrderedObject(this, INHERITEDACCESSCONTROLCONTEXT, acc);
    eraseThreadLocals(); // clear before registering
    this.pool = pool;
    this.workQueue = pool.registerWorker(this);
}
```
在构造函数中，使用是初始化了`ForkJoinPool`和`WorkQueue`对象。在初始化 WorkQueue 对象时，调用了 ForkJoinPool 类中的 `registerWorker()`方法。该方法在下文分析。 

###### run源码

```java
//重载Thread中的run，执行ForkJoinTask任务
public void run() {
    //仅在工作队列中的任务列表为空时，才会执行run方法中的实现逻辑
    if (workQueue.array == null) { // only run once
        Throwable exception = null;
        try {
            onStart();
            //调用pool的runWorker来获取任务执行
            pool.runWorker(workQueue);
        } catch (Throwable ex) {
            exception = ex;
        } finally {
            try {
                onTermination(exception);
            } catch (Throwable ex) {
                if (exception == null)
                    exception = ex;
            } finally {
                pool.deregisterWorker(this, exception);
            }
        }
    }
}
```
我们在使用多线程的时候，已经了解过，要新建一个线程，需要继承`Thread`类，然后实现`run()`方法。在线程启动后，会调用相应的`run()`方法。在`ForkJoinWorkerThread`类的`run()`方法实现中， 调用了ForkJoinPool 类中的 `runWorker()`方法。该方法在下文分析。 

##### WorkerQueue数据结构

```java
//内部工作队列
@sun.misc.Contended
static final class WorkQueue {

    static final int INITIAL_QUEUE_CAPACITY = 1 << 13;
    static final int MAXIMUM_QUEUE_CAPACITY = 1 << 26; // 64M
    // Instance fields
    volatile int scanState;    // versioned, <0: inactive; odd:scanning
    int stackPred;             // pool stack (ctl) predecessor
    int nsteals;               // number of steals
    int hint;                  // randomization and stealer index hint
    int config;                // pool index and mode
    // 队列状态
    volatile int qlock;        // 1: locked, < 0: terminate; else 0
    // 下一个出队元素的索引位
    volatile int base;         // index of next slot for poll
    // 为下一个入队元素准备的索引位
    int top;                   // index of next slot for push
    // 队列中使用数组存储元素
    ForkJoinTask<?>[] array;   // the elements (initially unallocated)
    // 队列所属的ForkJoinPool（可能为空）
    final ForkJoinPool pool;   // the containing pool (may be null)
    // 这个队列所属的归并计算工作线程。注意，工作队列也可能不属于任何工作线程
    final ForkJoinWorkerThread owner; // owning thread or null if shared
    volatile Thread parker;    // == owner during call to park; else null
    // 记录当前正在进行join等待的其它任务
    volatile ForkJoinTask<?> currentJoin;  // task being joined in awaitJoin
    // 当前正在偷取的任务
    volatile ForkJoinTask<?> currentSteal; // mainly used by helpStealer

}
```
在内部类`WorkQueue`的数据结构中，可以看到，该类内部有个数组类型的ForkJoinTask对象，用于存放待执行任务。`ForkJoinPool`类型的对象用于记录该队列所属的 ForkJoinPool。


##### WorkQueue内部主要方法

###### 构造函数

```java
WorkQueue(ForkJoinPool pool, ForkJoinWorkerThread owner) {
    this.pool = pool;
    this.owner = owner;
    // Place indices in the center of array (that is not yet allocated)
    base = top = INITIAL_QUEUE_CAPACITY >>> 1;
}
```
在下文分析中可知，在将任务插入相应工作队列后，如果队列为空，则会调用该方法进行队列初始化。

###### push源码

```java
/**
 * Pushes a task. Call only by owner in unshared queues.  (The
 * shared-queue version is embedded in method externalPush.)
 *
 * @param task the task. Caller must ensure non-null.
 * @throws RejectedExecutionException if array cannot be resized
 */
 //添加任务。 只有拥有者在非共享队列中才能调用
final void push(ForkJoinTask<?> task) {
    ForkJoinTask<?>[] a; ForkJoinPool p;
    int b = base, s = top, n;
    // 请注意，在执行task.fork时，触发push情况下，array不会为null
    // 因为在这之前workqueue中的array已经完成了初始化（在工作线程初始化时就完成了）
    if ((a = array) != null) {    // ignore if queue removed
        int m = a.length - 1;     // fenced write for task visibility
        // putOrderedObject方法在指定的对象a中，指定的内存偏移量的位置，赋予一个新的元素
        U.putOrderedObject(a, ((m & s) << ASHIFT) + ABASE, task);
        // putOrderedInt方法对当前指定的对象中的指定字段，进行赋值操作
        // 这里的代码意义是将workQueue对象本身中的top标示的位置 + 1，
        U.putOrderedInt(this, QTOP, s + 1);
        if ((n = s - b) <= 1) {
            if ((p = pool) != null)
                // signalWork方法的意义在于，在当前活动的工作线程过少的情况下，创建新的工作线程
                p.signalWork(p.workQueues, this);
        }
        else if (n >= m)    // 如果array的剩余空间不够了，则进行增加
            growArray();
    }
}
```
在分析 [ForkJoinTask](https://houlong123.github.io/2017/12/21/Fork-Join框架源码学习之ForkJoinTask学习/) 的`fork`源码时，我们已经知道：如果添加任务的线程为工作线程，则会调用该方法将任务入队。该方法的大致逻辑为：
1. 判断工作队列内部任务列表是否为空，如果非空，往下执行；否则方法结束。
2. 定位待插入元素的索引：`(m & s) << ASHIFT) + ABASE`
3. 将元素入队
4. 当前活动的工作线程过少的情况下，则调用`signalWork()`方法创建或激活工作线程。
5. 如果需要扩容，则进行扩容操作

###### pop源码

```java
/**
 * Takes next task, if one exists, in LIFO order.  Call only
 * by owner in unshared queues.
 */
//通过LIFO策略（后进先出队列），获取下一个任务
final ForkJoinTask<?> pop() {
    ForkJoinTask<?>[] a; ForkJoinTask<?> t; int m;
    if ((a = array) != null && (m = a.length - 1) >= 0) {   //在待工作任务不为空，且数量大于1的时候，从待工作列表中获取下一个任务
        for (int s; (s = top - 1) - base >= 0;) { //将top值减一，赋值给变量s，如果s>=base,则遍历数组，获取下个任务的索引
            long j = ((m & s) << ASHIFT) + ABASE;   //定位下个任务的索引
            if ((t = (ForkJoinTask<?>)U.getObject(a, j)) == null)   //从工作列表中获取指定索引处的值，如果为空，break。跳出循环
                break;
            if (U.compareAndSwapObject(a, j, t, null)) {    //将原来索引处的值置为null，并重新设置top值
                U.putOrderedInt(this, QTOP, s);
                return t;
            }
        }
    }
    return null;
}
```
该方法的功能：以后进先出（LIFO）策略，获取下一个待执行任务。该策略是在构造`ForkJoinPool`对象时根据`asyncMode`属性值决定：`策略 = asyncMode ? FIFO_QUEUE : LIFO_QUEUE`。

该方法逻辑很简单，就是从队列的头部（top位置）获取下个工作任务。


###### poll源码

```java
/**
 * Takes next task, if one exists, in FIFO order.
 */
// 通过FIFO策略（先进先去），获取下一个任务
final ForkJoinTask<?> poll() {
    ForkJoinTask<?>[] a; int b; ForkJoinTask<?> t;
    while ((b = base) - top < 0 && (a = array) != null) {   //在有待处理任务且列表不为空的情况下，循环遍历任务列表
        int j = (((a.length - 1) & b) << ASHIFT) + ABASE;    //定位下个任务的索引
        t = (ForkJoinTask<?>)U.getObjectVolatile(a, j);     //获取索引处的值
        if (base == b) {    //在没有其他线程修改列表的话，往下执行
            //任务不为空的情况下，设置列表，并修改base值，然后返回任务
            if (t != null) {
                if (U.compareAndSwapObject(a, j, t, null)) {
                    base = b + 1;
                    return t;
                }
            }
            else if (b + 1 == top) // now empty 如果b + 1 == top，则说明队列为空，没有可执行的任务，跳出循环
                break;
        }
    }
    return null;
}
```
该方法与`pop()`方法的逻辑一致，只是该方法是从队列的尾部(base位置)获取下个待执行任务。


###### nextLocalTask源码

```java
/**
 * Takes next task, if one exists, in order specified by mode.
 */
//获取下一个待执行任务。取决于构建ForkJoinPool对象时设置的asyncMode。在asyncMode=true时，表明队列为FIFO队列，使用poll()获取下一任务；否则使用pop()获取下一任务。
final ForkJoinTask<?> nextLocalTask() {
    /**
     * LIFO_QUEUE   = 0;
     * FIFO_QUEUE   = 1 << 16;
     * SMASK        = 0xffff;
     * mode = asyncMode ? FIFO_QUEUE : LIFO_QUEUE
     * config = (parallelism & SMASK) | mode
     */
    return (config & FIFO_QUEUE) == 0 ? pop() : poll();
}
```
该方法会在任务添加进工作队列后，调用`signalWork()`方法添加新工作线程时被调用。具体流程在另篇文章分析。

该方法的功能就是根据不同顺序获取下个待执行任务。该顺序取决于构建ForkJoinPool对象时设置的asyncMode。在asyncMode=true时，表明队列为FIFO队列，使用poll()获取下一任务；否则使用pop()获取下一任务。

###### tryUnpush源码

```java
/**
 * Pops the given task only if it is at the current top.
 * (A shared version is available only via FJP.tryExternalUnpush)
 */
//当指定的任务为当前队列顶部时，弹出
final boolean tryUnpush(ForkJoinTask<?> t) {
    ForkJoinTask<?>[] a; int s;
    if ((a = array) != null && (s = top) != base &&     //待处理任务列表不为空且有元素
            U.compareAndSwapObject
                    (a, (((a.length - 1) & --s) << ASHIFT) + ABASE, t, null)) {     //将任务出队
        U.putOrderedInt(this, QTOP, s);
        return true;
    }
    return false;
}
```
该方法主要的功能是：将任务t从工作队列中取出。从代码注释可知，该方法有一个相同逻辑的名为`tryExternalUnpush()`方法，该方法为`ForkJoinPool`中的方法。

###### runTask源码

```java
/**
 * Executes the given task and any remaining local tasks.
 */
final void runTask(ForkJoinTask<?> task) {
    if (task != null) {
        scanState &= ~SCANNING; //标识该线程处于繁忙状态 mark as busy
        (currentSteal = task).doExec(); //执行偷取的Task
        U.putOrderedObject(this, QCURRENTSTEAL, null); // release for GC

        //调用execLocalTasks对线程所属的WorkQueue内的任务进行LIFO执行
        execLocalTasks();
        ForkJoinWorkerThread thread = owner;
        if (++nsteals < 0)      // collect on overflow
            transferStealCount(pool);
        scanState |= SCANNING;
        if (thread != null)
            thread.afterTopLevelExec();
    }
}
```
该方法的功能是：执行给定的任务或者窃取其他工作队列的任务并执行。

###### execLocalTasks源码

```java
/**
 * Removes and executes all local tasks. If LIFO, invokes
 * pollAndExecAll. Otherwise implements a specialized pop loop
 * to exec until empty.
 */
//删除并执行所有本地任务。
final void execLocalTasks() {
    int b = base, m, s;
    //获取当前线程内部任务
    ForkJoinTask<?>[] a = array;

    if (b - (s = top - 1) <= 0 && a != null &&
            (m = a.length - 1) >= 0) {  //有任务

        if ((config & FIFO_QUEUE) == 0) {   //如果是先进先出（FIFO）队列，则从top位置依次取出任务并执行
            for (ForkJoinTask<?> t;;) {

                if ((t = (ForkJoinTask<?>)U.getAndSetObject
                        (a, ((m & s) << ASHIFT) + ABASE, null)) == null)    //任务为空，则说明还没有任务，则跳出循环，方法结束
                    break;

                //修改top值
                U.putOrderedInt(this, QTOP, s);
                //执行任务
                t.doExec();
                //如果任务全部执行完，则跳出循环，方法结束
                if (base - (s = top - 1) > 0)
                    break;
            }
        }
        else    //如果是后进先出（LIFO）队列，则调用pollAndExecAll方法删除并执行任务
            pollAndExecAll();
    }
}
```
该方法的功能是：删除并执行所有本地任务，即工作队列处理自己内部的所有任务。代码逻辑：
1. 获取当前工作队列的内部任务
2. 如果内部任务不为空，则往下执行；否则，方法结束
3. 如果队列为FIFO，则从top位置开始删除并执行任务
4. 如果队列为LIFO，则调用`pollAndExecAll()`方法执行任务。

###### pollAndExecAll源码

```java
/**
 * Polls and runs tasks until empty.
 */
final void pollAndExecAll() {
    for (ForkJoinTask<?> t; (t = poll()) != null;)
        t.doExec();
}
```
代码很简单，循环调用`poll()`方法从工作队列中获取任务，然后执行。



###### tryRemoveAndExec源码

```java
/**
 * If present, removes from queue and executes the given task,
 * or any other cancelled task. Used only by awaitJoin.
 *
 * @return true if queue empty and task not known to be done
 */
final boolean tryRemoveAndExec(ForkJoinTask<?> task) {
    ForkJoinTask<?>[] a; int m, s, b, n;

    //判断待处理任务列表是否为空
    if ((a = array) != null && (m = a.length - 1) >= 0 &&
            task != null) {

        //遍历 找到task,尝试进行执行和删除操作
        while ((n = (s = top) - (b = base)) > 0) {
            for (ForkJoinTask<?> t;;) {      // traverse from s to b
                //从索引s处开始遍历
                long j = ((--s & m) << ASHIFT) + ABASE;

                if ((t = (ForkJoinTask<?>)U.getObject(a, j)) == null)
                    return s + 1 == top;     // shorter than expected
                else if (t == task) {   //找到任务
                    boolean removed = false;

                    //移除任务
                    if (s + 1 == top) {      // pop
                        if (U.compareAndSwapObject(a, j, task, null)) {
                            U.putOrderedInt(this, QTOP, s);
                            removed = true;
                        }
                    }
                    else if (base == b)      // replace with proxy
                        removed = U.compareAndSwapObject(
                                a, j, task, new EmptyTask());

                    //如果任务被移除，执行task任务
                    if (removed)
                        task.doExec();
                    break;
                }
                else if (t.status < 0 && s + 1 == top) {    //遍历到的任务t被取消，则移除任务t，并重新设置top值，然后重新遍历
                    if (U.compareAndSwapObject(a, j, t, null))
                        U.putOrderedInt(this, QTOP, s);
                    break;                  // was cancelled
                }
                if (--n == 0)   //没有带处理任务
                    return false;
            }
            if (task.status < 0)
                return false;
        }
    }
    return true;
}
```
该方法的功能是：执行并移除指定任务。代码逻辑：
1. 判断功能队列中的任务是否为空，为空，直接返回；否则，往下执行。
2. 遍历任务列表，找到task,尝试进行执行和删除操作
    1. 从top位置遍历任务列表，获取索引j.
    2. 如果索引j处的任务为空，则自增top值，继续遍历。
    3. 如果索引j处的任务为指定任务，则从任务列表中移除任务，并执行。
    4. 如果索引j处的任务t被取消，则移除任务t，并重新设置top值，然后重新遍历。
    
`备注：WorkerQueue中的大部分方法都是在启动工作线程后，会被调用到。`


##### ForkJoinPool主要方法
###### 构造函数

```java
//无参构造函数。默认线程数为处理器的核数
public ForkJoinPool() {
    this(Math.min(MAX_CAP, Runtime.getRuntime().availableProcessors()),
            defaultForkJoinWorkerThreadFactory, null, false);
}

//指定线程数
public ForkJoinPool(int parallelism) {
    this(parallelism, defaultForkJoinWorkerThreadFactory, null, false);
}


public ForkJoinPool(int parallelism,
                    ForkJoinWorkerThreadFactory factory,
                    UncaughtExceptionHandler handler,
                    boolean asyncMode) {
    this(checkParallelism(parallelism),
            checkFactory(factory),
            handler,
            asyncMode ? FIFO_QUEUE : LIFO_QUEUE,
            "ForkJoinPool-" + nextPoolId() + "-worker-");
    checkPermission();
}

private ForkJoinPool(int parallelism,
                     ForkJoinWorkerThreadFactory factory,
                     UncaughtExceptionHandler handler,
                     int mode,
                     String workerNamePrefix) {
    this.workerNamePrefix = workerNamePrefix;
    this.factory = factory;
    this.ueh = handler;
    this.config = (parallelism & SMASK) | mode;
    long np = (long)(-parallelism); // offset ctl counts
    //ctl初始化为一个long型负数
    this.ctl = ((np << AC_SHIFT) & AC_MASK) | ((np << TC_SHIFT) & TC_MASK);
}
```
从代码中可知，ForkJoinPool 有四个构造函数，但是有一个是用`private`修饰符进行修饰的，说明它不能直接调到，其他三个间接或直接调用这个私有构造函数。在默认情况下，池中的线程数为处理器的核数，并使用默认线程创建工厂。


###### submit源码

```java
/**
 * Submits a ForkJoinTask for execution.
 *
 * @param task the task to submit
 * @param <T> the type of the task's result
 * @return the task
 * @throws NullPointerException if the task is null
 * @throws RejectedExecutionException if the task cannot be
 *         scheduled for execution
 */
 //提交一个ForkJoinTask执行
public <T> ForkJoinTask<T> submit(ForkJoinTask<T> task) {
    if (task == null)
        throw new NullPointerException();
    externalPush(task);
    return task;
}
```
该方法的功能是：`提交一个ForkJoinTask执行`。在提交任务之前，会进行非空判断。如果提交的任务为null，则抛异常；否则调用内部方法`externalPush()`进行处理。

&nbsp;&nbsp;&nbsp;&nbsp;在 ForkJoinPool 类中，有多个重载的`submit()`方法。重载的方法都会接受一个`Runnable` 或者 `Callable<T>` 类型的参数。在方法内部，都会将对应的参数封装成`ForkJoinTask` 类型的对象，然后同样调用内部方法`externalPush()`进行处理。

###### 重载submit源码

```java
public <T> ForkJoinTask<T> submit(Callable<T> task) {
    ForkJoinTask<T> job = new ForkJoinTask.AdaptedCallable<T>(task);
    externalPush(job);
    return job;
}
```

###### externalPush源码

```java
/**
 * Tries to add the given task to a submission queue at
 * submitter's current queue. Only the (vastly) most common path
 * is directly handled in this method, while screening for need
 * for externalSubmit.
 *
 * @param task the task. Caller must ensure non-null.
 */
 //该方法是在当前线程为非工作线程插入任务时调用。如果当前线程为任务线程，
// 则直接将任务插入当前线程中的工作队列中；如果当前线程不是任务线程，则需要从池中先随机定位到一个工作线程，然后将任务插入。
    
final void externalPush(ForkJoinTask<?> task) {
    WorkQueue[] ws; WorkQueue q; int m;
    //取得一个随机探查数，可能为0也可能为其它数
    int r = ThreadLocalRandom.getProbe();

    // 获取当前ForkJoinPool的运行状态
    int rs = runState;

    //工作队列列表不为空 && 指定位置的工作队列不空 && 成功将队列上锁
    /**
     * SQMASK  = 0x007e = 0000 0000 0111 1110 = 126
     * 任何数和126进行“与”运算，其结果只可能是0或者偶数，所以 (m & r & SQMASK) 取出的是第偶数个队列，
     *  即索引位为偶数的工作队列用于存储从外部提交到ForkJoinPool中的任务
     */
    if ((ws = workQueues) != null && (m = (ws.length - 1)) >= 0 &&
            (q = ws[m & r & SQMASK]) != null && r != 0 && rs > 0 &&
            U.compareAndSwapInt(q, QLOCK, 0, 1)) {

        ForkJoinTask<?>[] a; int am, n, s;
        if ((a = q.array) != null &&
                (am = a.length - 1) > (n = (s = q.top) - q.base)) { //工作队列中有待处理子任务

            int j = ((am & s) << ASHIFT) + ABASE;   //待处理子任务的插入索引
            //将task放入队列
            U.putOrderedObject(a, j, task);
            //将队列的top值加1
            U.putOrderedInt(q, QTOP, s + 1);
            //释放锁
            U.putIntVolatile(q, QLOCK, 0);

            // 如果条件成立，说明这时处于active的工作线程可能还不够，所以调用signalWork方法
            if (n <= 1)
                signalWork(ws, q);
            return;
        }

        //在工作队列中没有待处理子任务时，释放锁
        U.compareAndSwapInt(q, QLOCK, 1, 0);
    }

    //进行WorkQueue数组初始化
    externalSubmit(task);
}
```
实现逻辑：
1. 获取一个随机探查数。该数用于定位（`m & r & SQMASK`）待提交任务要提交到哪个工作队列。
2. 获取当前ForkJoinPool的运行状态
3. 在池中的工作队列列表不为空 && 指定位置的工作队列q不空，并成功将队列上锁，往下执行；否则，执行第4步。
    1. 队列q内部的任务不为空且工作队列中有待处理任务，则执行下一步；否则释放锁后，执行第4步。
    2. 计算待处理任务的插入索引
    3. 通过CAS将任务入队，并修改队列的top值及释放锁
    4. 如果处于active状态的工作线程不够，则调用`signalWork`激活线程或新建线程
    5. 方法结束返回

4. 执行 `externalSubmit()` 方法，进行WorkQueue数组初始化及入队操作。


###### externalSubmit源码

```java
/**
 * Full version of externalPush, handling uncommon cases, as well
 * as performing secondary initialization upon the first
 * submission of the first task to the pool.  It also detects
 * first submission by an external thread and creates a new shared
 * queue if the one at index if empty or contended.
 *
 * @param task the task. Caller must ensure non-null.
 */
private void externalSubmit(ForkJoinTask<?> task) {
    int r;                                    // initialize caller's probe
    // 初始化探测器
    if ((r = ThreadLocalRandom.getProbe()) == 0) {
        ThreadLocalRandom.localInit();
        r = ThreadLocalRandom.getProbe();
    }

    for (;;) {
        WorkQueue[] ws; WorkQueue q; int rs, m, k;
        boolean move = false;

        /**
         * 如果当前ForkJoinPool的运行状态为SHUTDOWN，表明已被终止，不接受任务了。则尝试中断，并抛异常
         *
         * RSLOCK     = 1
         * RSIGNAL    = 1 << 1
         * STARTED    = 1 << 2
         * STOP       = 1 << 29
         * TERMINATED = 1 << 30
         * SHUTDOWN   = 1 << 31 (负数)
         */
        if ((rs = runState) < 0) {
            tryTerminate(false, false);     // help terminate
            throw new RejectedExecutionException();
        }

        else if ((rs & STARTED) == 0 ||     // initialize
                ((ws = workQueues) == null || (m = ws.length - 1) < 0)) {   // 当前ForkJoinPool类中，还没有任何队列，所以要进行队列初始化
            int ns = 0;
            //获取runState锁
            rs = lockRunState();
            try {
                //判断是否已经初始化，正常（不考虑并发）情况是0。
                if ((rs & STARTED) == 0) {
                    // 通过原子操作，完成“任务窃取次数”这个计数器的初始化
                    U.compareAndSwapObject(this, STEALCOUNTER, null, new AtomicLong());
                    // create workQueues array with size a power of two
                    //创建数组大小为2的幂次方的工作队列

                    int p = config & SMASK; // ensure at least 2 slots

                    // n这个变量就是要计算的WorkQueue数组的大小
                    int n = (p > 1) ? p - 1 : 1;

                    /**
                     *  备注：SMASK = 0xffff。config = (parallelism & SMASK) | mode;
                     *  模拟 mode = 0，则 config = parallelism
                     *  由计算机计算结果为：输出格式：parallelism>>>>>数组大小
                         1>>>>>4
                         2>>>>>4
                         3>>>>>8
                         4>>>>>8
                         5>>>>>16
                         6>>>>>16
                         7>>>>>16
                         8>>>>>16
                         9>>>>>32
                         10>>>>>32
                     */
                    n |= n >>> 1; n |= n >>> 2;  n |= n >>> 4;
                    n |= n >>> 8; n |= n >>> 16; n = (n + 1) << 1;
                    workQueues = new WorkQueue[n];
                    ns = STARTED;
                }
            } finally {
                unlockRunState(rs, (rs & ~RSLOCK) | ns);
            }
        }
        else if ((q = ws[k = r & m & SQMASK]) != null) {    //和externalPush方法的实现逻辑大体一致
            if (q.qlock == 0 && U.compareAndSwapInt(q, QLOCK, 0, 1)) {
                ForkJoinTask<?>[] a = q.array;
                int s = q.top;
                boolean submitted = false; // initial submission or resizing
                try {                      // locked version of push
                    if ((a != null && a.length > s + 1 - q.base) ||
                            (a = q.growArray()) != null) {
                        int j = (((a.length - 1) & s) << ASHIFT) + ABASE;
                        U.putOrderedObject(a, j, task);
                        U.putOrderedInt(q, QTOP, s + 1);
                        submitted = true;
                    }
                } finally {
                    U.compareAndSwapInt(q, QLOCK, 1, 0);
                }

                // 如果task成功提交，会尝试启动worker
                if (submitted) {
                    signalWork(ws, q);
                    return;
                }
            }
            move = true;                   // move on failure
        }
        else if (((rs = runState) & RSLOCK) == 0) { // create new queue
            q = new WorkQueue(this, null);
            q.hint = r;
            q.config = k | SHARED_QUEUE;
            q.scanState = INACTIVE;
            rs = lockRunState();           // publish index
            if (rs > 0 &&  (ws = workQueues) != null &&
                    k < ws.length && ws[k] == null)
                ws[k] = q;                 // else terminated
            unlockRunState(rs, rs & ~RSLOCK);
        }
        else
            move = true;                   // move if busy
        if (move)
            r = ThreadLocalRandom.advanceProbe(r);
    }
}
```
功能分析：在调用`externalPush()`方法将任务入队时，如果待插入任务定位到的工作队列没空或者锁定失败，则会调用`externalSubmit()`方法。从方法注释可知，`externalSubmit()`方法是完整版的`externalPush()`方法。该方法处理不常见的情况，以及在首次将第一个任务提交到池时执行辅助初始化。如果索引处的为空或争用，它还检测外部线程的第一次提交，并创建一个新的共享队列。

实现逻辑：
1. 初始化探测器。该探测器产生的随机数用于定位工作队列索引。
2. 进入无限循环，失败即重试，直至任务成功入队
    1. 如果当前ForkJoinPool的运行状态为SHUTDOWN，表明已被终止，不接受任务了。则尝试中断，并抛异常
    2. 如果当前ForkJoinPool对象需要初始化，则进行初始化操作。循环继续
    3. 如果定位的索引处工作队列不为空，则加锁将任务入队。在入队的过程中，如需扩容，则进行扩容操作。
    4. 如果定位的索引处工作队列为空，则初始化该线程对应的工作队列。循环继续
    
在将任务入队的过程中，如果需要扩容，则会调用WorkerQueue类中的`growArray()`方法进行扩容操作。

###### growArray源码

```java
/**
 * Initializes or doubles the capacity of array. Call either
 * by owner or with lock held -- it is OK for base, but not
 * top, to move while resizings are in progress.
 */
 //该方法会初始化或者进行队列2倍扩容
final ForkJoinTask<?>[] growArray() {
    ForkJoinTask<?>[] oldA = array;

    int size = oldA != null ? oldA.length << 1 : INITIAL_QUEUE_CAPACITY;    //如果首次初始化，则数组默认大小为2^13。否则，在原基础上扩容2倍

    if (size > MAXIMUM_QUEUE_CAPACITY)  //如果数组越界，则抛出异常
        throw new RejectedExecutionException("Queue capacity exceeded");

    int oldMask, t, b;
    ForkJoinTask<?>[] a = array = new ForkJoinTask<?>[size];
    if (oldA != null && (oldMask = oldA.length - 1) >= 0 &&
            (t = top) - (b = base) > 0) {   //原数组有元素，则进行数组拷贝
        int mask = size - 1;    //计算元素索引的掩码

        //从base（下一个出队元素的索引位）位置开始拷贝原数组元素到新数组
        do { // emulate poll from old array, push to new array
            ForkJoinTask<?> x;
            int oldj = ((b & oldMask) << ASHIFT) + ABASE;   //旧元素的索引
            int j    = ((b &    mask) << ASHIFT) + ABASE;   //新元素的索引
            x = (ForkJoinTask<?>)U.getObjectVolatile(oldA, oldj);   //获取要移动的元素
            if (x != null &&
                    U.compareAndSwapObject(oldA, oldj, x, null))    //如果元素不为空，且成功将旧数组相应位置置空
                U.putObjectVolatile(a, j, x);   //拷贝到新数组
        } while (++b != t);
    }
    return a;
}
```
该方法实现逻辑相对比较简单。就是进行工作队列的初始化或者2倍扩容操作。在确定好工作队列中的新工作任务的数组大小后` int size = oldA != null ? oldA.length << 1 : INITIAL_QUEUE_CAPACITY;   `，在没有超过最大数组容量的前提下，会依次从旧数组中拷贝元素到新数组中。


在将任务成功入队后，还会通过调用`signalWork()`方法来启动worker

###### signalWork源码

```java
/**
 * Tries to create or activate a worker if too few are active.
 *
 * @param ws the worker array to use to find signallees
 * @param q a WorkQueue --if non-null, don't retry if now empty
 */
 //如果工作者太少活跃的话，则尝试新建或激活工作者
final void signalWork(WorkQueue[] ws, WorkQueue q) {
    long c; int sp, i; WorkQueue v; Thread p;
    //这里初始化后的ctl是个long型负数，但是转换成int后，变成0
    while ((c = ctl) < 0L) {                       // too few active
        if ((sp = (int)c) == 0) {                  // no idle workers
            if ((c & ADD_WORKER) != 0L)            // too few workers
                //初始状态，会调用这个方法，调用完退出
                tryAddWorker(c);
            break;
        }
        if (ws == null)                            // unstarted/terminated
            break;
        if (ws.length <= (i = sp & SMASK))         // terminated
            break;
        if ((v = ws[i]) == null)                   // terminating
            break;
        int vs = (sp + SS_SEQ) & ~INACTIVE;        // next scanState
        int d = sp - v.scanState;                  // screen CAS
        long nc = (UC_MASK & (c + AC_UNIT)) | (SP_MASK & v.stackPred);
        if (d == 0 && U.compareAndSwapLong(this, CTL, c, nc)) {
            v.scanState = vs;                      // activate v
            if ((p = v.parker) != null)
                U.unpark(p);
            break;
        }
        if (q != null && q.base == q.top)          // no more work
            break;
    }
}
```
该方法主要是在有任务添加到工作队列后调用。该方法主要功能是：在工作者活跃的太少时，会新增或者激活工作者。

###### tryAddWorker源码

```java
private void tryAddWorker(long c) {
    boolean add = false;
    do {
        long nc = ((AC_MASK & (c + AC_UNIT)) |
                (TC_MASK & (c + TC_UNIT)));
        if (ctl == c) {
            int rs, stop;                 // check if terminating
            if ((stop = (rs = lockRunState()) & STOP) == 0)
                add = U.compareAndSwapLong(this, CTL, c, nc);
            unlockRunState(rs, rs & ~RSLOCK);
            if (stop != 0)
                break;
            if (add) {
                createWorker();
                break;
            }
        }
    } while (((c = ctl) & ADD_WORKER) != 0L && (int)c == 0);
}
```
该方法会经过一系列判断，最终调用`createWorker()`方法进行工作者的新增。

###### createWorker源码

```java
/**
 * Tries to construct and start one worker. Assumes that total
 * count has already been incremented as a reservation.  Invokes
 * deregisterWorker on any failure.
 *
 * @return true if successful
 */
private boolean createWorker() {
    ForkJoinWorkerThreadFactory fac = factory;
    Throwable ex = null;
    ForkJoinWorkerThread wt = null;
    try {
        if (fac != null && (wt = fac.newThread(this)) != null) {
            wt.start();
            return true;
        }
    } catch (Throwable rex) {
        ex = rex;
    }
    deregisterWorker(wt, ex);
    return false;
}
```
该方法会通过`ForkJoinWorkerThreadFactory`对象构建`ForkJoinWorkerThread`类型线程，然后启动线程，进行任务处理。在上面我们已经对 ForkJoinWorkerThread 类进行了简单分析，其`run()` 方法内部调用了 ForkJoinPool 类的 `runWorker()` 方法。

###### runWorker源码

```java
/**
 * Top-level runloop for workers, called by ForkJoinWorkerThread.run.
 */
final void runWorker(WorkQueue w) {
    //分配工作队列，初始化或者扩容
    w.growArray();                   // allocate queue
    int seed = w.hint;               // initially holds randomization hint
    //准备偷取的队列索引
    int r = (seed == 0) ? 1 : seed;  // avoid 0 for xorShift
    for (ForkJoinTask<?> t;;) {
        if ((t = scan(w, r)) != null)   //调用scan尝试去偷取一个任务,然后调用runTask或者awaitWork
            w.runTask(t);
        else if (!awaitWork(w, r))  //如果任务中断，返回false
            break;
        r ^= r << 13; r ^= r >>> 17; r ^= r << 5; // xorshift
    }
}
```
实现逻辑：
1. 进行工作队列的扩容操作
2. 获取随机数，用户确定窃取任务的队列
3. 进行无限循环。
    1. 调用`scan()`方法尝试窃取一个任务t
    2. 如果窃取的任务t不为空，则调用`runTask()`方法执行任务
    3. 如果窃取的任务t为空，则调用`awaitWork()`方法阻塞工作线程等待窃取任务。如果任务中断，break跳出循环结束。
    4. 重新定位，再次循环

###### scan源码

```java
/**
 * Scans for and tries to steal a top-level task. Scans start at a
 * random location, randomly moving on apparent contention,
 * otherwise continuing linearly until reaching two consecutive
 * empty passes over all queues with the same checksum (summing
 * each base index of each queue, that moves on each steal), at
 * which point the worker tries to inactivate and then re-scans,
 * attempting to re-activate (itself or some other worker) if
 * finding a task; otherwise returning null to await work.  Scans
 * otherwise touch as little memory as possible, to reduce
 * disruption on other scanning threads.
 *
 * @param w the worker (via its WorkQueue)
 * @param r a random seed
 * @return a task, or null if none found
 */
//扫描并尝试窃取顶级任务。
private ForkJoinTask<?> scan(WorkQueue w, int r) {
    WorkQueue[] ws; int m;

    if ((ws = workQueues) != null && (m = ws.length - 1) > 0 && w != null) { //任务队列列表不为空
        int ss = w.scanState;                     // initially non-negative
        for (int origin = r & m, k = origin, oldSum = 0, checkSum = 0;;) {
            WorkQueue q; ForkJoinTask<?>[] a; ForkJoinTask<?> t;
            int b, n; long c;
            if ((q = ws[k]) != null) {
                if ((n = (b = q.base) - q.top) < 0 &&
                        (a = q.array) != null) {      // non-empty 有任务

                    //获取索引。在扫描任务时，从工作队列的尾部进行扫描
                    long i = (((a.length - 1) & b) << ASHIFT) + ABASE;

                    if ((t = ((ForkJoinTask<?>)
                            U.getObjectVolatile(a, i))) != null &&
                            q.base == b) {  //指定索引处的任务不空且没有其他线程并行操作
                        if (ss >= 0) {
                            if (U.compareAndSwapObject(a, i, t, null)) {
                                q.base = b + 1;
                                if (n < -1)       // signal others
                                    signalWork(ws, q);
                                return t;
                            }
                        }
                        else if (oldSum == 0 &&   // try to activate
                                w.scanState < 0)
                            tryRelease(c = ctl, ws[m & (int)c], AC_UNIT);
                    }
                    if (ss < 0)                   // refresh
                        ss = w.scanState;
                    r ^= r << 1; r ^= r >>> 3; r ^= r << 10;
                    origin = k = r & m;           // move and rescan
                    oldSum = checkSum = 0;
                    continue;
                }
                checkSum += b;
            }

            //如果我们遍历了一圈(((k = (k + 1) & m) == origin))都没有偷到,我们就认为当前的active 线程过剩了,我们准备将当前的线程(即owner)挂起
            if ((k = (k + 1) & m) == origin) {    // continue until stable
                if ((ss >= 0 || (ss == (ss = w.scanState))) &&
                        oldSum == (oldSum = checkSum)) {
                    if (ss < 0 || w.qlock < 0)    // already inactive
                        break;
                    int ns = ss | INACTIVE;       // try to inactivate
                    long nc = ((SP_MASK & ns) |
                            (UC_MASK & ((c = ctl) - AC_UNIT)));
                    w.stackPred = (int)c;         // hold prev stack top
                    U.putInt(w, QSCANSTATE, ns);
                    if (U.compareAndSwapLong(this, CTL, c, nc))
                        ss = ns;
                    else
                        w.scanState = ss;         // back out
                }
                checkSum = 0;
            }
        }
    }
    return null;
}
```
该方法主要是扫描尝试从其他工作队列窃取任务。大致逻辑为：
1. 判断池中的工作队列是否为空，在不为空的情况下，执行下一步；否则，直接返回null。
2. 进入for循环，根据随机数确定待窃取任务的工作队列（即从这个队列中窃取任务）索引。
3. 待窃取任务的工作队列不为空，且有任务的情况下，从队列的尾部（base位置）获取待窃取任务索引。
4. 满足一定条件下，获取指定索引处的任务，并返回。
5. 如果没有窃取到任务，说明工作线程过剩，此时将当前线程挂起，然后跳出循环，方法结束。

我们在分析 [ForkJoinTask](https://houlong123.github.io/2017/12/21/Fork-Join框架源码学习之ForkJoinTask学习/) 时有说过：`在线程处理自己内部的工作队列时，是从头部开始，即pop()；但是在窃取工作帮助其他线程处理任务时，是从尾部开始，即poll()。`


###### registerWorker源码

```java
/**
 * Callback from ForkJoinWorkerThread constructor to establish and
 * record its WorkQueue.
 *
 * @param wt the worker thread
 * @return the worker's queue
 */
final WorkQueue registerWorker(ForkJoinWorkerThread wt) {
    UncaughtExceptionHandler handler;
    //将工作线程设置为后台线程
    wt.setDaemon(true);                           // configure thread

    //设置异常处理类
    if ((handler = ueh) != null)
        wt.setUncaughtExceptionHandler(handler);

    //创建工作队列
    WorkQueue w = new WorkQueue(this, wt);
    int i = 0;                                    // assign a pool index
    //确定工作队列的类型
    int mode = config & MODE_MASK;
    int rs = lockRunState();
    try {
        WorkQueue[] ws; int n;                    // skip if no array

        //池中有工作队列列表时，将新创建的工作队列添加进去。在添加的过程中，如果池中有工作队列列表容量不够，则需要进行2倍扩容操作
        if ((ws = workQueues) != null && (n = ws.length) > 0) {
            int s = indexSeed += SEED_INCREMENT;  // unlikely to collide
            int m = n - 1;
            i = ((s << 1) | 1) & m;               // odd-numbered indices
            if (ws[i] != null) {                  // collision
                int probes = 0;                   // step by approx half n
                int step = (n <= 4) ? 2 : ((n >>> 1) & EVENMASK) + 2;
                while (ws[i = (i + step) & m] != null) {
                    if (++probes >= n) {
                        workQueues = ws = Arrays.copyOf(ws, n <<= 1);
                        m = n - 1;
                        probes = 0;
                    }
                }
            }
            w.hint = s;                           // use as random seed
            w.config = i | mode;
            w.scanState = i;                      // publication fence
            ws[i] = w;
        }
    } finally {
        unlockRunState(rs, rs & ~RSLOCK);
    }
    wt.setName(workerNamePrefix.concat(Integer.toString(i >>> 1)));
    return w;
}
```
该方法是在`ForkJoinWorkerThread` 的构造函数中被调用，主要用来创建工作队列。代码的实现逻辑相对比较简单：设置线程属性，然后将新建的工作队列丢进池中的工作队列列表。


