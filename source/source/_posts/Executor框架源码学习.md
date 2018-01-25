---
title: Executor框架源码学习
date: 2018-01-03 15:20:09
tags: java并发
---

#### 前言
在Java中，使用线程来异步执行任务。但是Java线程的创建与销毁需要一定的开销，因此我们可能会考虑使用线程池来复用线程以达到较高的性能。使用[线程池的好处](http://ifeve.com/java-threadpool/)：
1. 降低资源消耗。
2. 提高响应速度。
3. 提高线程的可管理性。

由于线程池的以上好处，JDK1.5中的Executors框架就因此而问世。

Java线程既是工作单元，也是执行单元。从JDK1.5开始，把`工作单元与执行机制分离开`来。工作单元包括Runnable 和 Callable，而执行机制由Executor框架提供。

#### Executors框架类结构图
<a data-flickr-embed="true"  href="https://www.flickr.com/photos/157389715@N05/39446282241/in/dateposted-public/" title="Executors框架类图"><img src="https://farm5.staticflickr.com/4730/39446282241_26ec33702a_z.jpg" width="640" height="558" alt="Executors框架类图"></a>


#### 简单使用

<!-- more -->

```java
public class ExecutorsTest {

    /**
     * 继承有返回结果的Callable接口
     */
    static class Task implements Callable<String> {

        private int seed;

        public Task(int seed) {
            this.seed = seed;
        }

        @Override
        public String call() throws Exception {
            TimeUnit.SECONDS.sleep(5);
            return "线程 " + Thread.currentThread().getName() + " 获取种子数： " + seed;
        }
    }


    /**
     * 继承无返回结果的Runnable接口
     */
    static class TestRunnable implements Runnable {
        @Override
        public void run(){
            System.out.println(Thread.currentThread().getName() + "线程被调用了。");
        }
    }


    public static void main(String[] args) throws ExecutionException, InterruptedException {

        /**
         * 手动创建线程池。因为使用Executors创建线程池会有一些弊端：
         *  1）newFixedThreadPool 和 newSingleThreadExecutor
         *  主要问题是堆积的请求队列可能会耗费非常大的内存，甚至OOM
         *
         *  2）newCachedThreadPool 和 newScheduledThreadPool
         *   主要问题是因为默认的线程数量为Integer.MAX_VALUE，可能会创建数量非常多的线程，甚至OOM
         */
        ExecutorService executor = new ThreadPoolExecutor(Runtime.getRuntime().availableProcessors(), 200,0L,
        TimeUnit.MILLISECONDS, new LinkedBlockingQueue<>(1024), Executors.defaultThreadFactory(), new ThreadPoolExecutor.AbortPolicy());

        Random random = new Random(10000);
        List<Future> list = new ArrayList<>();

        for (int index = 0; index < 1; index++) {
            Future<String> future = executor.submit(new Task(random.nextInt()));
            list.add(future);
        }

        executor.shutdown();
        for (Future<String> future : list) {
            //在执行的任务继承了有返回值的Callable接口时，在获取结果值之前，需要先判断任务是否已结束，因为在没有判断的时候直接调用get获取结果可能会导致线程睡眠
            while(!future.isDone()) {
                System.out.println("Future返回如果没有完成，则一直循环等待，直到Future返回完成");
            }

            System.out.println(future.get());
        }


        
        //##########################################   无返回结果实例  ################################################################

        /**
         * 使用Executors创建线程池
         */
        ExecutorService executorService = Executors.newFixedThreadPool(Runtime.getRuntime().availableProcessors());
        for (int i = 0; i < 200; i++){
            executorService.execute(new TestRunnable());
        }

        executorService.shutdown();

        //在执行的任务继承了无返回值的Runnable接口时，如果有要求在所有任务执行完后做一些工作A，则需要在开始工作A之前，需判断当前线程池是否已中断
        while (! executorService.isTerminated()) {
            System.out.println("任务执行中。。。。");
        }
        System.out.println("完成");

    }
}
```

#### Executors源码

Java API还提供的工具类Executors，可以帮我们针对不同的应用场景创建不同的Executor实现类。

```java
//创建一个FixedThreadPool，corePoolSize和max size 都为nThreads。线程存好时间是0秒。底层队列使用LinkedBlockingQueue。
public static ExecutorService newFixedThreadPool(int nThreads) {
    return new ThreadPoolExecutor(nThreads, nThreads,
                                  0L, TimeUnit.MILLISECONDS,
                                  new LinkedBlockingQueue<Runnable>());
}

//创建一个WorkStealingPool，串行数为parallelism。底层使用先进先出队列（FIFO_QUEUE）
public static ExecutorService newWorkStealingPool(int parallelism) {
    return new ForkJoinPool
        (parallelism,
         ForkJoinPool.defaultForkJoinWorkerThreadFactory,
         null, true);
}


//创建一个SingleThreadExecutor，corePoolSize和max size 都为1。线程存好时间是0秒。底层队列使用LinkedBlockingQueue。
public static ExecutorService newSingleThreadExecutor() {
    return new FinalizableDelegatedExecutorService
        (new ThreadPoolExecutor(1, 1,
                                0L, TimeUnit.MILLISECONDS,
                                new LinkedBlockingQueue<Runnable>()));
}


//创建一个CachedThreadPoo，corePoolSize为0，max size 为Integer.MAX_VALUE。线程存好时间是60秒。底层队列使用SynchronousQueue。
public static ExecutorService newCachedThreadPool() {
    return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                  60L, TimeUnit.SECONDS,
                                  new SynchronousQueue<Runnable>());
}

//创建一个SingleThreadScheduledExecutor，corePoolSize为0，max size 为Integer.MAX_VALUE。线程存好时间是0秒。底层队列使用DelayedWorkQueue。
public static ScheduledExecutorService newSingleThreadScheduledExecutor() {
    return new DelegatedScheduledExecutorService
        (new ScheduledThreadPoolExecutor(1));
}


//创建一个ScheduledThreadPool，corePoolSize为0，max size 为Integer.MAX_VALUE。线程存好时间是0秒。底层队列使用DelayedWorkQueue。
public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize) {
    return new ScheduledThreadPoolExecutor(corePoolSize);
}
```

由工具类Executors获取具体执行器后，会调用相应的方法将任务添加到执行队列中。其中 [ForkJoinPool](https://houlong123.github.io/2017/12/25/Fork-Join框架源码学习之ForkJoinPool学习/) 我们已经学习过，本章已最常用的`ThreadPoolExecutor`进行分析。关于`ScheduledThreadPoolExecutor`类日后再说。

#### ThreadPoolExecutor源码分析

##### 数据结构

```java
public class ThreadPoolExecutor extends AbstractExecutorService {
    /**
     * 主池控制状态ctl是包含两个概念字段的原子整数.
     *    1) workerCount，表示有效的线程数
     *    2) runState，指示是否运行，关闭等
     *
     * 使用了Integer类型来保存，高3位保存runState，低29位保存workerCount
     */
    private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));

    private static final int COUNT_BITS = Integer.SIZE - 3;     //29
    private static final int CAPACITY   = (1 << COUNT_BITS) - 1;    //0001 1111 1111 1111 1111 1111 1111 1111

    // runState is stored in the high-order bits
   /**
     *   各个状态说明：  
     *   RUNNING:  Accept new tasks and process queued tasks
     *    SHUTDOWN: Don't accept new tasks, but process queued tasks
     *    STOP:     Don't accept new tasks, don't process queued tasks,
     *             and interrupt in-progress tasks
     *    TIDYING:  All tasks have terminated, workerCount is zero,
     *             the thread transitioning to state  TIDYING
     *             will run the terminated() hook method
     *    TERMINATED: terminated() has completed
     *
     *  各个状态之间的转换：
     *  RUNNING -> SHUTDOWN
     *    On invocation of shutdown(), perhaps implicitly in finalize()
     *      
        (RUNNING or SHUTDOWN) -> STOP
     *    On invocation of shutdownNow()
     * 
        SHUTDOWN -> TIDYING
     *    When both queue and pool are empty
     * 
        STOP -> TIDYING
     *    When pool is empty
     
     *  TIDYING -> TERMINATED
     *    When the terminated() hook method has completed
     *
     */
    //线程池的运行状态
    // -1 的二进制：1111 1111 1111 1111 1111 1111 1111 1111
    private static final int RUNNING    = -1 << COUNT_BITS; //1110 0000 0000 0000 0000 0000 0000 0000  接受新task且处理排队的任务
    private static final int SHUTDOWN   =  0 << COUNT_BITS;  // 0:不接受新任务，处理排队的任务
    private static final int STOP       =  1 << COUNT_BITS; // 0010 0000 0000 0000 0000 0000 0000 0000  不接受新任务、不处理排队的任务，打断正在执行的任务
    private static final int TIDYING    =  2 << COUNT_BITS; // 0100 0000 0000 0000 0000 0000 0000 0000  所有的任务都已结束，工作线程数量等于0，将要执行terminate方法
    private static final int TERMINATED =  3 << COUNT_BITS; // 0110 0000 0000 0000 0000 0000 0000 0000  terminate()方法已经执行完毕
    
     // 等待执行的任务的阻塞队列
    private final BlockingQueue<Runnable> workQueue;
    private final ReentrantLock mainLock = new ReentrantLock();
    
    //工作线程集合
    private final HashSet<Worker> workers = new HashSet<Worker>();
    private volatile ThreadFactory threadFactory;
    //饱和策略。
    private volatile RejectedExecutionHandler handler;
    //线程活跃保持时间
    private volatile long keepAliveTime;
     //线程池的基本大小
    private volatile int corePoolSize;
    //线程池允许创建的最大线程数
    private volatile int maximumPoolSize;
    //内部工作者数据结构，继承AQS，并实现了Runnable接口。
    // 我们实现了一个简单的非重入互斥锁，而不是使用ReentrantLock，因为我们不希望工作任务在调用像setCorePoolSize这样的池控制方法时能够重新获取锁。
    private final class Worker extends AbstractQueuedSynchronizer implements Runnable {
        ...
    }
}
```
在 ThreadPoolExecutor 类内部结构中，定义的属性有
1. 线程池的各种状态
    
    ```
    RUNNING:  能接受新提交的任务，并且也能处理阻塞队列中的任务
    SHUTDOWN: 关闭状态，不再接受新提交的任务，但却可以继续处理阻塞队列中已保存的任务。
    STOP: 不能接受新任务，也不处理队列中的任务，会中断正在处理任务的线程。
    IDYING:  如果所有的任务都已终止了，workerCount (有效线程数) 为0，线程池进入该状态后会调用 terminated() 方法进入TERMINATED 状态。
    TERMINATED: 在terminated() 方法执行完后进入该状态。
    ```
    线程池的状态转换过程：
    <a data-flickr-embed="true"  href="https://www.flickr.com/photos/157389715@N05/39435374792/in/dateposted-public/" title="77c2251dacf9e26fe1ba1e1e54e62a65"><img src="https://farm5.staticflickr.com/4732/39435374792_6e17727a82_z.jpg" width="640" height="252" alt="77c2251dacf9e26fe1ba1e1e54e62a65"></a>

2. 主池控制状态ctl

    ctl相关的计算的方法：
    
    ```java
    // Packing and unpacking ctl
    private static int runStateOf(int c)     { return c & ~CAPACITY; }  //获取运行状态
    private static int workerCountOf(int c)  { return c & CAPACITY; }   //获取活动线程数
    private static int ctlOf(int rs, int wc) { return rs | wc; }        //获取运行状态和活动线程数的值

    ```

3. 等待执行的任务的阻塞队列。在线程池中的线程数不小于`corePoolSize` 时，会将新提交的任务丢进阻塞队列中。
4. 可重入ReentrantLock对象锁
5. 工作线程集合。用于维护线程池中的所有工作线程。工作线程会封装在内部类`Worker`中。仅仅在获取可重入锁后才能访问。
6. 饱和策略RejectedExecutionHandler类。在阻塞队列也满后，会使用RejectedExecutionHandler类对新提交任务进行处理。
7. 其他通过构造函数传递的一些基本参数。如：corePoolSize， maximumPoolSize，threadFactory等。




##### execute源码
由上面实例可知，在获取到执行器 `ExecutorService` 后，会直接调用其execute()方法。

```java
public void execute(Runnable command) {
    //执行任务不能为空
    if (command == null)
        throw new NullPointerException();
    /*
     * Proceed in 3 steps:
     *
     * 1. If fewer than corePoolSize threads are running, try to
     * start a new thread with the given command as its first
     * task.  The call to addWorker atomically checks runState and
     * workerCount, and so prevents false alarms that would add
     * threads when it shouldn't, by returning false. （如果线程运行的数量少于corePoolSize，则尝试将给定任务作为第一个任务来创建新线程）
     *
     * 2. If a task can be successfully queued, then we still need
     * to double-check whether we should have added a thread
     * (because existing ones died since last checking) or that
     * the pool shut down since entry into this method. So we
     * recheck state and if necessary roll back the enqueuing if
     * stopped, or start a new thread if there are none.（如果将任务成功入队后，需要再次检查是否应该添加一个线程或线程池是否已关闭）
     *
     * 3. If we cannot queue task, then we try to add a new
     * thread.  If it fails, we know we are shut down or saturated
     * and so reject the task.（如果不能将任务入队，则尝试添加一个线程，如果失败，则可知任务池已关闭或池已饱和，则拒绝任务）
     */
    int c = ctl.get();
    //获取活动线程数。如果小于corePoolSize，则新建工作线程
    if (workerCountOf(c) < corePoolSize) {

         /*
          * addWorker中的第二个参数表示限制添加线程的数量是根据corePoolSize来判断还是maximumPoolSize来判断；
          * 如果为true，根据corePoolSize来判断；
          * 如果为false，则根据maximumPoolSize来判断
          */
        if (addWorker(command, true))
            return;

        /*
         * 如果添加失败，则重新获取ctl值
         */
        c = ctl.get();
    }
    //如果当前正在运行状态、且任务添加到队列成功
    if (isRunning(c) && workQueue.offer(command)) {
        int recheck = ctl.get();

        // 再次判断线程池的运行状态，如果不是运行状态，由于之前已经把command添加到workQueue中了，
        // 这时需要移除该command
        // 执行过后通过handler使用拒绝策略对该任务进行处理，整个方法返回
        if (! isRunning(recheck) && remove(command))
            reject(command);
        else if (workerCountOf(recheck) == 0)   //当前工作线程为零，尝试添加空任务

            //在workerCountOf(recheck) == 0时执行addWorker(null, false);也是为了保证线程池在RUNNING状态下必须要有一个线程来执行任务
            addWorker(null, false);
    }

    /*
     * 如果执行到这里，有两种情况：
     * 1. 线程池已经不是RUNNING状态；
     * 2. 线程池是RUNNING状态，但workerCount >= corePoolSize并且workQueue已满。
     * 这时，再次调用addWorker方法，但第二个参数传入为false，将线程池的有限线程数量的上限设置为maximumPoolSize；
     * 如果失败则拒绝该任务
     */
    else if (!addWorker(command, false))    //尝试添加任务失败，线程池已饱和
        reject(command);
}
```
在代码注释中，也已说明，该方法分三步处理：
1. 如果线程运行的数量少于corePoolSize，则尝试将给定任务作为第一个任务来创建新线程。添加新线程成功，则方法返回；否则重新获取ctl值，执行第下步。
2. 如果线程池处于运行状态且任务成功添加到阻塞队列后，需要再次检查线程池是否已关闭。已关闭则将已入队的任务移除，使用饱和策略处理新提交任务。没有关闭，则判断是否需要添加一个工作线程
3. 如果线程池为非运行状态或阻塞队列已满，则尝试新加工作线程，如果失败，则使用饱和策略处理新提交任务。

##### addWorker源码

```java
private boolean addWorker(Runnable firstTask, boolean core) {
    //这里使用了goto
    retry:
    for (;;) {

        //获取池的运行状态
        int c = ctl.get();
        int rs = runStateOf(c);

        /*
         * 这个if判断
         * 如果rs >= SHUTDOWN，则表示此时不再接收新任务；
         * 接着判断以下3个条件，只要有1个不满足，则返回false：
         * 1. rs == SHUTDOWN，这时表示关闭状态，不再接受新提交的任务，但却可以继续处理阻塞队列中已保存的任务
         * 2. firsTask为空
         * 3. 阻塞队列不为空
         *
         * 首先考虑rs == SHUTDOWN的情况
         * 这种情况下不会接受新提交的任务，所以在firstTask不为空的时候会返回false；
         * 然后，如果firstTask为空，并且workQueue也为空，则返回false，
         * 因为队列中已经没有任务了，不需要再添加线程了
         */
        // Check if queue empty only if necessary.
        if (rs >= SHUTDOWN &&
            ! (rs == SHUTDOWN &&    //池为关闭状态
               firstTask == null && //首任务为null
               ! workQueue.isEmpty()))

            //非运行状态且(运行状态不为SHUTDOWN或任务为空或队列为空)
            return false;

        for (;;) {
            //获取活动线程数
            int wc = workerCountOf(c);

            //CAPACITY=2^29-1，线程超过容量不创建新线程
            if (wc >= CAPACITY ||
                wc >= (core ? corePoolSize : maximumPoolSize))
                return false;

            //增加工作线程
            if (compareAndIncrementWorkerCount(c))
                //满足创建线程条件且增加workCount成功跳出循环
                break retry;
            c = ctl.get();  // Re-read ctl
            if (runStateOf(c) != rs)
                //线程池状态发生变化重新循环
                continue retry;
            // else CAS failed due to workerCount change; retry inner loop
        }
    }

    boolean workerStarted = false;
    boolean workerAdded = false;
    Worker w = null;
    try {
        w = new Worker(firstTask);  //创建线程
        final Thread t = w.thread;
        if (t != null) {
            final ReentrantLock mainLock = this.mainLock;
            mainLock.lock(); //获取锁，维护同步状态
            try {
                // Recheck while holding lock.
                // Back out on ThreadFactory failure or if
                // shut down before lock acquired.
                int rs = runStateOf(ctl.get());

                //线程池处于运行状态或者（线程池已经SHUTDOWN且头任务为空）
                // rs < SHUTDOWN表示是RUNNING状态；
                // 如果rs是RUNNING状态或者rs是SHUTDOWN状态并且firstTask为null，向线程池中添加线程。
                // 因为在SHUTDOWN时不会在添加新的任务，但还是会执行workQueue中的任务

                if (rs < SHUTDOWN ||
                    (rs == SHUTDOWN && firstTask == null)) {
                    if (t.isAlive()) // precheck that t is startable
                        throw new IllegalThreadStateException();
                    //线程引用存储
                    workers.add(w);
                    int s = workers.size();
                    // largestPoolSize记录着线程池中出现过的最大线程数量
                    if (s > largestPoolSize)
                        //更新最大线程数量
                        largestPoolSize = s;
                    workerAdded = true;
                }
            } finally {
                mainLock.unlock();
            }

            //如果添加成功，则启动线程
            if (workerAdded) {
                t.start();
                workerStarted = true;
            }
        }
    } finally {
        //添加失败的后续处理
        if (! workerStarted)
            addWorkerFailed(w);
    }
    return workerStarted;
}
```
功能逻辑：
1. 进入外部for循环。
    1. 获取线程池的运行状态rs
    2. 如果线程池的运行状态为非运行状态，只有在状态为 `SHUTDOWN` 且任务队列不为空的情况下，执行下一步；否则，直接返回FALSE。（因为在池状态为`SHUTDOWN`时，虽不能接收新任务，但是却可以处理已提交任务，所以可以往下执行。其他非运行状态，则不允许往下执行）。
    3. 进入内部for循环
        1. 获取活动线程数
        2. 如果活跃线程数超过容量`CAPACITY=2^29-1`，则不创建新线程，返回FALSE。否则执行下一步。
        3. 增加工作线程数。成功，跳出外部for循环，执行第2步；失败，重新获取ctl值，如果线程池状态发生改变，则从外部for循环重新开始处理。

2. 根据新提交任务，创建工作线程。
3. 工作线程不为空的情况下，如果线程池处于运行状态或者（线程池已经SHUTDOWN且头任务为空），则将新创建的工作线程添加到线程集合`workers`中，然后启动工作线程。
4. 如果工作线程启动失败后，则调用`addWorkerFailed()`方法将新创建的工作线程从 线程集合`workers` 中移除。


##### addWorkerFailed源码

```java
//回滚工作线程创建。
private void addWorkerFailed(Worker w) {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        if (w != null)
            workers.remove(w);  //如果线程存在，移除
        decrementWorkerCount(); //减少工作线程数
        tryTerminate();  //中断
    } finally {
        mainLock.unlock();
    }
}
```
该方法的主要功能是：在新建工作线程启动失败后，将新建工作线程从线程集合`workers` 中移除，并将工作线程数减一，然后尝试中断线程池。

##### tryTerminate源码

```java
final void tryTerminate() {
    for (;;) {
        int c = ctl.get();

        /*
         * 当前线程池的状态为以下几种情况时，直接返回：
         * 1. RUNNING，因为还在运行中，不能停止；
         * 2. TIDYING或TERMINATED，因为线程池中已经没有正在运行的线程了；
         * 3. SHUTDOWN并且等待队列非空，这时要执行完workQueue中的task；
         */
        if (isRunning(c) ||
            runStateAtLeast(c, TIDYING) ||
            (runStateOf(c) == SHUTDOWN && ! workQueue.isEmpty()))
            return;

        //如果运行数量不为0，尝试interrupt一个线程
        if (workerCountOf(c) != 0) { // Eligible to terminate
            interruptIdleWorkers(ONLY_ONE);
            return;
        }

        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            if (ctl.compareAndSet(c, ctlOf(TIDYING, 0))) { //尝试设置线程池状态为TIDYING成功则准备调用terminated方法
                try {
                    terminated();
                } finally {
                    //设置状态
                    ctl.set(ctlOf(TERMINATED, 0));
                    termination.signalAll();
                }
                return;
            }
        } finally {
            mainLock.unlock();
        }
        // else retry on failed CAS
    }
}
```
功能逻辑：
1. 判断是否需要中断。当前线程池的状态为以下几种情况时，方法直接返回结束。
    
    ```java
    1. RUNNING，因为还在运行中，不能停止；
    2. TIDYING或TERMINATED，因为线程池中已经没有正在运行的线程了；
    3. SHUTDOWN并且等待队列非空，这时要执行完workQueue中的task；
    ```
2. 如果需要中断且工作线程数不为0，则调用 `interruptIdleWorkers()` 尝试中断一个线程，然后返回结束。
3. 如果工作线程数为0，则尝试将线程池状态修改为 `TIDYING`，成功后，调用terminated方法中断线程池。然后设置状态为 `TERMINATED`。最后唤醒所以沉睡线程。

##### interruptIdleWorkers源码

```java
private void interruptIdleWorkers() {
    interruptIdleWorkers(false);
}

private void interruptIdleWorkers(boolean onlyOne) {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        //遍历工作线程集合
        for (Worker w : workers) {
            Thread t = w.thread;
            if (!t.isInterrupted() && w.tryLock()) {    //如果线程没有中断且成功获取锁，则中断线程
                try {
                    t.interrupt();
                } catch (SecurityException ignore) {
                } finally {
                    w.unlock();
                }
            }

            //如果只中断一个线程的话，则退出
            if (onlyOne)
                break;
        }
    } finally {
        mainLock.unlock();
    }
}
```
从代码实现逻辑可知，interruptIdleWorkers方法主要就是循环工作集合，然后将工作线程全部中断。


##### Worker源码解析
在 `addWorker()` 方法中，添加工作线程时，会出现 `Worker` 类，该类封装了工作线程和任务。在上面的`ThreadPoolExecutor` 的数据结构中，我们已知ThreadPool维护的其实就是一组Worker对象。线程池中的每一个线程被封装成一个Worker对象。

###### Worker数据结构

```java
//内部工作者数据结构，继承AQS，并实现了Runnable接口。
// 我们实现了一个简单的非重入互斥锁，而不是使用ReentrantLock，因为我们不希望工作任务在调用像setCorePoolSize这样的池控制方法时能够重新获取锁。
private final class Worker extends AbstractQueuedSynchronizer implements Runnable {
    /** Thread this worker is running in.  Null if factory fails. */
    final Thread thread;
    /** Initial task to run.  Possibly null. */
    Runnable firstTask;
    /** Per-thread task counter */
    volatile long completedTasks;

    /**
     * Creates with given first task and thread from ThreadFactory.
     * @param firstTask the first task (null if none)
     */
    Worker(Runnable firstTask) {
        //把state变量设置为-1，这样做的目的是禁止任务没有执行时被中断
        setState(-1); // inhibit interrupts until runWorker
        this.firstTask = firstTask; //保存处理任务
        this.thread = getThreadFactory().newThread(this);   //利用线程工厂创建线程
    }
}
```
Worker类继承了AQS，并实现了Runnable接口，注意其中的firstTask和thread属性：firstTask用它来保存传入的任务；thread是在调用构造方法时通过ThreadFactory来创建的线程，是用来处理任务的线程。所以在`addWorker()` 方法中 `t.start();` 一句会调用 `Worker类` 的 `run()` 方法。

###### run源码

```java
//将方法代理到外部ThreadPoolExecutor的runWorker方法
public void run() {
    runWorker(this);
}
```
在Worker类中的run方法调用了runWorker方法来执行任务。

###### runWorker源码

```java
final void runWorker(Worker w) {
    //获取当前线程
    Thread wt = Thread.currentThread();
    //firstTask就是要执行的任务
    Runnable task = w.firstTask;
    w.firstTask = null;
    //允许中断。由于在创建Worker时，使用了setState(-1)方法将state设置为了-1。禁止任务在没有执行时被中断，
    // 所以runWorker方法中会先调用Worker对象的unlock方法将state设置为0。因为w.lock()中，是以state==0为前提的
    w.unlock(); // allow interrupts

    // 是否因为异常退出循环
    boolean completedAbruptly = true;
    try {
        while (task != null || (task = getTask()) != null) {    //在循环中不断取任务
            w.lock();
            // If pool is stopping, ensure thread is interrupted;
            // if not, ensure thread is not interrupted.  This
            // requires a recheck in second case to deal with
            // shutdownNow race while clearing interrupt
            //下面的if语句功能是：在线程池为STOP时，要确保线程中断
            if ((runStateAtLeast(ctl.get(), STOP) ||
                 (Thread.interrupted() &&
                  runStateAtLeast(ctl.get(), STOP))) &&
                !wt.isInterrupted())
                wt.interrupt();

            try {
                //如果线程池状态<=SHUTDOWN且线程没有interrupted则执行任务
                beforeExecute(wt, task);
                Throwable thrown = null;
                try {
                    //执行任务
                    task.run();
                } catch (RuntimeException x) {
                    thrown = x; throw x;
                } catch (Error x) {
                    thrown = x; throw x;
                } catch (Throwable x) {
                    thrown = x; throw new Error(x);
                } finally {
                    //任务执行完成后，做的事情
                    afterExecute(task, thrown);
                }
            } finally {
                // 将任务清空
                task = null;
                //完成任务数累加
                w.completedTasks++;
                //释放锁
                w.unlock();
            }
        }
        completedAbruptly = false;
    } finally {
        //执行完后的处理
        processWorkerExit(w, completedAbruptly);
    }
}
```
该方法的功能就是：重复获取任务，然后执行。先执行当前工作线程所拥有的首任务，如果没有，则从任务列表中获取，依次执行。代码逻辑：
1. 在首任务不为空或者调用方法`getTask()`获取的任务不为空，执行下一步；否则，执行执行processWorkerExit()方法。
2. 如果线程池正在停止，那么要保证当前线程是中断状态，否则要保证当前线程不是中断状态。
3. 调用task.run()执行任务；
4. runWorker方法执行完毕，也代表着Worker中的run方法执行完毕，销毁线程。


###### getTask源码

```java
private Runnable getTask() {
    // timeOut变量的值表示上次从阻塞队列中取任务时是否超时
    boolean timedOut = false; // Did the last poll() time out?

    for (;;) {  //再死循环中取任务
        int c = ctl.get();
        int rs = runStateOf(c);

        // Check if queue empty only if necessary.
        //线程池状态不可执行任务。从此处可知，在线程池的状态为SHUTDOWN，且工作队列不为空时，依然可以执行已提交任务。
        // 但是如果为rs >= STOP（rs为STOP，TIDYING，TERMINATED）时，不会执行已提交任务
        if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
            decrementWorkerCount();
            return null;
        }

        //获取工作线程数
        int wc = workerCountOf(c);

        // Are workers subject to culling?
        //线程是否需要超时退出
        boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;

        //如果线程数量过大或者(超时且(线程数>1或者队列空))
        /*
         * wc > maximumPoolSize的情况是因为可能在此方法执行阶段同时执行了setMaximumPoolSize方法；
         * timed && timedOut 如果为true，表示当前操作需要进行超时控制，并且上次从阻塞队列中获取任务发生了超时
         * 接下来判断，如果有效线程数量大于1，或者阻塞队列是空的，那么尝试将workerCount减1；
         * 如果减1失败，则返回重试。
         * 如果wc == 1时，也就说明当前线程是线程池中唯一的一个线程了。
         */
        if ((wc > maximumPoolSize || (timed && timedOut))
            && (wc > 1 || workQueue.isEmpty())) {
            if (compareAndDecrementWorkerCount(c))
                return null;
            continue;
        }

        try {
            //从工作队列中获取任务
            Runnable r = timed ?
                workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                workQueue.take();
            if (r != null)
                return r;
            timedOut = true;
        } catch (InterruptedException retry) {
            timedOut = false;
        }
    }
}
```
getTask方法用来从阻塞队列中取任务。代码逻辑：
1. 进入for无限循环。
2. 获取线程池的运行状态。
3. 如果线程池的运行状态为`非运行状态(SHUTDOWN，STOP，TIDYING，TERMINATED)`且任务队列为空，则直接返回null；如果线程池的运行状态为`SHUTDOWN且任务队列不为空`，则往下执行。

    <font color=gray>备注：在线程池状态为SHUTDOWN时，不能接收新任务，但是可以执行已提交任务，所以getTask会获取任务返回；如果线程池状态为STOP，TIDYING，TERMINATED，则既不能接收任务，也不能处理已提交任务，所以getTask会直接返回null</font>
4. 获取工作线程的数量
5. 控制线程池的有效线程数量。
6. 从阻塞队列中获取任务
7. 如果任务不为空，则返回任务，方法结束；否则继续for循环。


###### processWorkerExit源码

```java
//从工作集中删除线程。并尝试中断线程池或者如果completedAbruptly为true或工作线程少于corePoolSize，或队列不为空，则替换工作线程
private void processWorkerExit(Worker w, boolean completedAbruptly) {

    // 如果completedAbruptly值为true，则说明线程执行时出现了异常，需要将workerCount减1；
    // 如果线程执行时没有出现异常，说明在getTask()方法中已经已经对workerCount进行了减1操作，这里就不必再减了。
    if (completedAbruptly) // If abrupt, then workerCount wasn't adjusted
        //这里ctl自减1，直到成功
        decrementWorkerCount();

    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        //记录完成的任务的个数
        completedTaskCount += w.completedTasks;
        //这里清除掉worker
        workers.remove(w);
    } finally {
        mainLock.unlock();
    }

    //尝试转换到terminate状态
    tryTerminate();

    int c = ctl.get();

    /*
     * 当线程池是RUNNING或SHUTDOWN状态时，如果worker是异常结束，那么会直接addWorker；
     * 如果allowCoreThreadTimeOut=true，并且等待队列有任务，至少保留一个worker；
     * 如果allowCoreThreadTimeOut=false，workerCount不少于corePoolSize。
     */
    //可执行任务状态：SHUTDOWN 或 RUNNING
    if (runStateLessThan(c, STOP)) {
        if (!completedAbruptly) {
            int min = allowCoreThreadTimeOut ? 0 : corePoolSize;
            if (min == 0 && ! workQueue.isEmpty())
                min = 1;

            //如果当前工作线程大于corePoolSize结束方法
            if (workerCountOf(c) >= min)
                return; // replacement not needed
        }

        //否则新增线程，保证线程池中缓存的线程数,线程池填充
        addWorker(null, false);
    }
}
```
该方法的主要功能是：销毁工作线程，尝试中断线程池，并维护维护池中有效线程。代码逻辑：
1. 从工作集合中清除指定工作线程
2. 尝试中断线程池
3. 如果线程池的状态为可执行状态（SHUTDOWN 或 RUNNING），则维护池中有效工作线程。


#### 整体处理流程
<a data-flickr-embed="true"  href="https://www.flickr.com/photos/157389715@N05/27688438409/in/dateposted-public/" title="fb66e616aa6da061f92d0b4eebb64b7f"><img src="https://farm5.staticflickr.com/4597/27688438409_90c02f4ca5_z.jpg" width="640" height="500" alt="fb66e616aa6da061f92d0b4eebb64b7f"></a>

###### shutdown源码

```java
public void shutdown() {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        checkShutdownAccess();
        //状态切换到SHUTDOWN
        advanceRunState(SHUTDOWN);
        //中断工作线程集合中的一个线程
        interruptIdleWorkers();
        //hook方法
        onShutdown(); // hook for ScheduledThreadPoolExecutor
    } finally {
        mainLock.unlock();
    }
    tryTerminate();
}
```
该方法将线程池的状态改为`SHUTDOWN`，然后中断工作集合中的一个线程。


###### shutdownNow源码

```java
public List<Runnable> shutdownNow() {
    List<Runnable> tasks;
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        checkShutdownAccess();
        //状态切换到STOP
        advanceRunState(STOP);
        //中断工作线程集合中的所有线程
        interruptWorkers();
        //提取未完成任务
        tasks = drainQueue();
    } finally {
        mainLock.unlock();
    }
    tryTerminate();
    return tasks;
}
```
shutdownNow方法与shutdown方法类似，不同的地方在于：

1. 设置状态为STOP；
2. 中断所有工作线程，无论是否是空闲的；
3. 取出阻塞队列中没有被执行的任务并返回。

shutdownNow方法执行完之后调用tryTerminate方法，该方法在上文已经分析过了，目的就是使线程池的状态设置为TERMINATED。







#### 参考文章
[Executor框架使用与分析一](http://zhangzemiao.com/Java%E5%B9%B6%E5%8F%91%E7%BC%96%E7%A8%8B/Executor%E6%A1%86%E6%9E%B6%E4%BD%BF%E7%94%A8%E4%B8%8E%E5%88%86%E6%9E%90%E4%B8%80/)

[Executor框架使用与分析二](http://zhangzemiao.com/Java%E5%B9%B6%E5%8F%91%E7%BC%96%E7%A8%8B/Executor%E6%A1%86%E6%9E%B6%E4%BD%BF%E7%94%A8%E4%B8%8E%E5%88%86%E6%9E%90%E4%BA%8C/)

[Executor框架使用与分析三](http://zhangzemiao.com/Java%E5%B9%B6%E5%8F%91%E7%BC%96%E7%A8%8B/Executor%E6%A1%86%E6%9E%B6%E4%BD%BF%E7%94%A8%E4%B8%8E%E5%88%86%E6%9E%90%E4%B8%89/)

[【Java多线程】Executor框架的详解](http://www.cnblogs.com/study-everyday/p/6737428.html)

[深入理解 Java 线程池：ThreadPoolExecutor](https://juejin.im/entry/58fada5d570c350058d3aaad)
