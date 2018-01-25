---
title: Fork/Join框架源码学习之ForkJoinTask学习
date: 2017-12-21 15:12:04
tags: java并发
---


#### 前言
Fork/Join框架是一个用于并行执行任务的框架，核心是：<font color=red>把一个大任务分割成若干个小任务，最终汇总每个小任务结果后得到大任务结果。</font>体现了`分而治之`的思想。类似于算法中的`归并排序`。该框架之所以有更好的并发性能，是因为其充分利用了所有线程的工作能力，避免空闲线程，充分发挥多核并行的处理能力。


![image](http://www.oracle.com/ocom/groups/public/@otn/documents/digitalasset/422590.png)

#### 使用

##### 预备知识点
在java的官方文档介绍[Fork/Join](https://docs.oracle.com/javase/tutorial/essential/concurrency/forkjoin.html)中，可以知道Fork/Join框架的基本使用的伪代码为：

<!-- more -->

```java
if (my portion of the work is small enough)
   do the work directly
else
   split my work into two pieces
   invoke the two pieces and wait for the results
```
在使用的时候， 将这些伪代码封装到`ForkJoinTask`的子类中。通常使用的`ForkJoinTask`的子类是：[`RecursiveTask（有返回结果）`](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/RecursiveTask.html) 和 [`RecursiveAction（无返回结果）`](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/RecursiveAction.html)。我们只要根据需要，继承这两个类中的一个，并实现其中的方法，即可实现自己定义的任务类。然后在使用`ForkJoinPool`去执行任务。

##### 使用实例：排序

```java
public class SortTest {

    private static int MAX = 1000;

    private static int inits[] = new int[MAX];

    //数组初始化
    static {
        Random random = new Random();
        for (int i = 0; i < MAX; i++) {
            inits[i] = random.nextInt(1000000);
        }
    }

    public static void main(String[] args) {
    
        int a[] = inits.clone();
        
        //1. Create a task that represents all of the work to be done.
        SortTask task  = new SortTask(a);
        
        //2. Create the ForkJoinPool that will run the task
        ForkJoinPool pool = new ForkJoinPool();
        
        System.out.println("排序前：" + Arrays.toString(a));
        
        //3. Run the task
        pool.submit(task);
        
        do {
            try {
                TimeUnit.MILLISECONDS.sleep(5);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        } while(!task.isDone());
        System.out.println("排序后：" + Arrays.toString(a));

    }


    // 继承ForkJoinTask的子类RecursiveAction。由于是没有返回，所以需要在获取之前，先判断任务是否已完成，没有完成的话，继续等待。
    static class SortTask extends RecursiveAction {

        static final int THRESHOLD = 10;
        final int[] array; final int lo, hi;
        
        SortTask(int[] array, int lo, int hi) {
            this.array = array; this.lo = lo; this.hi = hi;
        }

        SortTask(int[] array) { this(array, 0, array.length); }

        @Override
        protected void compute() {
            if (hi - lo < THRESHOLD) {
                sortSequentially(lo, hi);
            } else {
                int mid = (lo + hi) >>> 1;
                invokeAll(new SortTask(array, lo, mid),
                        new SortTask(array, mid, hi));
                        
                 /**
                 * 另一种方式，与invokeAll实现逻辑一样。
                 *
                 * 备注：需要注意的是，join调用顺序要与fork的调用顺序相反。具体原因由下文源码分析可知。
                 * 或看博客https://yq.aliyun.com/articles/48736?spm=5176.100239.blogcont48737.8.txj280
                 *
                 *   SortTask right = new SortTask(array, lo, mid);
                 *   SortTask left = new SortTask(array, mid, hi);
                 *    //任务分解
                 *    right.fork();
                 *    left.fork();
                 *
                 *    //任务结果合并
                 *    left.join();
                 *    right.join();
                 */
                merge(lo, mid, hi);
            }
        }

        void sortSequentially(int lo, int hi) {
            Arrays.sort(array, lo, hi);
        }
        void merge(int lo, int mid, int hi) {
            int[] buf = Arrays.copyOfRange(array, lo, mid);
            for (int i = 0, j = lo, k = mid; i < buf.length; j++) {
                array[j] = (k == hi || buf[i] < array[k]) ? buf[i++] : array[k++];
            }

        }
    }
}
```

#### 源码解析
由上面的使用实例可知，Fork/Join框架的使用，分为三个步骤：
1. 创建任务
2. 创建执行任务的线程池ForkJoinPool对象。
3. 执行任务

首先我们分析`ForkJoinTask`源码

#### ForkJoinTask源码分析
##### 内部数据结构

```java
public abstract class ForkJoinTask<V> implements Future<V>, Serializable {
    /** The run status of this task */
    // 任务运行状态，下面是转换为十进制的值。可知，在状态为SIGNAL或者SMASK时，才会大于0
    /**
     *   DONE_MASK = -268435456
         NORMAL = -268435456
         CANCELLED = -1073741824
         EXCEPTIONAL = -2147483648
         SIGNAL = 65536
         SMASK = 65535
     */
    volatile int status; // accessed directly by pool and workers
    static final int DONE_MASK   = 0xf0000000;  // mask out non-completion bits
    static final int NORMAL      = 0xf0000000;  // must be negative
    static final int CANCELLED   = 0xc0000000;  // must be < NORMAL
    static final int EXCEPTIONAL = 0x80000000;  // must be < CANCELLED
    static final int SIGNAL      = 0x00010000;  // must be >= 1 << 16
    static final int SMASK       = 0x0000ffff;  // short bits for tags

}
```
ForkJoinTask类继承了`Future`类。该类内部主要是有个属性，记录任务的执行状态。由使用实例可知，ForkJoinTask类的主要方法是：fork，join，invokeAll等，下面进行主要方法分析

##### fork源码

```java
//将任务进行拆分
public final ForkJoinTask<V> fork() {
    Thread t;

    //如果当前的线程是Fork/join的线程,就添加到队列中
    if ((t = Thread.currentThread()) instanceof ForkJoinWorkerThread)
        ((ForkJoinWorkerThread)t).workQueue.push(this);
    else
        //当task是直接被调用,而不是使用ForkJoinWorkerThread的话,直接执行任务.
        ForkJoinPool.common.externalPush(this);
    return this;
}
```
该方法的主要功能是：`将任务进行拆分`。在内部实现中，如果当前线程为`ForkJoinWorkerThread`线程时，则直接将该任务添加进当前线程的工作队列中；否则，则通过线程池`ForkJoinPool`将任务添加到池中的随机一个工作队列中。具体逻辑，后文再分析。

##### join源码

```java
//执行子任务并合并子任务的结果集
public final V join() {
    int s;
    //执行任务
    if ((s = doJoin() & DONE_MASK) != NORMAL)
        reportException(s);
    return getRawResult();
}
```
该方法的功能是：`执行子任务并合并子任务的结果集`。内部实现中，会调用`doJoin()`方法，执行任务，如果执行成功，则调用抽象方法`getRawResult()`返回执行结果；否则，执行`reportException()`方法处理异常。

##### doJoin源码

```java
//doJoin方法是进行合并,合并之前需要进行判断,看当前的任务是否可以执行,如果可以执行则调用doExec方法,如果不能执行则加入等待的队列
//在join获取执行任务时，是从队列的头部开始，而在sacn窃取工作任务时，是从队列的尾部开始
private int doJoin() {
    int s; Thread t; ForkJoinWorkerThread wt; ForkJoinPool.WorkQueue w;

    //判断执行的状态
    return (s = status) < 0 ? s :

            //判断是否fork/join的线程
            ((t = Thread.currentThread()) instanceof ForkJoinWorkerThread) ?

                    //判断是否在队列的头部 && 在执行头部
                    (w = (wt = (ForkJoinWorkerThread)t).workQueue).tryUnpush(this) && (s = doExec()) < 0 ? s :
                            //加入到等待的队列
                            wt.pool.awaitJoin(w, this, 0L) :
                    //不是fork/join线程 阻塞
                    externalAwaitDone();
}
```
实现逻辑：
1. 判断任务的执行状态status。如果`status < 0`，则直接返回；否则执行第2步
2. 判断当前的任务是否可以执行,如果可以执行则调用doExec方法,如果不能执行则加入等待的队列。具体逻辑为：
    1. 如果当前线程为`ForkJoinWorkerThread`线程，则从当前线程的工作队列中弹出顶部任务，并执行该任务。如果执行成功，则返回；否则，如果执行失败，则将任务加入到等待的队列。（帮助其他线程处理任务，即窃取任务 或者阻塞直到任务完成或超时）
    2. 如果不是`ForkJoinWorkerThread`线程fork/join线程 阻塞
    
在ForkJoinPool类有关`WorkQueue`介绍如下：

```java
The pop operation (always performed by owner) is:
    if ((base != top) and
       (the task at top slot is not null) and (CAS slot to null))
          decrement top and return task;

And the poll operation (usually by a stealer) is
    if ((base != top) and
       (the task at base slot is not null) and 
       (base has not changed) and
       (CAS slot to null))
          increment base and return task;
```
可知，在线程处理自己内部的工作队列时，是从头部开始，即pop()；但是在窃取工作帮助其他线程处理任务时，是从尾部开始，即poll()。

所以在`doJoin()`中，如果为`ForkJoinWorkerThread`线程时，会调用`tryUnpush()`方法从头部获取待处理任务。

##### doExec源码

```java
final int doExec() {
    int s; boolean completed;

    //判断任务状态,是否执行
    if ((s = status) >= 0) {
        try {
            //在ForkJoinTask两个子类中的实现方式都是调用compute()方法。compute()方法就是Fork Join要执行的内容，是Fork Join任务的实质，需要开发者实现
            completed = exec();
        } catch (Throwable rex) {
            return setExceptionalCompletion(rex);
        }

        //完成任务,修改状态
        if (completed)
            s = setCompletion(NORMAL);
    }
    return s;
}
```
该方法的功能是：执行任务。在之前前会判断任务状态，如果任务可执行，则调用`exec()`方法，由继承关系可知，实际调用的是`ForkJoinTask`子类的相应实现。在`RecursiveAction` 和 `RecursiveTask`中，都是调用了`compute()`方法。即调用了开发者自己的实现逻辑。在执行成功后，会通过setCompletion()方法设置任务的状态。


##### externalAwaitDone源码

```java
//阻止非工作线程直到完成。
private int externalAwaitDone() {

    //执行任务。如果是CountedCompleter子类，则调用externalHelpComplete()方法；否则执行tryExternalUnpush()方法，弹出任务，并执行
    int s = ((this instanceof CountedCompleter) ? // try helping
            ForkJoinPool.common.externalHelpComplete((CountedCompleter<?>)this, 0) :
            ForkJoinPool.common.tryExternalUnpush(this) ? doExec() : 0);

    //任务状态大于0，说明任务还未成功处理，则中断
    if (s >= 0 && (s = status) >= 0) {
        boolean interrupted = false;
        do {
            if (U.compareAndSwapInt(this, STATUS, s, s | SIGNAL)) {
                synchronized (this) {
                    if (status >= 0) {  //任务状态为SIGNAL，则中断此任务
                        try {
                            wait(0L);
                        } catch (InterruptedException ie) {
                            interrupted = true;
                        }
                    }
                    else    //唤醒所有线程
                        notifyAll();
                }
            }
        } while ((s = status) >= 0);
        if (interrupted)
            Thread.currentThread().interrupt();
    }
    return s;
}
```

##### invoke源码

```java
//执行任务并返回执行结果。如有必要，需等待任务完成
public final V invoke() {
    int s;
    if ((s = doInvoke() & DONE_MASK) != NORMAL)
        reportException(s);
    return getRawResult();
}
```
该方法主要是：`执行任务并返回执行结果`。其内部实现与`join()`方法类似，但是在实现任务执行的逻辑时其内部调用的是`doInvoke()`方法。`doInvoke()`方法与`doJoin()`方法内部实现逻辑也很类似。

##### doInvoke源码

```java
//执行任务。执行成功直接返回；执行失败后，如果属于Fork/join的线程,将任务添加到等待队列；否则阻塞
private int doInvoke() {
    int s; Thread t; ForkJoinWorkerThread wt;
    return (s = doExec()) < 0 ? s :
            ((t = Thread.currentThread()) instanceof ForkJoinWorkerThread) ?
                    //加入到等待的队列
                    (wt = (ForkJoinWorkerThread)t).pool.awaitJoin(wt.workQueue, this, 0L) :
                    //不是fork/join线程 阻塞
                    externalAwaitDone();
}
```
该方法功能和`doJoin()`方法也很类似。区别是：doInvoke()方法直接调用`doExec()`方法执行任务。执行成功，直接返回；否则，判断是否为`ForkJoinWorkerThread`线程。是，调用`awaitJoin()`方法将任务添加到等待的队列；否则调用`externalAwaitDone()`阻塞任务。


##### invokeAll源码

```java
public static void invokeAll(ForkJoinTask<?> t1, ForkJoinTask<?> t2) {
    int s1, s2;
    t2.fork();  //调用子任务2的fork()方法，将任务添加到工作队列

    //先执行子任务1
    if ((s1 = t1.doInvoke() & DONE_MASK) != NORMAL)
        t1.reportException(s1);

    //调用子任务2的doJoin()，执行任务
    if ((s2 = t2.doJoin() & DONE_MASK) != NORMAL)
        t2.reportException(s2);
}
```
invokeAll()方法将处理两个子任务。由内部实现可知，实现逻辑是：现将子任务2放进工作队列，然后执行子任务1，在子任务1处理完后，执行子任务2。该方法等同于：


```java
//任务拆分
t2.fork();
t1.fork();

//合并结果集
t1.join();
t2.join();
```
但需要注意的是，子任务t1,t2调用`join()`方法的顺序要与调用`fork()`方法的顺序相反。因为在后面分析`ForkJoinPool`的入队逻辑时，我们可以知道，t1会放置在工作队列的top位置，而在`join()`时，会从工作队列的top位置取出任务并执行，如果执行的任务不是top位置的任务的话，线程最终只能挂起阻塞，等待通知。有关信息可以 [点击查看](https://yq.aliyun.com/articles/48736?spm=5176.100239.blogcont48739.13.zi6zna)


#### ForkJoinTask重要实现类

##### RecursiveTask源码

```java
public abstract class RecursiveTask<V> extends ForkJoinTask<V> {
    private static final long serialVersionUID = 5232453952276485270L;

    /**
     * The result of the computation.
     */
    V result;

    /**
     * The main computation performed by this task.
     * @return the result of the computation
     */
    //抽象方法。由开发者自己实现
    protected abstract V compute();

    //返回结果处理值
    public final V getRawResult() {
        return result;
    }

    protected final void setRawResult(V value) {
        result = value;
    }

    /**
     * Implements execution conventions for RecursiveTask.
     */
    //执行任务。内部调用compute()方法
    protected final boolean exec() {
        result = compute();
        return true;
    }

}
```
[RecursiveTask](https://docs.oracle.com/javase/7/docs/api/java/util/concurrent/RecursiveTask.html) 类是[ForkJoinTask](https://docs.oracle.com/javase/7/docs/api/java/util/concurrent/ForkJoinTask.html)的子类，该类会有返回值。在并发任务中，如果需要处理结果有返回值的话，继承该类并实现`compute()`即可。


##### RecursiveAction源码

```java
public abstract class RecursiveAction extends ForkJoinTask<Void> {
    private static final long serialVersionUID = 5232453952276485070L;

    /**
     * The main computation performed by this task.
     */
    //抽象方法。由开发者自己实现
    protected abstract void compute();

    /**
     * Always returns {@code null}.
     *
     * @return {@code null} always
     */
    //因为该类是没有返回值的，所以返回null
    public final Void getRawResult() { return null; }

    /**
     * Requires null completion value.
     */
    protected final void setRawResult(Void mustBeNull) { }

    /**
     * Implements execution conventions for RecursiveActions.
     */
    //执行任务。内部调用compute()方法
    protected final boolean exec() {
        compute();
        return true;
    }

}
```
[RecursiveAction](https://docs.oracle.com/javase/7/docs/api/java/util/concurrent/RecursiveAction.html?spm=5176.100239.blogcont48736.5.SXYfZt) 类与[RecursiveTask](https://docs.oracle.com/javase/7/docs/api/java/util/concurrent/RecursiveTask.html) 类非常类似。唯一的区别时 `RecursiveAction`类不会返回执行结果。用法与`RecursiveTask` 类一样。

---

至此，有关`ForkJoinTask` 类大致分析完，下篇文章接着分析 `ForkJoinPool` 类



#### 参考文章

[Fork/Join](https://docs.oracle.com/javase/tutorial/essential/concurrency/forkjoin.html)

[多任务处理（12）——Fork/Join框架（基本使用）](http://blog.csdn.net/yinwenjie/article/details/71524140)

[多任务处理（13）——Fork/Join框架（解决排序问题）](http://blog.csdn.net/yinwenjie/article/details/71915811)

[多任务处理（14）——Fork/Join框架（要点1）](http://blog.csdn.net/yinwenjie/article/details/72520759)

[多任务处理（15）——Fork/Join框架（要点2）](http://blog.csdn.net/yinwenjie/article/details/72639297)

[聊聊并发（八）——Fork/Join框架介绍](http://www.infoq.com/cn/articles/fork-join-introduction)

[Fork/Join框架使用与分析](http://zhangzemiao.com/Java%E5%B9%B6%E5%8F%91%E7%BC%96%E7%A8%8B/Fork-Join%E6%A1%86%E6%9E%B6%E4%BD%BF%E7%94%A8%E4%B8%8E%E5%88%86%E6%9E%90/)

[Java源码分析 - ForkJoinTask篇](http://blog.csdn.net/jsu_9207/article/details/78141682)

[Fork and Join: Java Can Excel at Painless Parallel Programming Too!](http://www.oracle.com/technetwork/articles/java/fork-join-422606.html)

[Doug Lea --《A Java Fork/Join Framework》](http://gee.cs.oswego.edu/dl/papers/fj.pdf)

[ForkJoinPool解读](http://kaimingwan.com/post/java/forkjoinpooljie-du)

