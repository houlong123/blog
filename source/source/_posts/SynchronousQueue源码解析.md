---
title: SynchronousQueue源码解析
date: 2017-12-12 15:09:27
tags: java并发集合
---

#### 前言
SynchronousQueue是BlockingQueue阻塞队列的一种实现。但是作为阻塞队列的一种，SynchronousQueue与其他BlockingQueue有着不同特性：
1. SynchronousQueue没有容量(`A synchronous queue does not have any internal capacity, not even a capacity of one.`)。所以其内部方法：`isEmpty`，`size`，`clear`，`contains`，`remove`都是默认实现。
2. SynchronousQueue分为公平（队列实现）和非公平（栈实现）策略。默认情况下采用非公平性访问策略。


#### 数据结构

<!-- more -->

```java
public class SynchronousQueue<E> extends AbstractQueue<E>
    implements BlockingQueue<E>, java.io.Serializable {
    
    // 可用的处理器
    static final int NCPUS = Runtime.getRuntime().availableProcessors();

    // 最大空旋时间
    static final int maxTimedSpins = (NCPUS < 2) ? 0 : 32;
    
    // 无限时的等待的最大空旋时间
    static final int maxUntimedSpins = maxTimedSpins * 16;
    
     // 超时空旋等待阈值
    static final long spinForTimeoutThreshold = 1000L;
    
    private transient volatile Transferer<E> transferer;
    
    //非公平策略实现类
    static final class TransferStack<E> extends Transferer<E> {
    
        // 表示消费数据的消费者
        static final int REQUEST    = 0;
        // 表示生产数据的生产者
        static final int DATA       = 1;
        //表示该操作节点处于真正匹配状态
        static final int FULFILLING = 2;
        //内部节点类
        static final class SNode {}
    }
    
    //公平策略实现类
    static final class TransferQueue<E> extends Transferer<E> {
        
        //队列头
        transient volatile QNode head;
        //队列尾
        transient volatile QNode tail;
        //当要删除的节点为队列中最后一个元素时，会引用到这个节点
        transient volatile QNode cleanMe;
        
        //内部节点类
        static final class QNode {}
    }
    
    //内部抽象类，其公平/非公平策略都是基于该类
   abstract static class Transferer<E> {
        /**
         * Performs a put or take.
         *
         * @param e if non-null, the item to be handed to a consumer;
         *          if null, requests that transfer return an item
         *          offered by producer.
         * @param timed if this operation should timeout
         * @param nanos the timeout, in nanoseconds
         * @return if non-null, the item provided or received; if null,
         *         the operation failed due to timeout or interrupt --
         *         the caller can distinguish which of these occurred
         *         by checking Thread.interrupted.
         */
        // 转移数据，put或者take操作
        abstract E transfer(E e, boolean timed, long nanos);
    } 
}
```
由源码可知，SynchronousQueue阻塞队列的内部，有一个`Transferer接口`，该接口中的API由队列（公平策略）/栈（非公平策略）共同使用，实现不同功能。SynchronousQueue队列的所有功能操作都是通过`Transferer`对象来实现的。

#### 构造函数

```java
//默认构造函数
public SynchronousQueue() {
    this(false);
}

public SynchronousQueue(boolean fair) {
    // 通过 fair 值来决定内部用 使用 queue 还是 stack 存储线程节点。队列是公平的
    transferer = fair ? new TransferQueue<E>() : new TransferStack<E>();
}
```
从构造函数中可知，`SynchronousQueue`阻塞队列的默认实现方式是`TransferStack(非公平策略)`，我们可以通过传递参数来决定使用何种阻塞队列实现策略。


#### put 源码

```java
// 将指定元素添加到此队列，如有必要则等待另一个线程接收它
public void put(E e) throws InterruptedException {
    //元素e为空，抛异常
    if (e == null) throw new NullPointerException();

    if (transferer.transfer(e, false, 0) == null) { // 进行转移操作
        Thread.interrupted();   // 中断当前线程
        throw new InterruptedException();
    }
}
```
由源码可知，`SynchronousQueue`阻塞队列不允许存储为`null`的元素。然后操作转给`Transfer`对象去处理，具体的处理细节，详见下面的分析。


#### offer源码

```java
public boolean offer(E e) {
    if (e == null) throw new NullPointerException();
    return transferer.transfer(e, true, 0) != null;
}

//超时offer源码
public boolean offer(E e, long timeout, TimeUnit unit)
    throws InterruptedException {
    //元素e为空，抛异常
    if (e == null) throw new NullPointerException();
    //进行转移操作
    if (transferer.transfer(e, true, unit.toNanos(timeout)) != null)
        return true;
    if (!Thread.interrupted())  // 当前线程没有被中断
        return false;
    throw new InterruptedException();
}
```
offer的源码实现与put的源码实现很类似。不在分析。


#### take源码

```java
// 获取并移除此队列的头，如有必要则等待另一个线程插入它
public E take() throws InterruptedException {
    E e = transferer.transfer(null, false, 0);  // 进行转移操作
    if (e != null)
        return e;
    Thread.interrupted();
    throw new InterruptedException();
}
```
在使用 `take` 从阻塞队列中获取元素时，也是通过`Transfer`对象去处理。在获取的元素不为null时，直接返回；否则，中断并抛出异常。

#### poll源码

```java
// 如果另一个线程当前正要使用某个元素，则获取并移除此队列的头
public E poll() {
    return transferer.transfer(null, true, 0);
}

//超时poll源码
// 获取并移除此队列的头，如有必要则等待指定的时间，以便另一个线程插入它
public E poll(long timeout, TimeUnit unit) throws InterruptedException {
    E e = transferer.transfer(null, true, unit.toNanos(timeout));
    if (e != null || !Thread.interrupted())  // 元素不为null或者当前线程没有被中断
        return e;
    throw new InterruptedException();
}
```
同理，具体实现也是通过`Transfer`对象。

由上面的源码可知，阻塞队列SynchronousQueue的有关操作，都是通过内部接口`Transfer`的实现类来间接完成的。`Transfer接口`的实现类有两个，即两种模式：

1. 公平模式

    所谓公平就是遵循 `先来先服务` 的原则，因此其内部使用了一个FIFO队列 来实现其功能。

2. 非公平模式

    SynchronousQueue 中的非公平模式是默认的模式，其内部使用栈来实现其功能，也就是 `后来的先服务`。




#### 公平模式 TransferQueue

##### 数据结构

```java
/** Dual Queue */
/**
 *  这是一个典型的 queue , 它有如下的特点
 *  1. 整个队列有 head, tail 两个节点
 *  2. 队列初始化时会有个 dummy 节点
 *  3. 这个队列的头节点是个 dummy 节点/ 或 哨兵节点, 所以操作的总是队列中的第二个节点(AQS的设计中也是这也)
 */
static final class TransferQueue<E> extends Transferer<E> {

    //队列头
    transient volatile QNode head;
    //队列尾
    transient volatile QNode tail;
    //当要删除的节点为队列中最后一个元素时，会引用到这个节点
    transient volatile QNode cleanMe;

    //单元节点类
    static final class QNode {
        volatile QNode next;          // next node in queue
        volatile Object item;         // CAS'ed to or from null
        volatile Thread waiter;       // to control park/unpark
        final boolean isData;
    }
}
```
在队列中，内部有个`QNode`类，代表的是队列中的一个存储节点。在该类的内部没有使用锁，而是通过cas算法来完成原子操作。

##### 构造函数

```java
TransferQueue() {
    /**
     * 构造一个 dummy node, 而整个 queue 中永远会存在这样一个 dummy node
     * dummy node 的存在使得 代码中不存在复杂的 if 条件判断
     */
    QNode h = new QNode(null, false); // initialize to dummy node.
    head = h;
    tail = h;
}
```
TransferQueue的构造函数，构造了一个哨兵节点，然后头尾节点都指向这个哨兵节点。

##### transfer源码

```java
/**
 * Puts or takes an item.
 */
@SuppressWarnings("unchecked")
E transfer(E e, boolean timed, long nanos) {
    /* Basic algorithm is to loop trying to take either of
     * two actions:
     *
     * 1. If queue apparently empty or holding same-mode nodes,
     *    try to add node to queue of waiters, wait to be
     *    fulfilled (or cancelled) and return matching item.
     *
     * 2. If queue apparently contains waiting items, and this
     *    call is of complementary mode, try to fulfill by CAS'ing
     *    item field of waiting node and dequeuing it, and then
     *    returning matching item.
     *
     * In each case, along the way, check for and try to help
     * advance head and tail on behalf of other stalled/slow
     * threads.
     *
     * The loop starts off with a null check guarding against
     * seeing uninitialized head or tail values. This never
     * happens in current SynchronousQueue, but could if
     * callers held non-volatile/final ref to the
     * transferer. The check is here anyway because it places
     * null checks at top of loop, which is usually faster
     * than having them implicitly interspersed.
     */

    QNode s = null; // constructed/reused as needed
    // 确定此次转移的类型（put or take）
    boolean isData = (e != null);

    for (;;) {
        QNode t = tail; // 获取尾结点
        QNode h = head; //获取头结点

        //初始化的队头和队尾都是指向的一个"空" 节点，不会是null
        if (t == null || h == null)       // 看到未初始化的头尾结点  // saw uninitialized value
            continue;                       // spin

        if (h == t || t.isData == isData) { //入队操作。 队列为空 或者 尾结点的模式与当前结点模式相同,即上次和本次是同样的操作：入队或者出队。每一个入队和出队是互相匹配的  // empty or same-mode
            // 获取尾结点的next域
            QNode tn = t.next;

            // 存在并发，队列被修改了，从头开始
            if (t != tail)                  // t不为尾结点，不一致，有并发，重试  // inconsistent read
                continue;

            //队列被修改了，tail不是队尾，则辅助推进tail
            if (tn != null) {           //tn不为null，有其他线程添加了tail的next结点，帮助推进 tail    // lagging tail
                // 设置新的尾结点为tn
                advanceTail(t, tn);
                continue;
            }

            //如果进行了超时等待操作，发生超时则返回NULL
            if (timed && nanos <= 0)    // 设置了timed并且等待时间小于等于0，表示不能等待，需要立即操作    // can't wait
                return null;

            if (s == null)
                s = new QNode(e, isData);   // 新生一个结点并赋值给s

            // 将新节点 cas 设置成tail的后继 ,失败则重来
            if (!t.casNext(null, s))   // 将 新建的节点加入到 队列中  // failed to link in
                continue;

            // 添加了一个节点，推进队尾指针，将队尾指向新加入的节点
            advanceTail(t, s);              // swing tail and wait

            // 空旋或者阻塞直到有匹配操作，即s结点被匹配
            Object x = awaitFulfill(s, e, timed, nanos);    // 调用awaitFulfill, 若节点是 head.next, 则进行一些自旋, 若不是的话, 直接 block, 知道有其他线程 与之匹配, 或它自己进行线程的中断

            //如果操作被取消
            if (x == s) {          // x与s相等，表示已经取消         // wait was cancelled
                clean(t, s);    // 清除
                return null;
            }

            //匹配的操作到来，s操作完成，离开队列，如果没离开，使其离开
            if (!s.isOffList()) {           // not already unlinked
                //推进head ,cas将head 由t(是t不是h)，设置为s
                advanceHead(t, s);          // unlink if head
                // x不为null
                if (x != null)              // and forget fields
                    // 设置s结点的item
                    s.item = s;
                // 设置s结点的waiter域为null
                s.waiter = null;
            }
            return (x != null) ? (E)x : e;

        } else {  // 模式互补。不同的模式那么是匹配的（put 匹配take ，take匹配put），出队操作             // complementary-mode

            // 获取头结点的next域（匹配的结点）。 这里是从head.next开始，因为TransferQueue总是会存在一个dummy node节点
            QNode m = h.next;               // node to fulfill

            //队列发生变化，重来
            if (t != tail || m == null || h != head)    // t不为尾结点或者m为null或者h不为头结点（不一致）
                continue;                   // inconsistent read

            Object x = m.item;  // 获取m结点的元素域

            // 如果isData == (x != null) 为true 那么表示是相同的操作。
            //x==m 则是被取消了的操作
            //m.casItem(x, e) 将m的item设置为本次操作的数据域
            if (isData == (x != null) ||    // m结点被匹配 // m already fulfilled
                x == m ||                   // m结点被取消 // m cancelled
                !m.casItem(x, e)) {         // CAS操作失败 // lost CAS
                advanceHead(h, m);          // 队列头结点出队列，并重试 // dequeue and retry
                continue;
            }

            // 匹配成功，设置新的头结点
            advanceHead(h, m);              // successfully fulfilled

            // 唤醒匹配操作的线程
            LockSupport.unpark(m.waiter);
            return (x != null) ? (E)x : e;
        }
    }
}
```
主要逻辑：
1. 通过判断元素e，来确认当前操作类型。（在元素e为null时，则表示消费（take操作）；在元素e不为null时，则表示生产（put操作））
2. 进入无限循环
3. 获取头尾节点，若没有初始化，则 `continue` 继续。
4. 如果队列为空或者尾结点的模式与当前结点模式相同，则进行入队操作。执行下面新步骤，否则执行第5步骤。
    1. 获取尾节点的next域 `tn`
    2. 如果队列存在并发，则 `continue` 继续。
    3. 如果 `tn 不为null`，说明有并发，则辅助推进tail节点，然后 `continue` 继续。
    4. 如果超时，则直接返回null
    5. 将元素e组装成节点s
    6. 将节点s设置为队列尾节点的next节点。如果成功，执行第7步骤，否则， `continue` 继续。
    7. 将节点s设置为队列尾节点
    8. 调用`awaitFulfill`方法，空旋或者阻塞直到有匹配操作。
    9. 如果操作被取消，则清除节点s，并返回null
    10. 判断节点s是否脱离队列，脱离，返回元素，方法结束；否则辅助节点s脱离队列。
5. 执行至此，说明当前操作与上次操作匹配（put 匹配take ，take匹配put），进行出队操作。
    1. 获取头节点的next域m节点
    2. 如果有并发情景，则`continue` 继续。
    3. 获取m节点的item域
    4. 如果 m节点已被匹配或者m节点被取消或者CAS操作m节点的item域失败，在重新设置队列的头元素，并`continue` 继续。
    5. 匹配成功，重新设置队列的头元素
    6. 唤醒匹配操作的线程
    7. 返回元素
    

##### awaitFulfill源码

```java
//主逻辑: 若节点是 head.next 则进行 spins 一会, 若不是, 则调用 LockSupport.park / parkNanos(), 直到其他的线程对其进行唤醒
Object awaitFulfill(QNode s, E e, boolean timed, long nanos) {          // s 是刚刚入队的操作节点，e是操作（也可说是数据）
    /* Same idea as TransferStack.awaitFulfill */
    // 根据timed标识计算截止时间
    final long deadline = timed ? System.nanoTime() + nanos : 0L;
    Thread w = Thread.currentThread();
    // 计算空旋时间
    int spins = ((head.next == s) ?
                 (timed ? maxTimedSpins : maxUntimedSpins) : 0);    //若当前节点为head.next时才进行自旋

    for (;;) {

        //如果当前线程发生中断，那么尝试取消该操作
        if (w.isInterrupted())
            s.tryCancel(e); // 若线程中断, 直接将 item 设置成了 this, 在 transfer 中会对返回值进行判断

        Object x = s.item;  // 获取s的元素域

        // s.item ！=e则返回，生成s节点的时候，s.item是等于e的，当取消操作或者匹配了操作的时候会进行更改。匹配对应transfer方法中的 m.casItem(x, e) 代码
        if (x != e) // 在进行线程阻塞->唤醒, 线程中断, 等待超时, 这时 x != e,直接return 回去
            return x;

        //如果设置了超时等待
        if (timed) {    // 设置了timed
            nanos = deadline - System.nanoTime();   // 计算继续等待的时间

            //发生了超时，尝试取消该操作
            if (nanos <= 0L) {
                //将s的item域指向this实现取消操作
                s.tryCancel(e);
                continue;
            }
        }

        //自旋控制
        if (spins > 0)  // 空旋时间大于0，空旋
            --spins;

        //设置等待线程 waiter
        else if (s.waiter == null)
            // 设置等待线程
            s.waiter = w;

        else if (!timed)
            // 禁用当前线程并设置了阻塞者
            LockSupport.park(this);
        else if (nanos > spinForTimeoutThreshold)   //自旋次数过了, 直接 + timeout 方式 park
            LockSupport.parkNanos(this, nanos);
    }
}
```
这个方法应该比较好理解，就是等待匹配操作的到来，如果设置了超时等待，那么就等待一定时间，如果发生超时，那么就取消该操作，如果没有设置超时等待，那么就一直等待，直到匹配的操作到来，或者发生中断（取消该操作）。

##### clean源码

```java
//用于移除已经被取消的结点。pred 是s的前驱
void clean(QNode pred, QNode s) {
    s.waiter = null; // forget thread
    /*
     * At any given time, exactly one node on list cannot be
     * deleted -- the last inserted node. To accommodate this,
     * if we cannot delete s, we save its predecessor as
     * "cleanMe", deleting the previously saved version
     * first. At least one of node s or the node previously
     * saved can always be deleted, so this always terminates.
     */
    while (pred.next == s) { // pred的next域为s // Return early if already unlinked
        QNode h = head; // 获取头结点
        // 获取头结点的next域
        QNode hn = h.next;   // Absorb cancelled first node as head

        //操作取消了，那么推进head
        if (hn != null && hn.isCancelled()) {   // hn不为null并且hn被取消，重新设置头结点
            advanceHead(h, hn);
            continue;
        }

        // 获取尾结点，保证对尾结点的读一致性
        QNode t = tail;      // Ensure consistent read for tail
        if (t == h) // 尾结点为头结点，表示队列为空
            return;

        // 获取尾结点的next域
        QNode tn = t.next;
        if (t != tail)  // t不为尾结点，不一致，重试
            continue;

        //tn 理应为null ,如果不为空，说明其它线程进行了入队操作，更新tail
        if (tn != null) {   //tn不为null ，说明其他线程修改了尾节点
            advanceTail(t, tn); //设置新的尾节点
            continue;
        }

        // s!=t ,则s不是尾节点了，本来最开始是尾节点，其它线程进行了入队操作
        if (s != t) {   // s不为尾结点，移除s      // If not tail, try to unsplice
            QNode sn = s.next;

            // 如果s.next ==s ,则已经离开队列
            //设置pred的后继为s的后继，将s从队列中删除
            if (sn == s || pred.casNext(s, sn))
                return;
        }

        // 到这里，那么s就是队尾，那么暂时不能删除
        //cleanMe标识的是需要删除节点的前驱
        QNode dp = cleanMe;

        //有需要删除的节点
        if (dp != null) { // dp不为null，断开前面被取消的结点   // Try unlinking previous cancelled node
            QNode d = dp.next;
            QNode dn;
            /**
             * cleamMe 失效的情况有：
             *   (1)cleanMe的后继而空（cleanMe 标记的是需要删除节点的前驱）
             *   (2)cleanMe的后继等于自身（这个前面有分析过）
             *   (3)需要删除节点的操作没有被取消
             *   (4)被删除的节点不是尾节点且其后继节点有效，并将待删除节点删除
             */
             if (d == null ||               // d is gone or
                d == dp ||                 // d is off list or
                !d.isCancelled() ||        // d not cancelled or
                (d != t &&                 // d not tail and
                 (dn = d.next) != null &&  //   has successor
                 dn != d &&                //   that is on list
                 dp.casNext(d, dn)))       // d unspliced

                //清除cleanMe节点
                casCleanMe(dp, null);

            //dp==pred 表示已经被设置过了
            if (dp == pred)
                return;      // s is already saved node
        } else if (casCleanMe(null, pred))  // 原来的 cleanMe 是 null, 则将 pred 标记为 cleamMe 为下次 清除 s 节点做标识
            return;          // Postpone cleaning s
    }
}
```
链表的删除只需要设置节点指针的关系就可以了，这个clean的方法就在链表基础上增加了一个逻辑：就是如果删除的节点不是尾节点，那么可以直接进行删除，如果删除的节点是尾节点，那么用cleanMe标记需要删除的节点的前驱，这样在下一轮的clean的过程将会清除打了标记的节点。 

1. pred.next == s 表示节点s还在队列中 
2. 从队头开始，如果节点被取消，那么则推进head。 
3. 如果队列发生变化，则重新开始或者推进tail. 
4. 如果删除的节点不是尾节点，那么进行cas 删除操作 
5. 删除的节点是尾节点，那么需要先检查cleanMe是否已经被标记了 
6. 如果cleanMe已经被标记了，那么检查标记是否还有效 
7. cleamMe 失效的情况有：(1)cleanMe的后继而空（cleanMe 标记的是需要删除节点的前驱），(2)cleanMe的后继等于自身（这个前面有分析过）,(3)需要删除节点的操作没有被取消，(4)被删除的节点不是尾节点且其后继节点有效。 
8. 如果cleanMe没有被标记，那么就标记为被删除节点的前驱。


#### 公平队列SynchronousQueue 总结

公平的SynchronousQueue 使用队列来实现的，其内部没有使用传统的锁来控制，而是通过CAS来完成，因此需要反复检查状态是否有效，SynchronousQueue 将相同的操作（这个操作可以是put也可以是take）入队，不同的操作（put 对应take,take 对应put）那么就是互相匹配的操作，当请求的操作和队尾的操作相同时入队，否则进行出队匹配，如果有超时设置，那么就进行超时操作，入队后的线程被阻塞，直到匹配的操作到来，如果有取消操作需要被清除，如果该操作节点不是尾节点，那么执行删除操作，否则用cleanMe标记其父节点，在下一轮的clean过程中再根据情况进行删除。 

#### 非公平模式 TransferStack

##### 数据结构

```java
static final class TransferStack<E> extends Transferer<E> {
    // 表示消费数据的消费者
    static final int REQUEST    = 0;
    // 表示生产数据的生产者
    static final int DATA       = 1;
    //表示该操作节点处于真正匹配状态
    static final int FULFILLING = 2;
    // 栈头结点
    volatile SNode head;
    
    //内部节点
    static final class SNode {
            // 下一个结点
            volatile SNode next;        // next node in stack
            // 相匹配的结点
            volatile SNode match;       // the node matched to this
            // 等待的线程
            volatile Thread waiter;     // to control park/unpark
            // 元素项
            Object item;                // data; or null for REQUESTs

            /**
             * 模式。有四种可能。
             *  1) REQUEST 0000
             *  2) DATA    0001
             *  3) REQUEST（0000）| FULFILLING（0010） =  0010 表示消费者匹配到了生成者。REQUEST为此次操作，即出队
             *  4）DATA（0001）| FULFILLING（0010） =  0011 表示生成者匹配到了消费者。DATA为此次操作，即入队
             *  后两种是在成功匹配后，将节点入队时设置的节点模式。通过：FULFILLING|mode 计算节点模式
             */
            int mode;
    }
}
```
TransferStack中定义了三个状态：REQUEST表示消费者，DATA表示生产者，FULFILLING，表示操作匹配状态。同时还包含一个head域，表示栈顶。


###### transfer源码

```java
E transfer(E e, boolean timed, long nanos) {
/*
 * Basic algorithm is to loop trying one of three actions:
 *
 * 1. If apparently empty or already containing nodes of same
 *    mode, try to push node on stack and wait for a match,
 *    returning it, or null if cancelled.
 *
 * 2. If apparently containing node of complementary mode,
 *    try to push a fulfilling node on to stack, match
 *    with corresponding waiting node, pop both from
 *    stack, and return matched item. The matching or
 *    unlinking might not actually be necessary because of
 *    other threads performing action 3:
 *
 * 3. If top of stack already holds another fulfilling node,
 *    help it out by doing its match and/or pop
 *    operations, and then continue. The code for helping
 *    is essentially the same as for fulfilling, except
 *    that it doesn't return the item.
 */

SNode s = null; // constructed/reused as needed
// 根据e确定此次转移的模式（是put or take）
int mode = (e == null) ? REQUEST : DATA;

for (;;) {  //无限循环
    SNode h = head; //保存头结点

    //相同的操作模式
    if (h == null || h.mode == mode) {  // 头结点为null或者头结点的模式与此次转移的模式相同 empty or same-mode

        if (timed && nanos <= 0) {      // 设置了timed并且等待时间小于等于0，表示不能等待，需要立即操作 can't wait
            if (h != null && h.isCancelled())   // 头结点不为null并且头结点被取消
                casHead(h, h.next);     // 重新设置头结点（弹出之前的头结点） pop cancelled node
            else    // 头结点为null或者头结点没有被取消
                return null;
        } else if (casHead(h, s = snode(s, e, h, mode))) {  // 生成一个SNode结点；将原来的head头结点设置为该结点的next结点；将head头结点设置为该结点。入队
            //等待匹配操作
            SNode m = awaitFulfill(s, timed, nanos);    // 空旋或者阻塞直到s结点被FulFill操作所匹配
            if (m == s) {               // 匹配的结点为s结点（s结点被取消） wait was cancelled
                clean(s);
                return null;
            }

            //s 还没有离开栈，帮助其离开
            if ((h = head) != null && h.next == s)  // h重新赋值为head头结点，并且不为null；头结点的next域为s结点，表示有结点插入到s结点之前，完成了匹配
                casHead(h, s.next);     // 比较并替换head域（移除插入在s之前的结点和s结点） // help s's fulfiller
            return (E) ((mode == REQUEST) ? m.item : s.item);    // 根据此次转移的类型返回元素
        }
    } else if (!isFulfilling(h.mode)) { // 不同的模式，并且没有处于正在匹配状态，则进行匹配 // try to fulfill
        //节点取消，更新head
        if (h.isCancelled())            // already cancelled
            // 比较并替换head域（弹出头结点）
            casHead(h, h.next);         // pop and retry

        else if (casHead(h, s=snode(s, e, h, FULFILLING|mode))) {   //入队一个新节点，并且处于匹配状态（表示h正在匹配）
            for (;;) { // loop until matched or waiters disappear
                // s.next 是真正的操作节点
                SNode m = s.next;       // m is s's match

                // next域为null
                if (m == null) {        // all waiters are gone
                    // 比较并替换head域
                    casHead(s, null);   // pop fulfill node
                    s = null;           // use new node next time
                    break;              // restart main loop
                }

                // m结点的next域
                SNode mn = m.next;
                if (m.tryMatch(s)) {    // 尝试匹配，并且成功
                    // 比较并替换head域（弹出s结点和m结点）
                    casHead(s, mn);     // pop both s and m
                    return (E) ((mode == REQUEST) ? m.item : s.item);   // 根据此次转移的类型返回元素
                } else                  // lost match
                    // 没有匹配成功，说明有其他线程已经匹配了，把m移出
                    s.casNext(m, mn);   // help unlink
            }
        }
    } else {         // 头结点是真正匹配的状态，那么就帮助它匹配                  // help a fulfiller
        // 保存头结点的next域
        SNode m = h.next;               // m is h's match
        if (m == null)                  // waiter is gone
            // 比较并替换head域（m被其他结点匹配了，需要弹出h）
            casHead(h, null);           // pop fulfilling node
        else {
            // 获取m结点的next域
            SNode mn = m.next;
            if (m.tryMatch(h))  // 帮助匹配 / help match
                // 比较并替换head域（弹出h和m结点）
                casHead(h, mn);         // pop both h and m
            else    // 匹配不成功                     // lost match
                // 比较并替换next域（移除m结点）
                h.casNext(m, mn);       // help unlink
        }
    }
}
}
```
大体逻辑：
1. 如果当前栈为空或者获取节点模式与栈顶模式一样，则尝试将节点加入栈内，同时通过阻塞（或自旋一段时间，如果有超时设置，则进行超时等待）等待节点匹配，最后返回匹配的节点或者本身（被取消） 
2. 如果栈不为空且节点的模式与首节点模式匹配，则尝试将该节点打上FULFILLING标记，然后加入栈中，与相应的节点匹配，成功后将这两个节点弹出栈并返回匹配节点的数据 
3. 如果有节点在匹配，那么帮助这个节点完成匹配和出栈操作，然后在主循环中继续执行。


#### awaitFulfill源码

```java
SNode awaitFulfill(SNode s, boolean timed, long nanos) {
    /*
     * When a node/thread is about to block, it sets its waiter
     * field and then rechecks state at least one more time
     * before actually parking, thus covering race vs
     * fulfiller noticing that waiter is non-null so should be
     * woken.
     *
     * When invoked by nodes that appear at the point of call
     * to be at the head of the stack, calls to park are
     * preceded by spins to avoid blocking when producers and
     * consumers are arriving very close in time.  This can
     * happen enough to bother only on multiprocessors.
     *
     * The order of checks for returning out of main loop
     * reflects fact that interrupts have precedence over
     * normal returns, which have precedence over
     * timeouts. (So, on timeout, one last check for match is
     * done before giving up.) Except that calls from untimed
     * SynchronousQueue.{poll/offer} don't check interrupts
     * and don't wait at all, so are trapped in transfer
     * method rather than calling awaitFulfill.
     */
    // 根据timed标识计算截止时间
    final long deadline = timed ? System.nanoTime() + nanos : 0L;
    // 获取当前线程
    Thread w = Thread.currentThread();
    // 根据s确定空旋等待的时间
    int spins = (shouldSpin(s) ?
                 (timed ? maxTimedSpins : maxUntimedSpins) : 0);
    for (;;) {  // 无限循环，确保操作成功
        if (w.isInterrupted())  // 当前线程被中断
            // 取消s结点，原理是将节点s的match域设置为this
            s.tryCancel();
        // 获取s结点的match域
        SNode m = s.match;
        if (m != null)   // m不为null，存在匹配结点
            return m;
        if (timed) {
            // 确定继续等待的时间
            nanos = deadline - System.nanoTime();
            if (nanos <= 0L) {  // 继续等待的时间小于等于0，等待超时
                s.tryCancel();   // 取消s结点
                continue;
            }
        }
        if (spins > 0)  // 空旋等待的时间大于0
            spins = shouldSpin(s) ? (spins-1) : 0;   // 确定是否还需要继续空旋等待
        else if (s.waiter == null)   // 等待线程为null
            // 设置waiter线程为当前线程
            s.waiter = w; // establish waiter so can park next iter
        else if (!timed)
            // 禁用当前线程并设置了阻塞者
            LockSupport.park(this);
        else if (nanos > spinForTimeoutThreshold)   // 继续等待的时间大于阈值
            LockSupport.parkNanos(this, nanos); // 禁用当前线程，最多等待指定的等待时间，除非许可可用
    }
}
```
整体逻辑：
1. 获取超时截止时间，当前线程及空旋等待时间
2. 进入无限循环 
    1. 判断当前线程是否被中断。是，将节点s取消，即将节点s的 `match域设置为this`；否，进入下一步。
    2. 获取节点s的match域值m。如果m不为空，则说明节点s已被匹配，直接返回m；否则，进行下一步。
    3. 如果设置了超时时间，则判断是否超时。超时，则将节点s取消，`continue`继续循环。
    4. 如果没有设置超时，则空旋一段时间，最后将当前线程阻塞。


#### clean源码

```java
// 移除从栈顶头结点开始到该结点（不包括）之间的所有已取消结点。
void clean(SNode s) {
    // s结点的item设置为null
    s.item = null;   // forget item
    // s结点的waiter设置为null
    s.waiter = null; // forget thread

    /*
     * At worst we may need to traverse entire stack to unlink
     * s. If there are multiple concurrent calls to clean, we
     * might not see s if another thread has already removed
     * it. But we can stop when we see any node known to
     * follow s. We use s.next unless it too is cancelled, in
     * which case we try the node one past. We don't check any
     * further because we don't want to doubly traverse just to
     * find sentinel.
     */
    //被删除节点的后继
    SNode past = s.next;

    //后继节点操作被取消，直接移除该节点
    if (past != null && past.isCancelled()) // next域不为null并且next域被取消
        past = past.next;   // 重新设置past

    // Absorb cancelled nodes at head
    SNode p;
    //如果栈顶是取消了的操作节点，则移除
    while ((p = head) != null && p != past && p.isCancelled())   // 从栈顶头结点开始到past结点（不包括），将连续的取消结点移除
        casHead(p, p.next);

    // Unsplice embedded nodes
    //因为是单向链表，因此需要从head 开始，遍历到被删除节点的后继
    while (p != null && p != past) {     // 移除上一步骤没有移除的非连续的取消结点
        SNode n = p.next;   // 获取p的next域
        if (n != null && n.isCancelled())   // n不为null并且n被取消
            p.casNext(n, n.next);    // 比较并替换next域
        else
            p = n;
    }
}
```
主要功能：从栈顶删除节点s。大体逻辑：
1. 将待删除节点s的item域，waiter域都设置为null
2. 获取待删除节点s的next域节点past
3. 如果节点past不为null且节点past已被取消，则先删除其后继节点
4. 如果head操作节点也被取消，那么就重新更新头节点
5. 因为是单链表，因此需要遍历链表，从head 到s的后继中，有被取消了的操作的节点，那么就移除掉。 


#### isFulfilling源码

```java
//是否处于匹配状态
static boolean isFulfilling(int m) { 
    return (m & FULFILLING) != 0; 
}
```

#### tryMatch源码

```java
boolean tryMatch(SNode s) {
    if (match == null &&
        UNSAFE.compareAndSwapObject(this, matchOffset, null, s)) {
        Thread w = waiter;
        if (w != null) {    // waiters need at most one unpark
            waiter = null;
            LockSupport.unpark(w);
        }
        return true;
    }
    return match == s;
}
```
如果`match`不为空，则说明当前操作已经被匹配或者取消了，则直接判断 `match == s` 是否为true；否则CAS将当前节点的`match`值设置为节点s，如果成功，则通过`LockSupport.unpark(w);` 唤起线程。

#### TransferStack 总结
将一个操作和栈顶进行匹配，如果和栈顶是相同的操作，那么就直接入栈，如果和栈顶不是相同的操作（也就是匹配的操作，take匹配put,put匹配take）,那么现在先不急出栈，因为此时可能有线程真正入栈，为了避免出现操作错误，这里加了一个环节，如果操作是匹配的(即需要出栈)，那么入栈一个节点，并标记是真正匹配状态，表示的是栈顶操作节点真正匹配，如果其他线程发现这个过程，那么就会帮助其匹配（使其顺序完成出栈工作），完成匹配过后，再进行自身的操作. 


#### 参考文章
[Java 并发 --- 阻塞队列之SynchronousQueue源码分析(真心不错)](http://blog.csdn.net/u014634338/article/details/78419445)

[【死磕Java并发】-----J.U.C之阻塞队列：SynchronousQueue](http://www.jianshu.com/p/9d2c706e45b7?utm_campaign=maleskine&utm_content=note&utm_medium=seo_notes&utm_source=recommendation)

[SynchronousQueue 源码分析 (基于Java 8)](http://www.jianshu.com/p/95cb570c8187)
