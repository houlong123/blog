---
title: java并发学习之StampedLock学习
date: 2017-10-15 14:33:39
tags: java并发锁
---

#### 前言
&nbsp;&nbsp;&nbsp;&nbsp;在 [ReentrantReadWriteLock源码学习](https://houlong123.github.io/2017/09/28/java并发学习之ReentrantReadWriteLock学习/) 中，学习了读写锁的源码实现。但是在 <font color='red'>`读多写少`</font> 的情况下，使用 ReentrantReadWriteLock 可能会使写入线程遭遇饥饿（Starvation）问题，也就是写入线程迟迟无法竞争到锁而一直处于等待状态。`StampedLock锁是对ReentrantReadWriteLock锁的一种改进，即StampedLock是一种改进的读写锁`。

&nbsp;&nbsp;&nbsp;&nbsp; StampedLock锁有三种模式：写，读，乐观读。StampedLock锁的`状态是由版本和模式两个部分组成`，锁获取方法返回一个数字作为票据stamp，它用相应的锁状态表示并控制访问。


#### 使用

<!-- more -->

```java
public class Point {
   private double x, y;
   private final StampedLock sl = new StampedLock();

   public void move(double deltaX, double deltaY) { // an exclusively locked method
       /**
        * stampedLock调用writeLock和unlockWrite时候都会导致stampedLock的state（锁状态）属性值的变化
        * 即每次高8位 +1，直到加到最大值，然后从0重新开始. 
        * 当锁被写模式所占有，没有读或者乐观的读操作能够成功。
        */
     long stamp = sl.writeLock();
     try {
       x += deltaX;
       y += deltaY;
     } finally {
         //释放写锁
       sl.unlockWrite(stamp);
     }
   }

   public double distanceFromOrigin() { // A read-only method
       /**
        * tryOptimisticRead是一个乐观的读，使用这种锁的读不阻塞写
        * 每次读的时候得到一个当前的stamp值
       */
     long stamp = sl.tryOptimisticRead();
     double currentX = x, currentY = y;

     /**
      * validate()方法校验从调用tryOptimisticRead()之后有没有线程获得写锁，
      *     true:无写锁，state与stamp匹配
      *     false:有写锁，state与stamp不匹配，或者stamp=0（调用tryOptimisticRead()时已经被其他线程持有写锁）
     */
     if (!sl.validate(stamp)) {
         /**
          * 被写锁入侵需要使用悲观读锁重读，阻塞写锁（防止再次出现脏数据） 或者 等待写锁释放锁
          * 当然重读的时候还可以使用tryOptimisticRead，此时需要结合循环了，即类似CAS方式
          */
        stamp = sl.readLock();
        try {
          currentX = x;
          currentY = y;
        } finally {
            //释放读锁
           sl.unlockRead(stamp);
        }
     }
     return (currentX +  currentY);
   }

   /**
    * 初始化 x,y
    * @param newX
    * @param newY
    */
   public void moveIfAtOrigin(double newX, double newY) { 
     // 以乐观读锁的方式开始，而不是悲观读锁
     long stamp = sl.readLock();
     try {
       while (x == 0.0 && y == 0.0) {
         /**
          * 尝试转换成写锁
          *     0：获得写锁失败
          *     非0：获得写锁成功
          */
         long ws = sl.tryConvertToWriteLock(stamp);
         //持有写锁
         if (ws != 0L) {
           stamp = ws;
           x = newX;
           y = newY;
           break;
         }
         //否则调用writeLock()直到获得写锁
         else {
           sl.unlockRead(stamp);
           stamp = sl.writeLock();
         }
       }
     } finally {
         //释放锁，可以是writeLock，也可是readLock
         sl.unlock(stamp);
     }
   }
}
```

#### 源码解析

##### 内部数据结构

```java
public class StampedLock implements java.io.Serializable {
    /** Number of processors, for spin control */
    /** 获取服务器CPU核数 */
    private static final int NCPU = Runtime.getRuntime().availableProcessors();

    /** Maximum number of retries before enqueuing on acquisition */
    /** 线程入队列前自旋次数 */
    private static final int SPINS = (NCPU > 1) ? 1 << 6 : 0;

    /** Maximum number of retries before blocking at head on acquisition */
    /** 队列头结点自旋获取锁最大失败次数后再次进入队列 */
    private static final int HEAD_SPINS = (NCPU > 1) ? 1 << 10 : 0;

    /** Maximum number of retries before re-blocking */
    //重新阻止之前的最大重试次数
    private static final int MAX_HEAD_SPINS = (NCPU > 1) ? 1 << 16 : 0;

    /** The period for yielding when waiting for overflow spinlock */
    //等待溢出螺旋锁时产生屈服期
    private static final int OVERFLOW_YIELD_RATE = 7; // must be power 2 - 1

    /** The number of bits to use for reader count before overflowing */
    private static final int LG_READERS = 7;
    
    /**
     *   有没有写线程获取到了写状态只需判断：state < WBIT
         读状态是否超出：(state & ABITS) < RFULL
         获取读状态：  state + RUNIT(或者readerOverflow + 1)
         获取写状态:      state + WBIT
         释放读状态：  state - RUNIT(或者readerOverflow - 1)
         释放写状态：  (s += WBIT) == 0L ? ORIGIN : s
         是否为写锁：  (state & WBIT) != 0L
         是否为读锁：  (state & RBITS) != 0L
     */

    // Values for lock state and stamp operations
    private static final long RUNIT = 1L;
    private static final long WBIT  = 1L << LG_READERS; //0000 0000 0000 0000  0000 0000 0000  0000 0000 0000 0000  0000 0000 1000 0000
    private static final long RBITS = WBIT - 1L;        //0000 0000 0000 0000  0000 0000 0000  0000 0000 0000 0000  0000 0000 0111 1111
    private static final long RFULL = RBITS - 1L;       //0000 0000 0000 0000  0000 0000 0000  0000 0000 0000 0000  0000 0000 0111 1110
    private static final long ABITS = RBITS | WBIT;     //0000 0000 0000 0000  0000 0000 0000  0000 0000 0000 0000  0000 0000 1111 1111
    private static final long SBITS = ~RBITS;           //1111 1111 1111 1111  1111 1111 1111  1111 1111 1111 1111  1111 1111 1000 0000  note overlap with ABITS

    // Initial value for lock state; avoid failure value zero
    //锁state初始值，第9位为1，避免算术时和0冲突
    private static final long ORIGIN = WBIT << 1;   //0000 0000 0000 0000  0000 0000 0000  0000 0000 0000 0000  0000 0001 0000 0000

    // Special value from cancelled acquire methods so caller can throw IE
    private static final long INTERRUPTED = 1L;

    // Values for node status; order matters
    // WNode节点的status值
    private static final int WAITING   = -1;
    private static final int CANCELLED =  1;

    // Modes for nodes (int not boolean to allow arithmetic)
    // WNode节点的读写模式
    private static final int RMODE = 0;
    private static final int WMODE = 1;
    

    /** Head of CLH queue */
    private transient volatile WNode whead;
    /** Tail (last) of CLH queue */
    private transient volatile WNode wtail;

    // views
    transient ReadLockView readLockView;
    transient WriteLockView writeLockView;
    transient ReadWriteLockView readWriteLockView;

    /** Lock sequence/state */
    /** 锁队列状态， 当处于写模式时第8位为1，读模式时前7位为1-126（附加的readerOverflow用于当读者超过126时） */
    private transient volatile long state;

    /** extra reader count when state read count saturated */
    /** 将state超过 RFULL=126的值放到readerOverflow字段中 */
    private transient int readerOverflow;
    
    //内部节点
    static final class WNode {
        volatile WNode prev;
        volatile WNode next;
        volatile WNode cowait;    // list of linked readers （读模式使用该节点形成栈。用于链接等待获取读状态的节点。）
        volatile Thread thread;   // non-null while possibly parked
        volatile int status;      // 0, WAITING, or CANCELLED
        final int mode;           // RMODE or WMODE
        WNode(int m, WNode p) { mode = m; prev = p; }
    }
    
    final class ReadLockView implements Lock {}
    
    final class WriteLockView implements Lock {}
    
    final class ReadWriteLockView implements ReadWriteLock {}
}
```
由源码可知，StampedLock锁与其他的锁不一样。其他的锁，如：ReentrantLock，都是基于AQS实现的，StampedLock锁并没有实现AQS抽象类。StampedLock锁与AQS一样，内部也有一个`state`字段，用来表示锁状态，但是声明为`long`型，而非AQS中的`int`型。<font color='red'>在StampedLock锁中，当处于写模式时stats二进制的第8位为1，读模式时前7位为1-126（附加的readerOverflow用于当读者超过126时）。</font>

##### 构造函数

```java
public StampedLock() {
    state = ORIGIN;
}
```
在构造StampedLock对象时，会初始化`state`的值为:
```math
2^7
```
即二进制：
0000 0000 0000 0000  0000 0000 0000  0000 0000 0000 0000  0000 0001 0000 0000

##### 读锁获取

###### readLock源码

```java
//悲观读锁，非独占锁，为获得锁一直处于阻塞状态，直到获得锁为止
public long readLock() {
    long s = state, next;  // bypass acquireRead on common uncontended case
    // 队列为空   && 没有写锁同时读锁数小于126  && CAS修改状态成功      则状态加1并返回，否则自旋获取读锁
    return ((whead == wtail && (s & ABITS) < RFULL &&
             U.compareAndSwapLong(this, STATE, s, next = s + RUNIT)) ?
            next : acquireRead(false, 0L));
}
```
该方法实现了悲观锁的获取。在`排队等待队列为空` 且`没有写锁同时读锁数小于126` 且 `CAS修改状态成功`，则状态加1并返回，否则自旋获取读锁。

###### acquireRead源码

```java
private long acquireRead(boolean interruptible, long deadline) {
    WNode node = null, p;
    //自旋
    for (int spins = -1;;) {
        WNode h;

        /**
         *   if块功能：
         *   先判断同步队列是否为空，如果为空那么尝试获取读状态，同时如果此时写状态被占有的话还是会根据spins的值随机的自旋一定的时间如果还是没获取到则跳出自旋进入外层的循环。
         */
        if ((h = whead) == (p = wtail)) {   //判断队列为空
            for (long m, s, ns;;) {
                //将state超过 RFULL=126的值放到readerOverflow字段中。m < WBIT说明没有写状态被占有
                if ((m = (s = state) & ABITS) < RFULL ?
                    U.compareAndSwapLong(this, STATE, s, ns = s + RUNIT) :
                    (m < WBIT && (ns = tryIncReaderOverflow(s)) != 0L))
                    return ns;
                else if (m >= WBIT) { //state高8位大于0，那么说明当前锁已经被写锁独占，那么我们尝试自旋  + 随机的方式来探测状态
                    if (spins > 0) {
                        if (LockSupport.nextSecondarySeed() >= 0)
                            --spins;
                    }
                    else {
                        if (spins == 0) {
                            WNode nh = whead, np = wtail;
                            //一直获取锁失败，或者有线程入队列了退出内循环自旋，后续进入队列。
                            // 判断稳定性（有没有被修改）
                            if ((nh == h && np == p) || (h = nh) != (p = np))
                                break;
                        }
                        //自旋 SPINS 次
                        spins = SPINS;
                    }
                }
            }
        }


        //如果同步队列不为空，说明已经有别的线程在排队了（写线程），那么开始检查是否需要初始化，如果没有初始化则构造一个WMODE的节点作为头节点。由上个if语句可知，只有当前锁被写锁独占，才能跳出上个if语句中的for循环执行到此处。
        if (p == null) { // initialize queue //初始队列
            //由于此时是有写线程占有同步状态所以用一个WMODE的节点放入队列
            WNode hd = new WNode(WMODE, null);
            //CAS插入，如果失败的话下次循环再次尝试
            if (U.compareAndSwapObject(this, WHEAD, null, hd))
                wtail = hd;
        }


        /**
         *   if语句功能分析：
         *   此时构造当前线程的节点node尝试加入同步队列，加入的方式有两种，
         *      1：是如果队列的tail是WMODE或者队列的head==tail，那么直接加入队列的尾部，并跳出外层循环；
         *      2：是加入tail节点的cowait的链中。并继续执行。
         */
        //当前节点为空则构建当前节点，模式为RMODE，前驱节点为p即尾节点。（初始化代表当前读线程的节点）
        else if (node == null)
            node = new WNode(RMODE, p);

        //将当前线程node节点加入同步队列方式1，添加到队列尾部
            //当前队列为空即只有一个节点（whead=wtail）或者当前尾节点的模式不是RMODE，那么我们会尝试在尾节点后面添加该节点作为尾节点，然后跳出外层循环
        else if (h == p || p.mode != RMODE) {
            if (node.prev != p)
                node.prev = p;
            else if (U.compareAndSwapObject(this, WTAIL, p, node)) {
                p.next = node;
                break;   //入队列成功，退出自旋。执行第二个for循环
            }
        }

        //将当前线程node节点加入同步队列方式2，添加到队列尾部节点中的cowait队列中
        else if (!U.compareAndSwapObject(p, WCOWAIT,
                                         node.cowait = p.cowait, node))   //队列不为空并且是RMODE模式， 添加该节点到尾节点的cowait链（实际上构成一个读线程stack）中
            node.cowait = null; //失败处理


        //如果上面加入tail节点的cowait链中的CAS操作成功，则释放cowait链中的节点。
        else {
            //通过CAS方法将该节点node添加至尾节点的cowait链中，node成为cowait中的顶元素，cowait构成了一个LIFO队列。
            for (;;) {
                WNode pp, c; Thread w;

                //如果head不为空那么尝试去解放head的cowait链中的节点
                if ((h = whead) != null && (c = h.cowait) != null &&
                    U.compareAndSwapObject(h, WCOWAIT, c, c.cowait) &&
                    (w = c.thread) != null) // help release
                    U.unpark(w);


                //node所在的根节点p的前驱就是whead或者p已经是whead或者p的前驱为null
                if (h == (pp = p.prev) || h == p || pp == null) {  //因为是成功执行U.compareAndSwapObject()操作后，程序才能走到此处，因此可知，p依然为队列中的尾节点。
                    long m, s, ns;
                    do {
                        //根据state再次积极的尝试获取锁
                        if ((m = (s = state) & ABITS) < RFULL ?
                            U.compareAndSwapLong(this, STATE, s,
                                                 ns = s + RUNIT) :
                            (m < WBIT &&
                             (ns = tryIncReaderOverflow(s)) != 0L))
                            return ns;
                    } while (m < WBIT); //条件为读模式
                }

                //判断是否稳定
                if (whead == h && p.prev == pp) {
                    long time;
                    //如果tail的前驱是null或者head==tail或者tail已经被取消了(p.status > 0)
                    //直接将node置为null跳出循环，回到最开的for循环中去再次尝试获取同步状态
                    if (pp == null || h == p || p.status > 0) {
                        //这样做的原因是被其他线程闯入夺取了锁，或者p已经被取消
                        node = null; // throw away
                        break;
                    }

                    //判断超时
                    if (deadline == 0L)
                        time = 0L;
                    else if ((time = deadline - System.nanoTime()) <= 0L)    //如果超时则取消当前线程
                        return cancelWaiter(node, p, false);
                    Thread wt = Thread.currentThread();
                    U.putObject(wt, PARKBLOCKER, this);
                    node.thread = wt;
                    //tail的前驱不是head或者当前只有写线程获取到同步状态
                    if ((h != pp || (state & ABITS) == WBIT) &&
                        whead == h && p.prev == pp)
                        U.park(false, time);
                    node.thread = null;
                    U.putObject(wt, PARKBLOCKER, null);
                    //中断的话取消
                    if (interruptible && Thread.interrupted())
                        return cancelWaiter(node, p, true);
                }
            }
        }
    }

    /**
     * 在该for循环中，节点的自旋限制为先驱节点就是头节点，并且自旋同样不是无休止的，而是通过一个spins的值来控制，并且是相对随机的。
     */
    //如果tail的mode是WMODE写状态，那么node被加入到队列的tail之后进入这个循环
    for (int spins = -1;;) {
        WNode h, np, pp; int ps;
        //如果p(node的前驱节点)就是head，那么自旋方式尝试获取同步状态
        if ((h = whead) == p) {
            if (spins < 0)
                spins = HEAD_SPINS;
            else if (spins < MAX_HEAD_SPINS)
                spins <<= 1;
            for (int k = spins;;) { // spin at head
                long m, s, ns;

                //自旋方式尝试获取同步状态
                //获取成功的话将node设置为head并解放node的cowait链中的节点并返回stamp
                if ((m = (s = state) & ABITS) < RFULL ?
                    U.compareAndSwapLong(this, STATE, s, ns = s + RUNIT) :
                    (m < WBIT && (ns = tryIncReaderOverflow(s)) != 0L)) {
                    WNode c; Thread w;
                    whead = node;
                    node.prev = null;
                    while ((c = node.cowait) != null) {
                        if (U.compareAndSwapObject(node, WCOWAIT,
                                                   c, c.cowait) &&
                            (w = c.thread) != null)
                            U.unpark(w);
                    }
                    return ns;
                }
                //如果有写线程获取到了同步状态(因为可能有写线程闯入)那么随机的--k控制循环次数
                else if (m >= WBIT &&
                         LockSupport.nextSecondarySeed() >= 0 && --k <= 0)
                    break;
            }
        }

        //如果head不为null，解放head的cowait链中的节点
        else if (h != null) {
            WNode c; Thread w;
            while ((c = h.cowait) != null) {
                if (U.compareAndSwapObject(h, WCOWAIT, c, c.cowait) &&
                    (w = c.thread) != null)
                    U.unpark(w);
            }
        }

        //判断稳定性
        if (whead == h) {
            if ((np = node.prev) != p) {
                if (np != null)
                    (p = np).next = node;   // stale
            }

            //尝试设tail的状态位WAITING表示后面还有等待的节点
            else if ((ps = p.status) == 0)
                U.compareAndSwapInt(p, WSTATUS, 0, WAITING);
            else if (ps == CANCELLED) {   //如果tail已经取消了，跳过tail
                if ((pp = p.prev) != null) {
                    node.prev = pp;
                    pp.next = node;
                }
            }
            else {
                long time;
                if (deadline == 0L)
                    time = 0L;
                else if ((time = deadline - System.nanoTime()) <= 0L)
                    return cancelWaiter(node, node, false);
                Thread wt = Thread.currentThread();
                U.putObject(wt, PARKBLOCKER, this);
                node.thread = wt;
                if (p.status < 0 &&
                    (p != h || (state & ABITS) == WBIT) &&
                    whead == h && node.prev == p)
                    U.park(false, time);
                node.thread = null;
                U.putObject(wt, PARKBLOCKER, null);
                if (interruptible && Thread.interrupted())
                    return cancelWaiter(node, node, true);
            }
        }
    }
}
```
在调用readLock方法时，会首先尝试直接CAS改变state（`在whead==wtail和(s & ABITS) < RFULL的情况下`），成功的话直接返回stamp（next）。
竞争失败的情况下进入acquireRead的逻辑。acquireRead内部有两个代码比较多的for循环：
1. 第一个for循环
    
    功能：首先在队列为空且没有写锁的情况下，尝试循环获取读锁，直至有写锁时，把当前线程入队，并使当前线程睡眠。

2. 第二个for循环

    功能：在第一个for循环中，把当前线程入队时，有两种情况：（1）队列不为空且尾节点为RMODE模式，则把当前线程添加到尾节点的cowait链中（实际上构成一个读线程stack）；（2）情况1不满足的情况下，则把当前线程作为尾节点。在情况2发生时，则会跳出第一个for循环，进入第二个for循环。该循环的功能主要是在当前线程组成的node的前驱节点位head时，继续尝试获取锁；在head不为null时，释放cowait链中的节点；在可能的情况下，使当前线程睡眠。
    
下面详细描述acquireRead的内部逻辑：
+ acquireRead方法调用，然后进入第一个for循环。
+ 首先取得whead和wtail两个值，假如这两个值不等说明等待队列不为空，那么获取读锁没希望了，会进入等待队列。
+ 如果`whead == wtail` ，会进入内部的for循环，在读锁未超过RFULL=126时，尽力尝试通过前7bit上递增state来获取锁；如果超过了，并且在没有写锁的情况下(m < WBIT)，超出126的部分最终到了readerOverflow中，加入获取了锁就返回stamp。假如m >= WBIT，也就是说m（state前8位）值大于或等于128，那么说明当前锁已经被写者独占，那么我们尝试自旋+随机的方式来探测状态，并且在当前队列和进入循环前一样（说明还没有其他入队者）或者当前队列中已经有了入队者的情况下内层循环跳出，接着肯定会入队。
+ 首先根据尾节点为null的情况探测是否初始化队列，使用一个WMODE模式的节点初始化whead和wtail。（因为只有当前锁被写锁独占，才能跳出上个if语句中的for循环，所以会创建一个WMODE模式的节点初始化队列）
+ 然后假如当前节点为空则构建当前节点，模式为RMODE，前驱节点为p即尾节点。
+ 接着假如当前队列为空即只有一个节点（whead=wtail）或者当前尾节点的模式不是RMODE，那么我们会尝试在尾节点后面添加该节点作为尾节点，然后跳出外层循环；假如当前队列不为空并且当前尾节点模式就是RMODE，那么我们会尝试下一步：添加该节点到尾节点的cowait链（实际上构成一个stack）中。

当前尾节点模式为RMODE模式时逻辑：
+ 通过CAS方法将该节点node添加至尾节点的cowait链中，node成为cowait中的顶元素，cowait构成了一个LIFO队列。
+ 成功后进入另一个循环，首先尝试unpark头元素（whead）的cowait中的第一个元素，这只是一种辅助作用，因为头元素whead所伴随的那个线程（假如存在）必定是已经获得锁了，假如是读锁那么也必定会释放cowait链。
+ 假如当前节点node所在的根节点p的前驱就是whead或者p已经是whead或者p的前驱为null，那么我们会根据state再次积极的尝试获取锁（当m < WBIT）。
+ 否则我们探测当前队列是否稳定：`whead == h && p.prev == pp`，在稳定的情况下，假如发现p成为过head或者p已经被取消（status>0），我们尝试node=null，并且跳出当前循环，回到一开的循环里面去尝试获取锁（这样做的原因是被其他线程闯入夺取了锁，或者p已经被取消）。
+ 接着我们判断是否为限时版本，以及限时版本所需时间。
+ 然后park当前线程以及可能出现的中断情况下取消当前节点的cancelWaiter操作。

我们再来分析跳出循环的另一种情况：队列中无节点或者尾节点模式为WMODE：这样我们的节点必须直接拼接到尾节点后面。下面为第二个for循环逻辑：

+ p作为当前节点的前驱节点，假如正好是whead的话，那么会尝试自旋+随机的方式在积极得探测state，从而能够取得锁。并且在获得锁，重置whead和node.prev=null之后释放当前cowait链中的节点。最后返回stamp。
+ 否则只需h不为null时尝试释放当前头节点的cowait链，作为一种协作的积极行动。
+ 然后在whead==h即队列稳定时，首先会CAS操作当前节点前驱的status，从0变为WAITING从而指示后面有等待的节点。假如发现p的状态已经为取消了，则重新选择node的前驱。
+ 前面的这些都处理完成之后，使用类似的park以及cancelWaiter操作。区别在于这里的p.status<0必须保证（因为等待状态WAITING是-1）。

###### tryIncReaderOverflow源码

```java
private long tryIncReaderOverflow(long s) {
    // assert (s & ABITS) >= RFULL;
    if ((s & ABITS) == RFULL) {
        //将state超过 RFULL=126的值放到readerOverflow字段中，state保持不变，但锁状态+1
        if (U.compareAndSwapLong(this, STATE, s, s | RBITS)) {
            ++readerOverflow;
            state = s;
            return s;
        }
    }
    else if ((LockSupport.nextSecondarySeed() &     //LockSupport.nextSecondarySeed() 生成随机数
              OVERFLOW_YIELD_RATE) == 0)
        Thread.yield(); //线程放弃CPU资源
    return 0L;
}
```
将state超过 RFULL=126的值放到readerOverflow字段中，state的前七位记录到126之后就会稳定在这个值，偶尔会到127，但是超出126的部分最终到了readerOverflow，加入获取了锁就返回stamp。

###### cancelWaiter源码

```java
/**
 * @param node if nonnull, the waiter
 * @param group either node or the group node is cowaiting with
 * @param interrupted if already interrupted
 * @return INTERRUPTED if interrupted or Thread.interrupted, else zero
 */
private long cancelWaiter(WNode node, WNode group, boolean interrupted) {
    if (node != null && group != null) {
        Thread w;
        node.status = CANCELLED;
        // unsplice cancelled nodes from group
        // 移除栈中取消状态的节点
        for (WNode p = group, q; (q = p.cowait) != null;) {
            if (q.status == CANCELLED) {
                U.compareAndSwapObject(p, WCOWAIT, q, q.cowait);
                p = group; // restart
            }
            else
                p = q;
        }
        if (group == node) {
            //唤醒栈中所有非取消状态节点线程
            for (WNode r = group.cowait; r != null; r = r.cowait) {
                if ((w = r.thread) != null)
                    U.unpark(w);       // wake up uncancelled co-waiters
            }
            for (WNode pred = node.prev; pred != null; ) { // unsplice
                // 寻找到node后面的第一个非CANCELLED节点，直接拼接到pred上
                WNode succ, pp;        // find valid successor
                while ((succ = node.next) == null ||
                       succ.status == CANCELLED) {
                    WNode q = null;    // find successor the slow way
                    for (WNode t = wtail; t != null && t != node; t = t.prev)
                        if (t.status != CANCELLED)
                            q = t;     // don't link if succ cancelled
                    if (succ == q ||   // ensure accurate successor
                        U.compareAndSwapObject(node, WNEXT,
                                               succ, succ = q)) {
                        if (succ == null && node == wtail)
                            U.compareAndSwapObject(this, WTAIL, node, pred);
                        break;
                    }
                }
                // 将当前节点的前置节点链接到当前节点的后继节点
                if (pred.next == node) // unsplice pred link
                    U.compareAndSwapObject(pred, WNEXT, node, succ);
                if (succ != null && (w = succ.thread) != null) {
                    succ.thread = null;
                    U.unpark(w);       // wake up succ to observe new pred
                }
                //检查前驱节点状态，假如为CANCELLED则也需要重置前驱节点。
                if (pred.status != CANCELLED || (pp = pred.prev) == null)
                    break;
                node.prev = pp;        // repeat if new pred wrong/cancelled
                U.compareAndSwapObject(pp, WNEXT, pred, succ);
                pred = pp;
            }
        }
    }
    WNode h; // Possibly release first waiter
    while ((h = whead) != null) {
        long s; WNode q; // similar to release() but check eligibility
        if ((q = h.next) == null || q.status == CANCELLED) {
            for (WNode t = wtail; t != null && t != h; t = t.prev)
                if (t.status <= 0)
                    q = t;
        }
        if (h == whead) {
            if (q != null && h.status == 0 &&
                ((s = state) & ABITS) != WBIT && // waiter is eligible
                (s == 0L || q.mode == RMODE))
                release(h);
            break;
        }
    }
    return (interrupted || Thread.interrupted()) ? INTERRUPTED : 0L;
}
```
该方法的功能就是在等待队列中取消当前节点。内部逻辑为：
+ 首先设置node的状态为CANCELLED，可以向其他线程传递这个节点是删除了的信息。
+ 然后再聚合节点gruop上清理所有状态为CANCELLED的节点（即删除节点）
+ 接下来假如当期node节点本身就是聚合节点，那么首先唤醒cowait链中的所有节点（读者），寻找到node后面的第一个非CANCELLED节点，直接拼接到pred上（从而删除当前节点），然后再检查前驱节点状态，假如为CANCELLED则也需要重置前驱节点。
+ 最后，在队列中不为空，并且头结点的状态为0即队列中的节点还未设置WAITING信号&当前没有持有写入锁模式&（当前没有锁或者只有乐观锁 | 队列中第一个等待者为读模式），那么就从队列头唤醒一次。

###### release源码

```java
//释放当前节点， 唤醒继任者节点线程
private void release(WNode h) {
    if (h != null) {
        WNode q; Thread w;
        U.compareAndSwapInt(h, WSTATUS, WAITING, 0);    //将状态由WAITING改为0
        if ((q = h.next) == null || q.status == CANCELLED) {    //找到下个status为WAITING的节点，并唤醒线程。如果当期节点的后继节点为null，或者状态为CANCELLED时，从wtail往前遍历，找到status为WAITING的节点
            for (WNode t = wtail; t != null && t != h; t = t.prev)
                if (t.status <= 0)
                    q = t;
        }
        if (q != null && (w = q.thread) != null)
            U.unpark(w);    //唤醒线程
    }
}
```

##### 读锁释放

###### unlockRead源码

```java
public void unlockRead(long stamp) {
    long s, m; WNode h;
    for (;;) {
        if (((s = state) & SBITS) != (stamp & SBITS) ||
            (stamp & ABITS) == 0L || (m = s & ABITS) == 0L || m == WBIT)    //不匹配抛出异常。state & SBITS 之后将抹去前7位以外的部分只剩下读状态
            throw new IllegalMonitorStateException();
        if (m < RFULL) {    //小于最大记录数值
            if (U.compareAndSwapLong(this, STATE, s, s - RUNIT)) {
                if (m == RUNIT && (h = whead) != null && h.status != 0)
                    release(h);
                break;
            }
        }
        else if (tryDecReaderOverflow(s) != 0L) //否则readerOverflow减一
            break;
    }
}
```
在释放锁时，会传递一个stamp值，进行锁验证，如果验证通过，则直接修改`state`中代表读状态的值。在读状态数小于最大记录数时，直接修改`state`的值，并唤醒头结点中的线程；在大于时，修改`readerOverflow`值。

###### tryDecReaderOverflow源码

```java
private long tryDecReaderOverflow(long s) {
    // assert (s & ABITS) >= RFULL;
    if ((s & ABITS) == RFULL) {
        if (U.compareAndSwapLong(this, STATE, s, s | RBITS)) {
            int r; long next;
            if ((r = readerOverflow) > 0) {
                readerOverflow = r - 1;
                next = s;
            }
            else
                next = s - RUNIT;
             state = next;
             return next;
        }
    }
    else if ((LockSupport.nextSecondarySeed() &
              OVERFLOW_YIELD_RATE) == 0)
        Thread.yield();
    return 0L;
}
```

##### 写锁获取

```java
//获取写锁，获取失败会一直阻塞，直到获得锁成功
public long writeLock() {
    long s, next;  // bypass acquireWrite in fully unlocked case only
    return ((((s = state) & ABITS) == 0L &&         //完全没有任何锁（没有读锁和写锁）的时候可以通过，即判断state是否为初始态
             U.compareAndSwapLong(this, STATE, s, next = s + WBIT)) ?   ////第8位置为1,next = s + WBIT = 1 0000 0000 + 1000 0000 = 1 1000 0000
            next : acquireWrite(false, 0L));
}
```
在获取写锁时，如果在完全没有锁的情况下（既没有读锁，也没有写锁），会直接尝试CAS获取写锁。获取成功，直接返回；获取失败，则会调用acquireWrite方法。

###### acquireWrite源码

```java
private long acquireWrite(boolean interruptible, long deadline) {
    WNode node = null, p;

    /**
     * 写状态的获取基本和读一样，区别在于写状态获取的时候根本就没有去判断同步队列里面是否有节点，
     * 而且尝试获取写状态的条件是(s = state) & ABITS) == 0L，也就是说要没有任何的其他锁占用的情况下才会去CAS尝试获取写状态。
     * 同时如果获取失败加入同步队列的时候只会直接加入同步队列的尾部，不会加入cowait链。这也说明了StampedLock的写是无条件去获取锁。
     */
    for (int spins = -1;;) { // spin while enqueuing（轮询入队）
        long m, s, ns;
        if ((m = (s = state) & ABITS) == 0L) { //无锁
            if (U.compareAndSwapLong(this, STATE, s, ns = s + WBIT))
                return ns;
        }
        else if (spins < 0) //持有写锁，并且队列为空
            spins = (m == WBIT && wtail == whead) ? SPINS : 0;
        else if (spins > 0) {
            if (LockSupport.nextSecondarySeed() >= 0) //恒成立
                --spins;
        }
        else if ((p = wtail) == null) { // initialize queue
            WNode hd = new WNode(WMODE, null);  //初始化队列，写锁入队列
            if (U.compareAndSwapObject(this, WHEAD, null, hd))
                wtail = hd;
        }
        else if (node == null) //p不为空，即队列不为空，写锁入队列
            node = new WNode(WMODE, p);
        else if (node.prev != p)
            node.prev = p;
        else if (U.compareAndSwapObject(this, WTAIL, p, node)) {
            p.next = node;
            break; //入队列成功退出循环
        }
    }


    for (int spins = -1;;) {
        WNode h, np, pp; int ps;
        if ((h = whead) == p) { //前驱节点为头节点
            if (spins < 0)
                spins = HEAD_SPINS;
            else if (spins < MAX_HEAD_SPINS)
                spins <<= 1;
            for (int k = spins;;) { // spin at head
                long s, ns;
                if (((s = state) & ABITS) == 0L) {  //无锁
                    if (U.compareAndSwapLong(this, STATE, s,
                                             ns = s + WBIT)) {
                        whead = node;  //当前节点设置为头结点
                        node.prev = null;
                        return ns;
                    }
                }
                else if (LockSupport.nextSecondarySeed() >= 0 &&
                         --k <= 0)
                    break;
            }
        }
        else if (h != null) { // help release stale waiters
            WNode c; Thread w;
            while ((c = h.cowait) != null) {    //头结点为读锁将栈中所有读锁线程唤醒
                if (U.compareAndSwapObject(h, WCOWAIT, c, c.cowait) &&
                    (w = c.thread) != null)
                    U.unpark(w);
            }
        }
        if (whead == h) {
            if ((np = node.prev) != p) {
                if (np != null)
                    (p = np).next = node;   // stale
            }
            else if ((ps = p.status) == 0)
                U.compareAndSwapInt(p, WSTATUS, 0, WAITING);
            else if (ps == CANCELLED) {
                if ((pp = p.prev) != null) {
                    node.prev = pp;
                    pp.next = node;
                }
            }
            else {
                long time; // 0 argument to park means no timeout
                if (deadline == 0L)
                    time = 0L;
                else if ((time = deadline - System.nanoTime()) <= 0L)
                    return cancelWaiter(node, node, false);
                Thread wt = Thread.currentThread();
                U.putObject(wt, PARKBLOCKER, this);
                node.thread = wt;
                if (p.status < 0 && (p != h || (state & ABITS) != 0L) &&
                    whead == h && node.prev == p)
                    U.park(false, time);  // emulate LockSupport.park
                node.thread = null;
                U.putObject(wt, PARKBLOCKER, null);
                if (interruptible && Thread.interrupted())
                    return cancelWaiter(node, node, true);
            }
        }
    }
}
```
acquireWrite方法逻辑实现中，也有两个for循环：
1. 第一个for循环

    功能：在无任何锁的情况下，会直接通过CAS操作获取写锁；否则，在持有写锁，并且队列为空的情况下，自旋一段时间获取写锁后，如果还未成功获取到写锁，则将当前线程入队列（在入队列之前，若队列为空，先初始化队列），入队成功后，跳出第一个循环。

2. 第二个for循环 （与acquireRead方法的第二个for循环逻辑差不多）

    + 在当前线程入队成功后，如果其前驱结点为头结点，且无任何锁的情况下，则根据头结点自旋次数，循环获取写锁；自旋次数完后，若还未获取锁，则跳出内部循环。
    + 如头结点不为空，则释放头结点中cowait链上的节点
    + 然后若在whead==h即队列稳定时，首先会CAS操作当前节点前驱的status，从0变为WAITING从而指示后面有等待的节点。假如发现p的状态已经为取消了，则重新选择node的前驱。
   + 前面的这些都处理完成之后，使用类似的park以及cancelWaiter操作。区别在于这里的p.status<0必须保证（因为等待状态WAITING是-1）。

###### 释放写锁

```java
public void unlockWrite(long stamp) {
    WNode h;
    //因为写锁是独占式的所以可以简单判断state != stamp
    if (state != stamp || (stamp & WBIT) == 0L) //state不匹配stamp  或者 没有写锁
        throw new IllegalMonitorStateException();
    //state += WBIT， 第8位置为0
    /*
    *这里的(stamp += WBIT) == 0L ? ORIGIN : stamp解释：
    *假设stamp为：ORIGIN + WBIT(第一次获取了写锁的状态)
    *0000 0000 0000 0000  0000 0000 0000  0000 0000 0000 0000  0000 0001 1000 0000
    *那么加上一个WBIT之后位
    *0000 0000 0000 0000  0000 0000 0000  0000 0000 0000 0000  0000 0010 0000 0000
    *此时第八位已经为0，表示已经释放了写锁
    *但是随着这样累加上去可能最后会溢出结果64位全部为0，所以如果这种情况就置为ORIGIN
    */
    state = (stamp += WBIT) == 0L ? ORIGIN : stamp;
    if ((h = whead) != null && h.status != 0)
        release(h); //唤醒继承者节点线程
}
```
在释放写锁时，如果stamp校验失败，则抛出异常；否则，释放锁，其实就是将`state的第8位置为0`，所以使用`state`加上
    
```math
2^9
```
即：

0000 0000 0000 0000  0000 0000 0000  0000 0000 0000 0000  0000 0010 0000 0000

即可将state的第8位置为0，达到释放锁的效果。因为`state`为long型，会发生溢出，64位全部为0，所以如果这种情况就置为ORIGIN。

##### 乐观读锁获取

###### tryOptimisticRead源码
```java
public long tryOptimisticRead() {
    long s; //有写锁返回0.   否则返回256
    return (((s = state) & WBIT) == 0L) ? (s & SBITS) : 0L;
}
```
在获取乐观锁的时候，如果没有写锁，则返回固定值`256`，否则返回0。`返回0表示获取锁失败`。

###### validate源码

```java
public boolean validate(long stamp) {
    U.loadFence();  //强制读取操作和验证操作在一些情况下的内存排序问题
    return (stamp & SBITS) == (state & SBITS);  //当持有写锁后再释放写锁，该校验也不成立，返回false
}
```
在成功获取乐观锁并读取所需数据后，需要调用validate方法校验在读取期间是否发生了其他写锁（因为与SBITS进行&操作时，只会检查8-64位，所以`(stamp & SBITS) == (state & SBITS)`时，说明期间没有发生新的写锁）。

###### unlock源码

```java
public void unlock(long stamp) {
    long a = stamp & ABITS, m, s; WNode h;
    while (((s = state) & SBITS) == (stamp & SBITS)) {   //有锁，state匹配stamp
        if ((m = s & ABITS) == 0L)  //初始状态
            break;
        else if (m == WBIT) {   //写锁
            if (a != m)
                break;
            //s += WBIT， 第8位置为0
            state = (s += WBIT) == 0L ? ORIGIN : s;
            if ((h = whead) != null && h.status != 0)
                release(h);
            return;
        }
        else if (a == 0L || a >= WBIT)  //表示没有锁
            break;
        else if (m < RFULL) {   //读锁
            if (U.compareAndSwapLong(this, STATE, s, s - RUNIT)) {
                if (m == RUNIT && (h = whead) != null && h.status != 0)
                    release(h);
                return;
            }
        }
        //如果是读锁并且overflow
        else if (tryDecReaderOverflow(s) != 0L)
            return;
    }
    throw new IllegalMonitorStateException();
}
```
该方法释放的锁，既可以是读锁，也可以是写锁。首先，在stamp校验不通过时，直接抛异常；通过时
+ 若`state`为初始态，即没有锁时，抛异常；
+ 为写锁时，当`a != m`时，抛异常；否则释放写锁，并唤醒后续节点。
+ 若为读锁，并`m < RFULL`，释放读锁并唤醒后续节点；否则减少overflow


###### tryConvertToWriteLock源码

```java
/**
 state匹配stamp时, 执行下列操作之一：
        1、stamp 已经持有写锁，直接返回.
        2、读模式，但是没有更多的读取者，并返回一个写锁stamp.
        3、有一个乐观读锁，只在即时可用的前提下返回一个写锁stamp
        4、其他情况都返回0
*/
public long tryConvertToWriteLock(long stamp) {
    //ABITS 1111 1111
    //SBITS 1000 0000
    //WBIT  1000 0000
    //RUNIT 0000 0001
    long a = stamp & ABITS, m, s, next;
    while (((s = state) & SBITS) == (stamp & SBITS)) {  //state匹配stamp
        if ((m = s & ABITS) == 0L) {    //没有锁
            if (a != 0L)
                break;
            if (U.compareAndSwapLong(this, STATE, s, next = s + WBIT))  //CAS修改状态为持有写锁，并返回
                return next;
        }
        else if (m == WBIT) {   //持有写锁
            if (a != m) //其他线程持有写锁
                break;
            return stamp;   //当前线程已经持有写锁
        }
        else if (m == RUNIT && a != 0L) {   ////有一个读锁
            //释放读锁，并尝试持有写锁
            if (U.compareAndSwapLong(this, STATE, s,
                                     next = s - RUNIT + WBIT))
                return next;
        }
        else
            break;
    }
    return 0L;
}
```
该方法将读锁转为写锁。

###### tryConvertToReadLock源码

```java
/**
 state匹配stamp时, 执行下列操作之一.
     1、stamp 表示持有写锁，释放写锁，并持有读锁
     2 stamp 表示持有读锁 ，返回该读锁
     3 有一个乐观读锁，只在即时可用的前提下返回一个读锁stamp
     4、其他情况都返回0，表示失败
*/
public long tryConvertToReadLock(long stamp) {
    long a = stamp & ABITS, m, s, next; WNode h;
    while (((s = state) & SBITS) == (stamp & SBITS)) {
        if ((m = s & ABITS) == 0L) {    //没有锁
            if (a != 0L)
                break;
            else if (m < RFULL) {
                if (U.compareAndSwapLong(this, STATE, s, next = s + RUNIT))
                    return next;
            }
            else if ((next = tryIncReaderOverflow(s)) != 0L)
                return next;
        }
        else if (m == WBIT) {   //写锁
            if (a != m) //非当前线程持有写锁
                break;
            state = next = s + (WBIT + RUNIT);  //释放写锁持有读锁
            if ((h = whead) != null && h.status != 0)
                release(h);
            return next;
        }
        else if (a != 0L && a < WBIT)   //持有读锁
            return stamp;
        else
            break;
    }
    return 0L;
}
```
该方法将写锁转为读锁。
+ 在没有任何锁的时候，如果`m < RFULL`，则直接CAS获取读锁，否则增加`readerOverflow`。
+ 在持有写锁时，则释放写锁持有读锁（`state = s + (WBIT + RUNIT)`）
+ 在持有读锁时，直接返回`stamp`。




#### 参考文档
[深入理解StampedLock及其实现原理](http://blog.csdn.net/luoyuyou/article/details/30259877)

[JDK1.8 StampedLock源码解析](http://blog.csdn.net/huzhiqiangcsdn/article/details/76694836)

[StampedLock实现浅析](http://blog.5ibc.net/p/121849.html)