---
title: ConcurrentLinkedQueue源码学习
date: 2017-11-10 14:52:42
tags: java并发集合
---

#### 前言
[ConcurerntLinkedQueue](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ConcurrentLinkedQueue.html)
一个 **`基于链接节点的无界线程安全的FIFO队列`**。和其他大部分并发集合类似，此队列不允许使用null元素。

![ConcurerntLinkedQueue数据结构](http://images2015.cnblogs.com/blog/616953/201605/616953-20160531094819899-444696106.png)

#### 源码解析
##### 内部数据结构

```java
public class ConcurrentLinkedQueue<E> extends AbstractQueue<E>
        implements Queue<E>, java.io.Serializable {
        
    //内部节点类
    private static class Node<E> {
        volatile E item;    //表示元素
        volatile Node<E> next;  //表示下一个结点
    }
    
    private transient volatile Node<E> head;    //头结点
    private transient volatile Node<E> tail;    //尾节点
    
    //遍历器类
    private class Itr implements Iterator<E> {
        /**
         * Next node to return item for.
         */
        private Node<E> nextNode;

        /**
         * nextItem holds on to item fields because once we claim
         * that an element exists in hasNext(), we must return it in
         * the following next() call even if it was in the process of
         * being removed when hasNext() was called.
         */
        private E nextItem;

        /**
         * Node of the last returned item, to support remove.
         */
        private Node<E> lastRet;
    }
}
```
从源码可知，`ConcurrentLinkedQueue` 继承了`AbstractQueue`抽象类并实现了`Queue` 与 `Serializable` 接口。其内部数据结构有：

<!-- more -->

+ **内部类Node**：Node是ConcurrentLinkedQueue存储结构的基本单元。
+ **迭代器类Itr**：和大多数集合一样，都使用了迭代器模式，所以其内部提供一个元素遍历器。
+ **头尾节点**

    + <font color=red>head引用的不变性和可变性</font>
    
        + 不变性(invariants)
        
            1. 所有未删除节点，都能从head通过调用succ()方法遍历可达。
            2. head头结点不能为null
            3. head节点的next域不能引用到自身
        
        + 可变性(Non-invariants)
        
            1. head节点的item域可能为null，也可能不为null
            2. 允许tail滞后于head，即：从head开始遍历队列，不一定能到达tail
    
    + <font color=red>tail 引用的不变性和可变性</font>
        
        + 不变性(invariants)
        
            1. 通过tail调用succ()方法，最后节点总是可达的
            2. tail节点不能为null
         
        + 可变性(Non-invariants)
        
            1. tail节点的item域可能为null，也可能不为null
            2. 允许tail滞后于head，即：从head开始遍历队列，不一定能到达tail
            3. tail节点的next域可以引用到自身


##### 构造函数

```java
//默认会构造一个 dummy 节点
public ConcurrentLinkedQueue() {
    head = tail = new Node<E>(null);
}

public ConcurrentLinkedQueue(Collection<? extends E> c) {
    Node<E> h = null, t = null;
    for (E e : c) {
        checkNotNull(e);    // 保证元素不为空
        Node<E> newNode = new Node<E>(e);   // 新生一个结点
        if (h == null)  // 头结点为null
            h = t = newNode;     // 赋值头结点与尾结点
        else {
            t.lazySetNext(newNode); // 设置尾结点的next域
            t = newNode;     // 重新赋值尾结点
        }
    }
    if (h == null)
        h = t = new Node<E>(null);
    head = h;
    tail = t;
}
```
由源码可知，`ConcurrentLinkedQueue` 提供了两个构造函数：无参构造函数和有参构造函数。
    
    无参构造函数只是简单的构造一个dummy节点，
      该结点的item域和next域都为null，然后头尾节点都指向这个dummy节点；
    
    有参构造函数是使用参数集合构造一个ConcurrentLinkedQueue。


##### add源码

```java
public boolean add(E e) {
    return offer(e);
}
```
在往队列中添加元素时，会调用add方法，add方法内部会调用offer方法。

##### offer源码

```java
//在队列的尾部添加元素。因队列是无界的，因此方法不会返回false
public boolean offer(E e) {
    checkNotNull(e);  //即将入队的元素不能为空
    final Node<E> newNode = new Node<E>(e);     //创建节点

    //对于入队操作，采用失败即重试的方式，直到入队成功
    for (Node<E> t = tail, p = t;;) {   // 初始化变量 p = t = tail
        Node<E> q = p.next; //q为p节点的next节点
        if (q == null) {    // q结点为null，说明p为最后一个节点
            // p is last node
            if (p.casNext(null, newNode)) { //CAS操作p的next节点
                // Successful CAS is the linearization point
                // for e to become an element of this queue,
                // and for newNode to become "live".
                //在成功将newNode设置为p的next节点后，p指向
                if (p != t) // hop two nodes at a time 每每经过一次 p = q 操作(向后遍历节点), 则 p != t 成立, 这个也说明 tail 滞后于 head 的体现
                    casTail(t, newNode);  // Failure is OK.
                return true;
            }
            // Lost CAS race to another thread; re-read next
        }
        else if (p == q)   // 成立, 则说明p是pool()时调用 "updateHead" 导致的(删除头节点); 此时说明 tail 指针已经 fallen off queue
            // We have fallen off list.  If tail is unchanged, it
            // will also be off-list, in which case we need to
            // jump to head, from which all live nodes are always
            // reachable.  Else the new tail is a better bet.
            //在t != tail时，说明在多线程环境下，不但有线程操作了head节点，也有线程添加了新节点，所以p新赋值为tail；在t == tail 时，说明其他线程只进行了head节点的更新，所以p新赋值为head。然后在循环， 直到找到 node.next = null 的节点
            p = (t != (t = tail)) ? t : head;
        else    //在插入第二个节点时，才会更新tail节点
            // Check for tail updates after two hops.
            p = (p != t && t != (t = tail)) ? t : q;
    }
}
```
offer函数用于将指定元素插入此队列的尾部。大致逻辑为：
1. 判断指定元素是否为空，为空，抛出异常，方法结束
2. 在指定元素不为空的情况下， 组装节点
3. 采用失败即重试的方式，插入节点。
    1. 获取尾节点p（p = tail）的next节点q
    2. 如果`q==null`，说明p节点为最后一个节点，则CAS进行设置p的next节点。设置成功后，如果又必须，则重新设置tail节点。方法结束。
    3. 如果 `p == q`，说明其他线程有调用了 `updateHead方法`，此时tail指针已`fallen off list`，则重新设置p指针，进行再次循环。
    4. 否则，也重新设置p指针，进行再次循环。
    
> 单线程模拟入队操作

+ 队列初始态

    ![队列初始态](http://images2015.cnblogs.com/blog/616953/201605/616953-20160531150623024-33105347.png)
    
    由源码可知，队列的初始化时，创建了一个dummy节点，且头尾节点都指向这个dummy节点。
    
+ 元素10入队

    ![入队后状态](http://images2015.cnblogs.com/blog/616953/201605/616953-20160531150707492-188834822.png)

    由源码可知，在插入元素10时，由于tail节点的next节点为null，因此直接设置新节点为tail节点的next节点。在设置成功后，因为条件`p != t` 不成立，所以tail节点的指针不变。
    
+ 元素20入队

    ![入队后状态](http://images2015.cnblogs.com/blog/616953/201605/616953-20160531150752680-504185441.png)
    
    由源码可知，在队列中只有元素10时，tail节点的next节点依然不为null，且条件`p == q`也不成立。所以执行`p = (p != t && t != (t = tail)) ? t : q;`，则`p = tail.next`。进入第二次循环。二次循环成功后，条件`p != t` 成立，所以tail节点的指针发送改变。即:`hop two nodes at a time `。


##### poll源码

```java
//从队列的头结点取数据
public E poll() {
    restartFromHead:
    for (;;) {
        for (Node<E> h = head, p = h, q;;) {
            E item = p.item;

            if (item != null && p.casItem(item, null)) {    // 若 node.item != null, 则进行cas操作, cas成功则返回值
                // Successful CAS is the linearization point
                // for item to be removed from this queue.
                if (p != h) // hop two nodes at a time  //一次跳过两个节点
                    updateHead(h, ((q = p.next) != null) ? q : p);  //进行 cas 更新 head ; "(q = p.next) != null" 怕出现p此时是尾节点了; 在 ConcurrentLinkedQueue 中正真的尾节点只有1个(必须满足node.next = null)
                return item;
            }
            else if ((q = p.next) == null) {    //queue是空的, p是尾节点
                updateHead(h, p);   //这一步除了更新head 外, 还是helpDelete删除队列操作, 删除 p 之前的节点
                return null;
            }
            else if (p == q)    //说明 p节点已经是删除了的head节点
                continue restartFromHead;
            else
                p = q;
        }
    }
}
```
poll函数用于获取并移除此队列的头，如果此队列为空，则返回null。。大致逻辑为：
1. 从头结点开始，将头结点赋值给变量p，循环队列
2. 如果`p.item != null` 且CAS操作成功，则返回 p.item 的值。在返回值之前，需要判断是否需要更新头结点。
3. 如果`p.next == null`，则调用`updateHead`方法设置头结点，然后返回
4. 如果`p == q`，则说明p节点已是删除的头结点，则重新循环。
5. 否则，设置`p = p.next`，继续循环。

> 单线程模拟出队操作

+ 队列初始态

    ![队列初始态](http://images2015.cnblogs.com/blog/616953/201605/616953-20160531150752680-504185441.png)

    使用上面模拟入队后的队列

+ 首次调用poll函数

    <a data-flickr-embed="true"  href="https://www.flickr.com/photos/157389715@N05/37586647844/in/dateposted-public/" title="poll方法调用"><img src="https://farm5.staticflickr.com/4525/37586647844_faa740949b_b.jpg" width="662" height="353" alt="poll方法调用"></a>
    
    由源码可知，在首次调用poll函数时，由于头结点的item域为null，则执行`p = p.next`，然后再次循环。此时，节点的item域不为空，则返回item域值。在返回item域值之前，判断条件`p != h == true`，则调用`updateHead`方法设置头结点。

+ 再次调用poll函数

    <a data-flickr-embed="true"  href="https://www.flickr.com/photos/157389715@N05/38248234816/in/dateposted-public/" title="poll方法调用"><img src="https://farm5.staticflickr.com/4537/38248234816_d7e10d86a5_b.jpg" width="662" height="353" alt="poll方法调用"></a>
    
    由源码可知，再次调用poll方法时，头结点的item域不为空，所以直接返回item域的值，又因为条件`p != h = false`，所以头结点不会变更，只是头结点的item域变成了null。
    
##### updateHead源码

```java
/**
 * Tries to CAS head to p. If successful, repoint old head to itself
 * as sentinel for succ(), below.
 *
 * 将节点 p设置为新的头节点(这是原子操作),成功之后将原头节点的next指向它自己, 直接变成一个哨兵节点(为queue节点删除及garbage做准备)
 */
final void updateHead(Node<E> h, Node<E> p) {
    if (h != p && casHead(h, p))
        h.lazySetNext(h);   //将节点h变为哨兵节点，为queue节点删除及garbage做准备
}
```
该方法的功能是设置头结点。

##### remove方法

```java
public boolean remove(Object o) {
    if (o == null) return false;    // 元素为null，返回
    Node<E> pred = null;
    for (Node<E> p = first(); p != null; p = succ(p)) { // 获取第一个存活的结点
        E item = p.item;    // 第一个存活结点的item值
        if (item != null &&
            o.equals(item) &&
            p.casItem(item, null)) {    // 找到item相等的结点，并且将该结点的item设置为null
            Node<E> next = succ(p); // p的后继结点
            if (pred != null && next != null)
                pred.casNext(p, next);  //在pred不为null且next不为null的时候，设置pred的next域
            return true;
        }
        pred = p;
    }
    return false;
}
```
此函数用于从队列中移除指定元素的单个实例（如果存在）。大致逻辑为从头结点开始遍历，如果item域值为指定元素，则将对于的item域置为null，否则获取下一节点，循环处理。

##### first源码

```java
//返回队列中第一个没有被删除的节点，另一个版本的poll/peek方法
Node<E> first() {
    restartFromHead:
    for (;;) {  // 无限循环，确保成功
        for (Node<E> h = head, p = h, q;;) {
            boolean hasItem = (p.item != null); // p结点的item域是否为null
            if (hasItem || (q = p.next) == null) {      // item不为null或者next域为null
                updateHead(h, p);   // 更新头结点
                return hasItem ? p : null;
            }
            else if (p == q)     //说明 p节点已经是删除了的head节点
                continue restartFromHead;
            else
                p = q;   // p赋值为p.next，再次循环
        }
    }
}

```
first函数用于找到链表中第一个存活的结点。

##### succ方法

```java
/**
 * Returns the successor of p, or the head node if p.next has been
 * linked to self, which will only be true if traversing with a
 * stale pointer that is now off the list.
 *
 * 如果p.next已链接到p本身自己，则返回头节点；否则返回p的后继节点。
 * p == p.next 只有在使用不在列表中的陈旧指针进行遍历时，才会返回true
 * p.next == p 是 updateHead() 操作导致的。
 */
final Node<E> succ(Node<E> p) {
    //在并发队列中，node.next并不一定就是node的后继节点，还有一种特殊情况，就是node指向一个哨兵节点，该哨兵节点为queue节点删除及garbage做准备
    Node<E> next = p.next;
    return (p == next) ? head : next;
}
```
succ方法用于获取结点的下一个结点。如果结点的next域指向自身，则返回head头结点，否则，返回next结点。

> 模拟remove方法调用

+ 队列初始态

    ![队列初始态](http://images2015.cnblogs.com/blog/616953/201605/616953-20160531150752680-504185441.png)

    使用上面模拟入队后的队列
    
+ 执行remove(10)

    <a data-flickr-embed="true"  href="https://www.flickr.com/photos/157389715@N05/38271826052/in/dateposted-public/" title="remove方法调用"><img src="https://farm5.staticflickr.com/4574/38271826052_4a02c44f22_b.jpg" width="662" height="353" alt="remove方法调用"></a>
    
    由源码可知，在调用remove方法时，会先调用first方法获取队列的头元素，在first方法的时候，由于`p.item != null = false` 且 `(q = p.next == null) = false`，所以执行`p = p.next`，进入二次循环。在二次循环时，会执行`updateHead`设置头结点。所以原头结点被特殊处理，等后续清理。


##### size方法

```java
/**
 * Additionally, if elements are added or removed during execution
 * of this method, the returned result may be inaccurate.  Thus,
 * this method is typically not very useful in concurrent
 * applications.
*/
public int size() {
    int count = 0;
    for (Node<E> p = first(); p != null; p = succ(p))    // 从第一个存活的结点开始往后遍历
        if (p.item != null)  // 结点的item域不为null
            // Collection.size() spec says to max out
            if (++count == Integer.MAX_VALUE)   // 增加计数，若达到最大值，则跳出循环
                break;
    return count;
}
```
此函数用于返回ConcurrenLinkedQueue的大小，从第一个存活的结点（first）开始，往后遍历链表，当结点的item域不为null时，增加计数，之后返回大小。该方法在多线程情况下，并没有多大用处。


##### contains方法

```java
 public boolean contains(Object o) {
    if (o == null) return false;
    for (Node<E> p = first(); p != null; p = succ(p)) {     //循环遍历队列
        E item = p.item;
        if (item != null && o.equals(item))     //在节点的item值与指导元素相等时，返回
            return true;
    }
    return false;
}
```


#### 参考博客

[JDK1.8源码分析之ConcurrentLinkedQueue（五）](http://www.cnblogs.com/leesf456/p/5539142.html)

[ConcurrentLinkedQueue 源码分析 (基于Java 8)](http://www.jianshu.com/p/08e8b0c424c0)