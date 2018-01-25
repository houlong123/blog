---
title: ConcurrentHashMap源码学习
date: 2017-11-07 14:50:00
tags: java并发集合
---

#### 前言
HashMap是我们开发中使用最频繁的集合，但是它是非线程安全的，在涉及到多线程并发的情况时，put操作有可能会引起死循环。

解决方案有`HashTable`和`Collections.synchronizedMap(hashMap)`，它们是线程安全的，它们的实现原理是在所有涉及到多线程操作的都加上了synchronized关键字来锁住整个table，这就意味着所有的线程都在竞争一把锁，在多线程的环境下，它是安全的，但是无疑是效率低下的。相对于粗暴的锁住整个table的做法，ConcurrentHashMap则使用了一种 **`锁分段技术`**，实现并发的更新操作。底层采用 **` Node数组+链表+红黑树 `** 的存储结构。  

![image](http://incdn1.b0.upaiyun.com/2017/07/665f644e43731ff9db3d341da5c827e1.png)


#### 源码解析

<!-- more -->

##### 内部数据结构

```java
public class ConcurrentHashMap<K,V> extends AbstractMap<K,V>
    implements ConcurrentMap<K,V>, Serializable {
    
    /***************** 常量值 *******************/
    
    //最大表容量
    private static final int MAXIMUM_CAPACITY = 1 << 30;    //2^30
    //默认初始化表容量
    private static final int DEFAULT_CAPACITY = 16;
    //最大的数组长度（非2的次幂）
    static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;    //2^31 - 1
    //表的默认并发级别
    private static final int DEFAULT_CONCURRENCY_LEVEL = 16;
    //表的加载因子
    private static final float LOAD_FACTOR = 0.75f;
    static final int TREEIFY_THRESHOLD = 8;     //链表转树阀值，大于8时，转为数结构
    static final int UNTREEIFY_THRESHOLD = 6;   //树转链表阀值，小于等于6（tranfer时，lc、hc=0两个计数器分别++记录原bin、新binTreeNode数量，<=UNTREEIFY_THRESHOLD 则untreeify(lo)）。【仅在扩容tranfer时才可能树转链表】
    static final int MIN_TREEIFY_CAPACITY = 64; //转换为树的最小表容量
    private static final int MIN_TRANSFER_STRIDE = 16;  //每个转移步骤的最小重新分配次数
    private static int RESIZE_STAMP_BITS = 16;  //sizeCtl中用于生成邮票的位数
    // 2^15-1，帮助扩容的最大线程数
    private static final int MAX_RESIZERS = (1 << (32 - RESIZE_STAMP_BITS)) - 1;
    // 32-16=16，sizeCtl中记录size大小的偏移量
    private static final int RESIZE_STAMP_SHIFT = 32 - RESIZE_STAMP_BITS;
    static final int MOVED     = -1; // hash for forwarding nodes（ForwardingNode的hash值）
    static final int TREEBIN   = -2; // hash for roots of trees（树根节点的hash值）
    static final int RESERVED  = -3; // hash for transient reservations（ReservationNode的hash值）
    static final int HASH_BITS = 0x7fffffff; // usable bits of normal node hash
    static final int NCPU = Runtime.getRuntime().availableProcessors();
    
    
    /***************** 内部节点 *******************/
    
    static class Node<K,V> implements Entry<K,V> {
        final int hash;
        final K key;
        volatile V val; //带有同步锁的value
        volatile Node<K,V> next;    //带有同步锁的next指针
    }
    
    
    //存放Node节点的数组。在首次插入节点时进行初始化。大小为2的整数次幂。即2^x
    transient volatile Node<K,V>[] table;
    private transient volatile Node<K,V>[] nextTable;   //在扩容时不为null
    
    //hash表正在初始化或扩容的控制标识。负数代表正在初始化或者扩容操作。
    // -1表示正在初始化，-N表示有N-1个线程正在进行扩容操作。正数和0代表还没有初始化，这个数值表示初始化或者下一次扩容的大小。
    private transient volatile int sizeCtl;
    
    //扩容时使用
    static final class ForwardingNode<K,V> extends Node<K,V> {
        final Node<K,V>[] nextTable;
    }
    
    //树节点
    static final class TreeNode<K,V> extends Node<K,V> {
        TreeNode<K,V> parent;  // red-black tree links
        TreeNode<K,V> left;
        TreeNode<K,V> right;
        TreeNode<K,V> prev;    // needed to unlink next upon deletion
        boolean red;
    }

    //底层的树节点存储的为TreeBin对象
    static final class TreeBin<K,V> extends Node<K,V> {
        TreeNode<K,V> root;
        volatile TreeNode<K,V> first;
        volatile Thread waiter;
        volatile int lockState;
        // values for lockState
        static final int WRITER = 1; // set while holding write lock
        static final int WAITER = 2; // set when waiting for write lock
        static final int READER = 4; // increment value for setting read lock
    }
}
```
`ConcurrentHashMap`的底层数据结构为：**`Node数组 + 链表 + 红黑树`**，因此`ConcurrentHashMap`的内部数据主要有：
+ **内部类Node**：Node是ConcurrentHashMap存储结构的基本单元，继承于HashMap中的Entry，用于存储数据。
+ **内部类TreeNode**：TreeNode继承与Node，但是数据结构换成了二叉树结构，它是红黑树的数据的存储结构，用于红黑树中存储数据，当链表的节点数大于8时会转换成红黑树的结构。
+ **内部类TreeBin**：封装TreeNode的容器。在树底层结构存放的是TreeBin对象，而不是TreeNode对象。
+ **内部类ForwardingNode**：一个特殊的节点，hash值为-1，其中存储nextTable的引用。只有table发生扩容时，ForwardingNode才会发挥作用。


##### 内部类源码分析

###### Node内部类

```java
/**
 *  重点：该Node内部类与HashMap中定义的Node类很相似，但是区别为：
 *  1. 内部的value和next属性设置了volatile同步锁
 *  2. 不允许直接调用setValue方法修改Node的value域
 *  3. 增加了find方法辅助map.get()方法
 *  4. 键值都不能为null
 */
static class Node<K,V> implements Entry<K,V> {
    final int hash;
    final K key;
    volatile V val; //带有同步锁的value
    volatile Node<K,V> next;    //带有同步锁的next指针

    Node(int hash, K key, V val, Node<K,V> next) {
        this.hash = hash;
        this.key = key;
        this.val = val;
        this.next = next;
    }

    public final K getKey()       { return key; }
    public final V getValue()     { return val; }

    /**
     与HashMap的方法
     public final int hashCode() {
        return Objects.hashCode(key) ^ Objects.hashCode(value);
     }
     不同，HashMap中的hashCode()方法使用了Objects.hashCode(key)来获取hashCode，
     在Objects.hashCode(key)方法内，对key进行了判断操作。
     之所以在ConcurrentHashMap中不那么操作是因为ConcurrentHashMap中的Node的键值对规定不能为null
     */
    public final int hashCode()   { return key.hashCode() ^ val.hashCode(); }

    public final String toString(){ return key + "=" + val; }

    //不允许直接改变value的值
    public final V setValue(V value) {
        throw new UnsupportedOperationException();
    }

    public final boolean equals(Object o) {
        Object k, v, u; Entry<?,?> e;
        return ((o instanceof Map.Entry) &&
                (k = (e = (Entry<?,?>)o).getKey()) != null &&
                (v = e.getValue()) != null &&
                (k == key || k.equals(key)) &&
                (v == (u = val) || v.equals(u)));
    }

    /**
     * Virtualized support for map.get(); overridden in subclasses.
     增加find方法辅助get方法  ，HashMap中的Node类中没有此方法
     */
    Node<K,V> find(int h, Object k) {
        Node<K,V> e = this;
        if (k != null) {
            do {
                K ek;
                if (e.hash == h &&
                    ((ek = e.key) == k || (ek != null && k.equals(ek))))
                    return e;
            } while ((e = e.next) != null);
        }
        return null;
    }
}
```
Node数据结构很简单，从上可知，就是一个链表，但是`只允许对数据进行查找，不允许进行修改`。

###### TreeNode内部类

```java
/**
 * Nodes for use in TreeBins
 *
 * 直接继承自ConcurrentHashMap类的Node节点，
 * 与HashMap类中的TreeNode不同，HashMap中的TreeNode继承自LinkedHashMap.Entry<K,V>
 */
static final class TreeNode<K,V> extends Node<K,V> {
    TreeNode<K,V> parent;  // red-black tree links
    TreeNode<K,V> left;
    TreeNode<K,V> right;
    TreeNode<K,V> prev;    // needed to unlink next upon deletion
    boolean red;

    TreeNode(int hash, K key, V val, Node<K,V> next,
             TreeNode<K,V> parent) {
        super(hash, key, val, next);
        this.parent = parent;
    }

    Node<K,V> find(int h, Object k) {
        return findTreeNode(h, k, null);
    }

    /**
     * Returns the TreeNode (or null if not found) for the given key
     * starting at given root.
     */
    //根据key查找 从根节点开始找出相应的TreeNode，
    final TreeNode<K,V> findTreeNode(int h, Object k, Class<?> kc) {
        if (k != null) {
            TreeNode<K,V> p = this;
            do  {
                int ph, dir; K pk; TreeNode<K,V> q;
                TreeNode<K,V> pl = p.left, pr = p.right;
                if ((ph = p.hash) > h)
                    p = pl;
                else if (ph < h)
                    p = pr;
                else if ((pk = p.key) == k || (pk != null && k.equals(pk)))
                    return p;
                else if (pl == null)
                    p = pr;
                else if (pr == null)
                    p = pl;
                else if ((kc != null ||
                          (kc = comparableClassFor(k)) != null) &&
                         (dir = compareComparables(kc, k, pk)) != 0)
                    p = (dir < 0) ? pl : pr;
                else if ((q = pr.findTreeNode(h, k, kc)) != null)
                    return q;
                else
                    p = pl;
            } while (p != null);
        }
        return null;
    }
}
```

###### TreeBin内部类

```java
static final class TreeBin<K,V> extends Node<K,V> {
    TreeNode<K,V> root;
    volatile TreeNode<K,V> first;
    volatile Thread waiter;
    volatile int lockState;
    // values for lockState
    static final int WRITER = 1; // set while holding write lock
    static final int WAITER = 2; // set when waiting for write lock
    static final int READER = 4; // increment value for setting read lock

    /**
     * Creates bin with initial set of nodes headed by b.
     */
    //将节点组装为树结构
    TreeBin(TreeNode<K,V> b) {
        super(TREEBIN, null, null, null);
        this.first = b;
        TreeNode<K,V> r = null;
        for (TreeNode<K,V> x = b, next; x != null; x = next) {
            next = (TreeNode<K,V>)x.next;
            x.left = x.right = null;
            if (r == null) {    //设置根节点
                x.parent = null;
                x.red = false;
                r = x;
            }
            else {
                K k = x.key;
                int h = x.hash;
                Class<?> kc = null;
                for (TreeNode<K,V> p = r;;) {
                    int dir, ph;
                    K pk = p.key;
                    if ((ph = p.hash) > h)  //待插入节点的hash小于根节点的hash，则放在根节点的左侧，dir<=0，表示为左叶节点
                        dir = -1;
                    else if (ph < h)
                        dir = 1;

                    //下面的else if的功能主要是判断待插入节点k为根节点pk的什么节点。若dir<=0，表示为左叶节点，否则为右侧节点
                    else if ((kc == null && (kc = comparableClassFor(k)) == null) ||      //comparableClassFor方法是判断k是否为Comparable的子类，是，返回k的类型，否，返回null
                             (dir = compareComparables(kc, k, pk)) == 0)    //compareComparables方法在pk.getClass() == kc的情况下，返回 k.compareTo(pk)，否则0
                        dir = tieBreakOrder(k, pk);     //tieBreakOrder方法是判断k和pk的排序

                    TreeNode<K,V> xp = p;
                    if ((p = (dir <= 0) ? p.left : p.right) == null) {
                        x.parent = xp;
                        if (dir <= 0)
                            xp.left = x;
                        else
                            xp.right = x;
                        r = balanceInsertion(r, x);    //平衡红叉树
                        break;
                    }
                }
            }
        }
        this.root = r;
        assert checkInvariants(root);
    }
    ...
}
```
TreeBin是封装TreeNode的容器，它提供转换黑红树的一些条件和锁的控制。在链表需要转为树结构时，会使用到此类。

###### ForwardingNode内部类

```java
//一个特殊的节点，hash值为-1，其中存储nextTable的引用。只有table发生扩容时，ForwardingNode才会发挥作用。
static final class ForwardingNode<K,V> extends Node<K,V> {
    final Node<K,V>[] nextTable;
    ForwardingNode(Node<K,V>[] tab) {
        super(MOVED, null, null, null);
        this.nextTable = tab;
    }

    Node<K,V> find(int h, Object k) {
        // loop to avoid arbitrarily deep recursion on forwarding nodes
        outer: for (Node<K,V>[] tab = nextTable;;) {
            Node<K,V> e; int n;
            if (k == null || tab == null || (n = tab.length) == 0 ||
                (e = tabAt(tab, (n - 1) & h)) == null)
                return null;
            for (;;) {
                int eh; K ek;
                if ((eh = e.hash) == h &&
                    ((ek = e.key) == k || (ek != null && k.equals(ek))))
                    return e;
                if (eh < 0) {
                    if (e instanceof ForwardingNode) {
                        tab = ((ForwardingNode<K,V>)e).nextTable;
                        continue outer;
                    }
                    else
                        return e.find(h, k);
                }
                if ((e = e.next) == null)
                    return null;
            }
        }
    }
}
```
只有table发生扩容的时候，ForwardingNode才会发挥作用，作为一个占位符放在table中表示当前节点为null或则已经被移动。


##### ConcurrentHashMap源码解析

###### 构造函数

```java
//无参构造函数
public ConcurrentHashMap() {}

//在初始化的时候，ConcurrentHashMap的大小为大于指定值initialCapacity的最少的2的N次幂。
//ConcurrentHashMap在构造函数中只会初始化sizeCtl值，并不会直接初始化table，而是延缓到第一次put操作。
public ConcurrentHashMap(int initialCapacity) {
    if (initialCapacity < 0)
        throw new IllegalArgumentException();
    int cap = ((initialCapacity >= (MAXIMUM_CAPACITY >>> 1)) ?
               MAXIMUM_CAPACITY :
               tableSizeFor(initialCapacity + (initialCapacity >>> 1) + 1));
    this.sizeCtl = cap;
}

//有参构造函数
public ConcurrentHashMap(Map<? extends K, ? extends V> m) {
    this.sizeCtl = DEFAULT_CAPACITY;
    putAll(m);
}
    
public ConcurrentHashMap(int initialCapacity, float loadFactor) {
    this(initialCapacity, loadFactor, 1);
}

/**
 *
 * @param initialCapacity
 * @param loadFactor
 * @param concurrencyLevel  表示能够同时更新ConccurentHashMap且不产生锁竞争的最大线程数。默认值为16
 */
public ConcurrentHashMap(int initialCapacity,
                         float loadFactor, int concurrencyLevel) {
    if (!(loadFactor > 0.0f) || initialCapacity < 0 || concurrencyLevel <= 0)
        throw new IllegalArgumentException();
    if (initialCapacity < concurrencyLevel)   // Use at least as many bins
        initialCapacity = concurrencyLevel;   // as estimated threads
    long size = (long)(1.0 + (long)initialCapacity / loadFactor);
    int cap = (size >= (long)MAXIMUM_CAPACITY) ?
        MAXIMUM_CAPACITY : tableSizeFor((int)size);
    this.sizeCtl = cap;
}
```
在有参的`ConcurrentHashMap`构造函数中，主要的功能是初始化 **`sizeCtl`** 的值。

###### put 源码

```java
public V put(K key, V value) {
    return putVal(key, value, false);
}
```
由源码可知，其内部调用了putVal()方法

###### putVal源码

```java
final V putVal(K key, V value, boolean onlyIfAbsent) {
    if (key == null || value == null) throw new NullPointerException(); //ConcurrentHashMap限制key-value都不能为null
    int hash = spread(key.hashCode());  //两次hash，减少hash冲突，可以均匀分布
    int binCount = 0;
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
        if (tab == null || (n = tab.length) == 0)   //如果table表为空，则首先初始化
            tab = initTable();
        /**
         *  为什么长度必须是16或者2的N次幂？
         *    在定位元素的index时，index = (tab.length - 1) & hash，
         *    则tab.length - 1的值是所有二进制位全为1，这种情况下，index的结果等同于hashCode后几位的值。只要hashCode本身均匀分布，则Hash算法的结果也均匀分布
         */
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {    //table中定位索引位置index = (n - 1) & hash，n是table的大小。并获取对应索引的元素f。 若如果f为null，说明table中这个位置第一次插入元素。
            if (casTabAt(tab, i, null,
                         new Node<K,V>(hash, key, value, null)))
                break;                   // 如果CAS插入操作成功，则跳出循环。no lock when adding to empty bin
        }
        else if ((fh = f.hash) == MOVED)    //如果f的hash值为-1，说明当前f是ForwardingNode节点，意味有其它线程正在扩容，则一起进行扩容操作。
            tab = helpTransfer(tab, f);
        else {  //其余情况：将新的Node节点按链表或红黑树的方式插入到合适的位置。
            V oldVal = null;
            synchronized (f) {  //采用同步内置锁实现并发。hash值相同的链表的头节点
                if (tabAt(tab, i) == f) {   //验证稳定性。防止被其它线程修改
                    if (fh >= 0) {  //如果f.hash >= 0，说明f是链表结构的头结点，遍历链表，如果找到对应的node节点，则修改value，否则在链表尾部加入节点。
                        binCount = 1;   //节点个数
                        for (Node<K,V> e = f;; ++binCount) {
                            K ek;
                            if (e.hash == hash &&
                                ((ek = e.key) == key ||
                                 (ek != null && key.equals(ek)))) { //在链表中存在对应的key，则在onlyIfAbsent=FALSE情况下，替换旧值
                                oldVal = e.val;
                                if (!onlyIfAbsent)
                                    e.val = value;
                                break;
                            }
                            Node<K,V> pred = e;
                            if ((e = e.next) == null) { //在链表尾部加入节点。
                                pred.next = new Node<K,V>(hash, key,
                                                          value, null);
                                break;
                            }
                        }
                    }
                    else if (f instanceof TreeBin) { //如果f是TreeBin类型节点，说明f是红黑树根节点，则在树结构上遍历元素，更新或增加节点。
                        Node<K,V> p;
                        binCount = 2;
                        if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                       value)) != null) {
                            oldVal = p.val;
                            if (!onlyIfAbsent)
                                p.val = value;
                        }
                    }
                }
            }

            //插入成功后，如果插入的是链表节点，则要判断下该桶位是否要转化为树
            if (binCount != 0) {
                if (binCount >= TREEIFY_THRESHOLD)  //如果链表中节点数binCount >= TREEIFY_THRESHOLD(默认是8)，则把链表转化为红黑树结构。
                    treeifyBin(tab, i);
                if (oldVal != null)
                    return oldVal;
                break;
            }
        }
    }
    addCount(1L, binCount);     //检查当前容量是否需要进行扩容
    return null;
}
```
putVal()方法的整理流程为：
1. key,value是否为空，为空，抛异常，结束。非空，执行2
2. 将key二次hash，算出hash值
3. 遍历hash表。
    
    1）判断hash表是否为初始化，如果为初始化，则进行初始化。
    
    2）算出待插入元素在hash表中的索引 `index=(length-1) & hash`，若索引index处元素  (f = tab[index]) == null，则直接CAS进行插入操作，成功，跳出循环。
     
    3）如果 `f.hash == -1`，说明有其他线程正在扩容，则一起进行扩容操作。

    4）否则，将元素插入数组中。在插入数据时，首先将f节点加锁（synchronized），如果 `f.hash > 0`，则说明f为链表结构，则进行链表插入操作；如果 `f instanceof TreeBin` 为树结构，则进行数的相应插入操作。插入完成后，判断是否需要转为树结构。最后跳出循环。

4. 元素插入成功后，检查是否需要进行扩容。
5. 方法结束。


###### spread源码

```java
static final int spread(int h) {
    //HASH_BITS = 0x7fffffff
    return (h ^ (h >>> 16)) & HASH_BITS;
}
```
该方法的主要功能是给定的值进行两次hash，减少hash冲突，可以均匀分布。

###### initTable源码

```java
private final Node<K,V>[] initTable() {
    Node<K,V>[] tab; int sc;
    while ((tab = table) == null || tab.length == 0) {  //整个table还没有被完全初始化
        if ((sc = sizeCtl) < 0)  //如果sizeCtl < 0 ，表示其他线程正在初始化，则放弃此次操作，把线程挂起。自旋
            Thread.yield(); // lost initialization race; just spin
        else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {  //如果获得初始化权限，则CAS方法将sizeCtl置为-1，防止其他线程进入
            try {
                if ((tab = table) == null || tab.length == 0) {
                    int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                    @SuppressWarnings("unchecked")
                    Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                    table = tab = nt;
                    sc = n - (n >>> 2);     //初始化数组后，将sizeCtl的值改为0.75*n
                }
            } finally {
                sizeCtl = sc;
            }
            break;
        }
    }
    return tab;
}
```
sizeCtl默认为0，如果ConcurrentHashMap实例化时有传参数，sizeCtl会是一个2的幂次方的值。所以执行第一次put操作的线程会执行Unsafe.compareAndSwapInt方法修改sizeCtl为-1，有且只有一个线程能够修改成功，其它线程通过Thread.yield()让出CPU时间片等待table初始化完成。

###### helpTransfer源码

```java
//在多线程情况下，如果发现其它线程正在扩容，则帮助转移元素。
final Node<K,V>[] helpTransfer(Node<K,V>[] tab, Node<K,V> f) {  // 调用之前，nextTable一定已存在
    Node<K,V>[] nextTab; int sc;
    if (tab != null && (f instanceof ForwardingNode) &&
        (nextTab = ((ForwardingNode<K,V>)f).nextTable) != null) {
        int rs = resizeStamp(tab.length);   //标志位

        while (nextTab == nextTable && table == tab &&
               (sc = sizeCtl) < 0) {
            if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                sc == rs + MAX_RESIZERS || transferIndex <= 0)
                break;
            if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1)) {
                transfer(tab, nextTab); //调用扩容方法，直接进入复制阶段
                break;
            }
        }
        return nextTab;
    }
    return table;
}
```
其实helpTransfer（）方法的目的就是调用多个工作线程一起帮助进行扩容，这样的效率就会更高。

###### resizeStamp源码

```java
static final int resizeStamp(int n) {
    return Integer.numberOfLeadingZeros(n) | (1 << (RESIZE_STAMP_BITS - 1));    //numberOfLeadingZeros:返回n对应32位二进制数左侧0的个数，如9（1001）返回28
}
```

###### transfer源码

```java
private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
    int n = tab.length, stride;
    // 每核处理的量小于16，则强制赋值16
    if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
        stride = MIN_TRANSFER_STRIDE; // subdivide range
    if (nextTab == null) {            // initiating
        try {
            @SuppressWarnings("unchecked")
            Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];  //构造一个nextTable对象 它的容量是原来的两倍
            nextTab = nt;
        } catch (Throwable ex) {      // try to cope with OOME
            sizeCtl = Integer.MAX_VALUE;
            return;
        }
        nextTable = nextTab;
        transferIndex = n;
    }

    int nextn = nextTab.length;
    ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);  //构造一个连节点指针 用于标志位
    boolean advance = true; //并发扩容的关键属性 如果等于true 说明这个节点已经处理过
    boolean finishing = false; // to ensure sweep before committing nextTab
    for (int i = 0, bound = 0;;) {
        Node<K,V> f; int fh;

        //这个while循环体的作用就是在控制i--  通过i--可以依次遍历原hash表中的节点
        while (advance) {
            int nextIndex, nextBound;
            if (--i >= bound || finishing)
                advance = false;
            else if ((nextIndex = transferIndex) <= 0) {
                i = -1;
                advance = false;
            }
            // 用CAS计算得到的transferIndex
            else if (U.compareAndSwapInt
                     (this, TRANSFERINDEX, nextIndex,
                      nextBound = (nextIndex > stride ?
                                   nextIndex - stride : 0))) {
                bound = nextBound;
                i = nextIndex - 1;
                advance = false;
            }
        }


        if (i < 0 || i >= n || i + n >= nextn) {
            int sc;
            if (finishing) {  //如果所有的节点都已经完成复制工作  就把nextTable赋值给table 清空临时对象nextTable
                nextTable = null;
                table = nextTab;
                sizeCtl = (n << 1) - (n >>> 1);  //扩容阈值设置为原来容量的1.5倍  依然相当于现在容量的0.75倍
                return;
            }

            //利用CAS方法更新这个扩容阈值，在这里面sizectl值减一，说明新加入一个线程参与到扩容操作
            if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
                if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
                    return;
                finishing = advance = true;
                i = n; // recheck before commit
            }
        }
        else if ((f = tabAt(tab, i)) == null)  //如果遍历到的节点为空 则放入ForwardingNode指针
            advance = casTabAt(tab, i, null, fwd);
        else if ((fh = f.hash) == MOVED)    //如果遍历到ForwardingNode节点  说明这个点已经被处理过了 直接跳过  这里是控制并发扩容的核心
            advance = true; // already processed
        else {
            //节点上锁
            synchronized (f) {
                if (tabAt(tab, i) == f) {
                    Node<K,V> ln, hn;

                    //如果fh>=0 证明这是一个Node节点
                    if (fh >= 0) {
                        int runBit = fh & n;

                        //以下的部分完成的工作是构造两个链表  一个是原链表  另一个是原链表的反序排列
                        Node<K,V> lastRun = f;
                        for (Node<K,V> p = f.next; p != null; p = p.next) {
                            int b = p.hash & n;
                            if (b != runBit) {
                                runBit = b;
                                lastRun = p;
                            }
                        }
                        if (runBit == 0) {
                            ln = lastRun;
                            hn = null;
                        }
                        else {
                            hn = lastRun;
                            ln = null;
                        }
                        for (Node<K,V> p = f; p != lastRun; p = p.next) {
                            int ph = p.hash; K pk = p.key; V pv = p.val;
                            if ((ph & n) == 0)
                                ln = new Node<K,V>(ph, pk, pv, ln);
                            else
                                hn = new Node<K,V>(ph, pk, pv, hn);
                        }
                        setTabAt(nextTab, i, ln);
                        setTabAt(nextTab, i + n, hn);
                        setTabAt(tab, i, fwd);
                        advance = true;
                    }
                    else if (f instanceof TreeBin) {    //对TreeBin对象进行处理  与上面的过程类似
                        TreeBin<K,V> t = (TreeBin<K,V>)f;
                        TreeNode<K,V> lo = null, loTail = null;
                        TreeNode<K,V> hi = null, hiTail = null;
                        int lc = 0, hc = 0;

                        //构造正序和反序两个链表
                        for (Node<K,V> e = t.first; e != null; e = e.next) {
                            int h = e.hash;
                            TreeNode<K,V> p = new TreeNode<K,V>
                                (h, e.key, e.val, null, null);
                            if ((h & n) == 0) {
                                if ((p.prev = loTail) == null)
                                    lo = p;
                                else
                                    loTail.next = p;
                                loTail = p;
                                ++lc;
                            }
                            else {
                                if ((p.prev = hiTail) == null)
                                    hi = p;
                                else
                                    hiTail.next = p;
                                hiTail = p;
                                ++hc;
                            }
                        }

                        //如果扩容后已经不再需要tree的结构 反向转换为链表结构
                        ln = (lc <= UNTREEIFY_THRESHOLD) ? untreeify(lo) :
                            (hc != 0) ? new TreeBin<K,V>(lo) : t;

                        hn = (hc <= UNTREEIFY_THRESHOLD) ? untreeify(hi) :
                            (lc != 0) ? new TreeBin<K,V>(hi) : t;
                        setTabAt(nextTab, i, ln);
                        setTabAt(nextTab, i + n, hn);
                        setTabAt(tab, i, fwd);
                        advance = true;
                    }
                }
            }
        }
    }
}
```
哦，my god，太复杂，看不懂。🤦


###### treeifyBin源码

```java
//将数组下标为index上的元素，在必要的情况下转换为树结构。
private final void treeifyBin(Node<K,V>[] tab, int index) {
    Node<K,V> b; int n, sc;
    if (tab != null) {
        if ((n = tab.length) < MIN_TREEIFY_CAPACITY)    //容量<64，则table两倍扩容，不转树了,因为这个阈值扩容可以减少hash冲突，不必要去转红黑树
            tryPresize(n << 1); //扩容函数
        else if ((b = tabAt(tab, index)) != null && b.hash >= 0) {
            synchronized (b) {  //将数组下标index上的元素加锁
                if (tabAt(tab, index) == b) {   //在没有其他线程操作的情况下
                    TreeNode<K,V> hd = null, tl = null;
                    for (Node<K,V> e = b; e != null; e = e.next) {
                        TreeNode<K,V> p =
                            new TreeNode<K,V>(e.hash, e.key, e.val,
                                              null, null);  //将节点组装为树节点
                        if ((p.prev = tl) == null)
                            hd = p;
                        else
                            tl.next = p;
                        tl = p;
                    }
                    setTabAt(tab, index, new TreeBin<K,V>(hd));    //在树底层结构存放的是TreeBin对象，而不是TreeNode对象；
                }
            }
        }
    }
}
```
在成功插入元素后，如果插入位置对应的链表中的节点数大于等于8，则将链表转为红黑树结构。在转红黑树结构时，若hash表当前容量小于64，则不转数，只进行两倍扩容即可。在当前容量大于64时，通过加锁，将链表转为树结构。

###### addCount源码

```java
private final void addCount(long x, int check) {
    CounterCell[] as; long b, s;
    //更新baseCount，table的数量，counterCells表示元素个数的变化
    if ((as = counterCells) != null ||
        !U.compareAndSwapLong(this, BASECOUNT, b = baseCount, s = b + x)) {
        CounterCell a; long v; int m;
        boolean uncontended = true;
        //如果多个线程都在执行，则CAS失败，执行fullAddCount，全部加入count
        if (as == null || (m = as.length - 1) < 0 ||
            (a = as[ThreadLocalRandom.getProbe() & m]) == null ||
            !(uncontended = U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))) {
            fullAddCount(x, uncontended);
            return;
        }
        if (check <= 1)
            return;
        s = sumCount();
    }
    //check>=0表示需要进行扩容操作
    if (check >= 0) {
        Node<K,V>[] tab, nt; int n, sc;
        while (s >= (long)(sc = sizeCtl) && (tab = table) != null &&
               (n = tab.length) < MAXIMUM_CAPACITY) {
            int rs = resizeStamp(n);
            if (sc < 0) {
                if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                    sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                    transferIndex <= 0)
                    break;
                if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                    transfer(tab, nt);
            }
            else if (U.compareAndSwapInt(this, SIZECTL, sc,
                                         (rs << RESIZE_STAMP_SHIFT) + 2))
                transfer(tab, null);
            s = sumCount();
        }
    }
}
```
数据加入成功后，现在调用addCount()方法计算ConcurrentHashMap的size，在原来的基础上加一


###### get源码

```java
public V get(Object key) {
    Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
    int h = spread(key.hashCode()); //获取key的哈希值

    if ((tab = table) != null && (n = tab.length) > 0 &&
        (e = tabAt(tab, (n - 1) & h)) != null) {  //在table不为空，table长度大于0，在(n - 1) & h 的位置上有元素e
        if ((eh = e.hash) == h) {
            if ((ek = e.key) == key || (ek != null && key.equals(ek)))
                return e.val;
        }
        //hash值为负值表示正在扩容，这个时候查的是ForwardingNode的find方法来定位到nextTable来
        //查找，查找到就返回
        else if (eh < 0)
            return (p = e.find(h, key)) != null ? p.val : null;

        //既不是首节点也不是ForwardingNode，那就往下遍历
        while ((e = e.next) != null) {  //在节点链表上查询key对应的值
            if (e.hash == h &&
                ((ek = e.key) == key || (ek != null && key.equals(ek))))
                return e.val;
        }
    }
    return null;
}
```
ConcurrentHashMap的get操作的流程很简单，也很清晰，可以分为三个步骤来描述

1. 计算hash值，定位到该table索引位置，如果是首节点符合就返回
2. 如果遇到扩容的时候，会调用标志正在扩容节点ForwardingNode的find方法，查找该节点，匹配就返回
3. 以上都不符合的话，就往下遍历节点，匹配就返回，否则最后就返回null

###### size源码

```java
public int size() {
    long n = sumCount();
    return ((n < 0L) ? 0 :
            (n > (long)Integer.MAX_VALUE) ? Integer.MAX_VALUE :
            (int)n);
}
```

###### sumCount源码

```java
final long sumCount() {
    CounterCell[] as = counterCells; CounterCell a;
    long sum = baseCount;
    if (as != null) {
        for (int i = 0; i < as.length; ++i) {
            if ((a = as[i]) != null)
                sum += a.value;
        }
    }
    return sum;
}
```
在JDK1.8版本中，对于size的计算，在扩容和addCount()方法就已经有处理了。而JDK1.8版本中，ConcurrentHashMap还提供了另外一种方法可以获取大小，这个方法就是mappingCount。

###### mappingCount源码

```java
public long mappingCount() {
    long n = sumCount();
    return (n < 0L) ? 0L : n; // ignore transient negative values
}
```

###### containsValue源码

```java
public boolean containsValue(Object value) {
    if (value == null)
        throw new NullPointerException();
    Node<K,V>[] t;
    if ((t = table) != null) {  //遍历map，只要出现一次 v==value ，直接返回true
        Traverser<K,V> it = new Traverser<K,V>(t, t.length, 0, t.length);
        for (Node<K,V> p; (p = it.advance()) != null; ) {
            V v;
            if ((v = p.val) == value || (v != null && value.equals(v)))
                return true;
        }
    }
    return false;
}
```
检查在所有映射(k,v)中只要出现一次及以上的v==value，返回true。需注意：这个方法可能需要一个完全遍历Map，因此比containsKey要慢的多

###### containsKey源码

```java
public boolean containsKey(Object key) {    //直接调用get(int key)方法即可，如果有返回值，则说明是包含key的
    return get(key) != null;
}
```

###### remove源码

```java
public V remove(Object key) {
    return replaceNode(key, null, null);
}
```

###### replaceNode源码

```java
final V replaceNode(Object key, V value, Object cv) {
    int hash = spread(key.hashCode());  //计算hash值
    for (Node<K,V>[] tab = table;;) {   //死循环，直到找到
        Node<K,V> f; int n, i, fh;
        if (tab == null || (n = tab.length) == 0 ||
            (f = tabAt(tab, i = (n - 1) & hash)) == null)   //hash表为null或者在(n - 1) & hash位置上无节点，则跳出循环
            break;
        else if ((fh = f.hash) == MOVED)    //如果检测到其它线程正在扩容，则先帮助扩容，然后再来寻找，可见扩容的优先级之高
            tab = helpTransfer(tab, f);
        else {
            V oldVal = null;
            boolean validated = false;
            synchronized (f) {   //开始锁住这个桶，然后进行比对寻找满足(key,value)的节点
                if (tabAt(tab, i) == f) {   //重新检查，避免由于多线程的原因table[i]已经被修改
                    if (fh >= 0) {  //链表节点
                        validated = true;
                        for (Node<K,V> e = f, pred = null;;) {
                            K ek;
                            if (e.hash == hash &&
                                ((ek = e.key) == key ||
                                 (ek != null && key.equals(ek)))) { //满足条件就是找到key出现的节点位置
                                V ev = e.val;
                                if (cv == null || cv == ev ||
                                    (ev != null && cv.equals(ev))) {
                                    oldVal = ev;
                                    if (value != null)  //value不为空，则更新值
                                        e.val = value;
                                    else if (pred != null)  //value为空，则删除此节点
                                        pred.next = e.next;
                                    else
                                        setTabAt(tab, i, e.next);   //符合条件的节点e为头结点的情况
                                }
                                break;
                            }
                            //更改指向，继续向后循环
                            pred = e;
                            if ((e = e.next) == null)   //如果为到链表末尾了，则直接退出即可
                                break;
                        }
                    }
                    else if (f instanceof TreeBin) {
                        validated = true;
                        TreeBin<K,V> t = (TreeBin<K,V>)f;
                        TreeNode<K,V> r, p;
                        if ((r = t.root) != null &&
                            (p = r.findTreeNode(hash, key, null)) != null) {
                            V pv = p.val;
                            if (cv == null || cv == pv ||
                                (pv != null && cv.equals(pv))) {
                                oldVal = pv;
                                if (value != null)
                                    p.val = value;
                                else if (t.removeTreeNode(p))
                                    setTabAt(tab, i, untreeify(t.first));
                            }
                        }
                    }
                }
            }
            if (validated) {
                if (oldVal != null) {
                    if (value == null)  //如果删除了节点，则要减1
                        addCount(-1L, -1);
                    return oldVal;
                }
                break;
            }
        }
    }
    return null;
}
```

remove方法的实现思路也比较简单。如下；

1. 先根据key的hash值计算书其在table的位置 i。

2. 检查table[i]是否为空，如果为空，则返回null，否则进行3

3. 在table[i]存储的链表(或树)中开始遍历比对寻找，如果找到节点符合key的，则判断value是否为null来决定是否是更新还是删除该节点。


###### clear源码

```java
public void clear() {
    long delta = 0L; // negative number of deletions
    int i = 0;
    Node<K,V>[] tab = table;
    while (tab != null && i < tab.length) {
        int fh;
        Node<K,V> f = tabAt(tab, i);
        if (f == null)
            ++i;
        else if ((fh = f.hash) == MOVED) {
            tab = helpTransfer(tab, f);
            i = 0; // restart
        }
        else {
            synchronized (f) {
                if (tabAt(tab, i) == f) {
                    Node<K,V> p = (fh >= 0 ? f :
                                   (f instanceof TreeBin) ?
                                   ((TreeBin<K,V>)f).first : null);
                    while (p != null) {
                        --delta;
                        p = p.next;
                    }
                    setTabAt(tab, i++, null);
                }
            }
        }
    }
    if (delta != 0L)
        addCount(delta, -1);
}
```

#### 参考文章

[深入并发包 ConcurrentHashMap](http://www.importnew.com/26049.html)

[JDK 1.8 ConcurrentHashMap 源码剖析](http://www.voidcn.com/article/p-ugrqibkl-bbz.html)

[《Java源码分析》：ConcurrentHashMap JDK1.8](http://blog.csdn.net/u010412719/article/details/52145145)
