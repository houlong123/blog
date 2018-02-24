---
title: LinkedHashMap源码学习
date: 2018-02-24 15:06:36
tags: java基础集合
---

#### 前言
LinkedHashMap 继承自 HashMap，<font color=red>在 HashMap 基础上，通过维护一条双向链表，解决了 HashMap不能随时保持遍历顺序和插入顺序一致的问题。</font>LinkedHashMap 直接复用了HashMap中的许多方法，仅为维护双向链表覆写了部分方法。关于[HashMap源码](https://houlong123.github.io/2018/02/02/HashMap源码学习/) 在此之前已经分析过，不在讲述。

#### 数据结构图
在 [HashMap源码](https://houlong123.github.io/2018/02/02/HashMap源码学习/)分析文章中，我们已知道 HashMap的底层采用 <font color=red>`Node数组+链表+红黑树`</font> 的存储结构，结构示意图大致如下：


<a data-flickr-embed="true"  href="https://www.flickr.com/photos/157389715@N05/38640533260/in/dateposted-public/" title="HashMap底层数据结构"><img src="https://farm5.staticflickr.com/4708/38640533260_448fe112c7.jpg" width="500" height="233" alt="HashMap底层数据结构"></a>

---

LinkedHashMap 在上面结构的基础上，<!--more-->增加了一条双向链表，使得上面的结构可以保持键值对的插入顺序。同时通过对链表进行相应的操作，实现了访问顺序相关逻辑。其结构可能如下图：

<a data-flickr-embed="true"  href="https://www.flickr.com/photos/157389715@N05/39554813965/in/dateposted-public/" title="LinkedHashMap底层数据结构"><img src="https://farm5.staticflickr.com/4759/39554813965_365181ffb0.jpg" width="500" height="237" alt="LinkedHashMap底层数据结构"></a>
每当有新键值对节点插入，新节点最终会接在 tail 引用指向的节点后面。而 tail 引用则会移动到新的节点上，这样一个双向链表就建立起来了。

#### 源码分析
##### 数据结构

```java
//继承自HashMap，一个有序的Map接口实现，这里的有序指的是元素可以按插入顺序或访问顺序排列。最后插入或者最后访问的节点位于双向链表的尾部。
public class LinkedHashMap<K,V>
    extends HashMap<K,V>
    implements Map<K,V> {
       
    // 双向链表节点
    static class Entry<K,V> extends Node<K,V> {
        Entry<K,V> before, after;
        Entry(int hash, K key, V value, Node<K,V> next) {
            super(hash, key, value, next);
        }
    }
    
    /**
     * The head (eldest) of the doubly linked list.
     */
    // 保存双向链表的head和tail节点。该两个节点是为了维持元素的顺序
    transient Entry<K,V> head;

    /**
     * The tail (youngest) of the doubly linked list.
     */
    transient Entry<K,V> tail;
    final boolean accessOrder;
}
```
从 LinkedHashMap 的类声明中，可知该类继承自 `HashMap类` 并实现了 `Map接口`。在该类内部，有个节点类 `Entry` 继承自`HashMap.Node类`。之所以有该类的存在，是为了解决 HashMap 不能随时保持遍历顺序和插入顺序一致的问题。这也是 LinkedHashMap 有序的实现基础。`accessOrder` 属性用于申明 LinkedHashMap 的迭代顺序。`true: 访问顺序；false: 插入顺序`。

##### 构造函数

```java
/**
 * 生成一个空的LinkedHashMap,并指定其容量大小和负载因子，
 * 默认将accessOrder设为false，按插入顺序排序
 */
public LinkedHashMap(int initialCapacity, float loadFactor) {
    super(initialCapacity, loadFactor);
    accessOrder = false;
}

/**
 * 生成一个空的LinkedHashMap,并指定其容量大小，负载因子使用默认的0.75，
 * 默认将accessOrder设为false，按插入顺序排序
 */
public LinkedHashMap(int initialCapacity) {
    super(initialCapacity);
    accessOrder = false;
}

/**
 * 生成一个空的HashMap,容量大小使用默认值16，负载因子使用默认值0.75
 * 默认将accessOrder设为false，按插入顺序排序.
 */
public LinkedHashMap() {
    super();
    accessOrder = false;
}

/**
 * 生成一个空的LinkedHashMap,并指定其容量大小和负载因子，
 * 默认将accessOrder设为true，按访问顺序排序
 */
public LinkedHashMap(int initialCapacity,
                     float loadFactor,
                     boolean accessOrder) {
    super(initialCapacity, loadFactor);
    this.accessOrder = accessOrder;
}

public LinkedHashMap(Map<? extends K, ? extends V> m) {
    super();
    accessOrder = false;
    putMapEntries(m, false);
}
```
关于 LinkedHashMap 的构造函数，在其内部还是调用了父类 HashMap 的构造函数，唯一的区别是，在 LinkedHashMap 的构造函数内，对属性`accessOrder`进行了相应的赋值。


<font color=red>由于 LinkedHashMap 在实现上很多方法直接继承自HashMap，仅为维护双向链表覆写了部分方法。且HashMap的源码在另篇文章中已学习，所以，本篇文章只重点讲解 LinkedHashMap 特有的方法。 </font>

##### 元素添加

```java
//HashMap中的相应方法
final V putVal(int hash, K key, V value, boolean onlyIfAbsent, boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;

    //如果内部Node节点数组尚未初始化，则先进行初始化
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
        
    //如果带插入节点的位置上还没有值，则直接将节点插入
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);

    else {
        Node<K,V> e; K k;

        // 节点key存在，直接覆盖value
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;

        //判断该链是否为红黑树，是的话，通过数结构进行处理
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);

        //该链为链表
        else {
            for (int binCount = 0; ; ++binCount) {

                if ((e = p.next) == null) {
                    //将key-value键值对添加进Map
                    p.next = newNode(hash, key, value, null);

                    //链表长度大于8转换为红黑树进行处理
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash);
                    break;
                }

                // key已经存在直接覆盖value，跳出循环
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }

        //根据需要，判断是否需要覆盖旧值
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;

            //该方法为空实现，在LinkedHashMap有具体实现
            afterNodeAccess(e);
            return oldValue;
        }
    }
    ++modCount;

    // 超过threshold的时候就扩容。threshold = 负载因子 * 容量capacity
    if (++size > threshold)
        resize();

    //该方法为空实现，在LinkedHashMap有具体实现
    afterNodeInsertion(evict);
    return null;
}
```
该方法已在讲解 HashMap 源码的时候分析过。此处不再复述。在该方法中，有几个方法是 LinkedHashMap 独有的，我们将分析 LinkedHashMap 的相应方法。

##### newNode源码

```java
//添加新节点
Node<K,V> newNode(int hash, K key, V value, Node<K,V> e) {
    Entry<K,V> p = new Entry<K,V>(hash, key, value, e);

    //链接到双向链表的尾部
    linkNodeLast(p);
    return p;
}
```
LinkedHashMap 类覆写了 HashMap 类中的 `newNode()方法`。该方法除了组装好节点外，还调用了 `linkNodeLast()方法`，用于维护内部的双向链表结构。

##### linkNodeLast源码

```java
//将节点p设置为链表尾节点
private void linkNodeLast(Entry<K,V> p) {
    Entry<K,V> last = tail;
    tail = p;
    if (last == null)
        head = p;
    else {
        p.before = last;
        last.after = p;
    }
}
```
LinkedHashMap 创建了 Entry，并通过 linkNodeLast 方法将 Entry 接在双向链表的尾部，实现了双向链表的建立。


在 JDK 1.8 HashMap 的源码中，有3个以 `after`开头的方法

```java
// Callbacks to allow LinkedHashMap post-actions
void afterNodeAccess(Node<K,V> p) { }
void afterNodeInsertion(boolean evict) { }
void afterNodeRemoval(Node<K,V> p) { }
```
从名字也可以猜到，这三个方法都是在某种操作完成后需要调用的。在 HashMap 中全都给了空实现；但是在 LinkedHashMap 中，都给了相应的实现。从而也实现了 LinkedHashMap 顺序迭代元素的功能。

##### afterNodeAccess源码

```java
//如果accessOrder为true，则保持双向链表的访问顺序。最后访问的节点位于双向链表的尾部
void afterNodeAccess(Node<K,V> e) { // move node to last
    Entry<K,V> last;

    //不为双向链表的尾节点，因为访问尾节点不影响总体访问顺序
    if (accessOrder && (last = tail) != e) {
        Entry<K,V> p = (Entry<K,V>)e, b = p.before, a = p.after;
        p.after = null;


        //将刚刚访问过的节点从链表中去除
        if (b == null)
            head = a;
        else
            b.after = a;
        if (a != null)
            a.before = b;
        else
            last = b;

        //将去除的节点链接到链表的尾部
        if (last == null)
            head = p;
        else {
            p.before = last;
            last.after = p;
        }
        tail = p;
        ++modCount;
    }
}
```
该方法的功能就是将访问的节点重新位于双向链表的尾部。

假设我们访问下图键值为3的节点，访问前结构为：

<a data-flickr-embed="true"  href="https://www.flickr.com/photos/157389715@N05/26580411168/in/dateposted-public/" title="原数据"><img src="https://farm5.staticflickr.com/4615/26580411168_8fb5bc311f.jpg" width="500" height="237" alt="原数据"></a>

访问后，键值为3的节点将会被移动到双向链表的最后位置，其前驱和后继也会跟着更新。访问后的结构如下：

<a data-flickr-embed="true"  href="https://www.flickr.com/photos/157389715@N05/39740739264/in/dateposted-public/" title="访问后数据"><img src="https://farm5.staticflickr.com/4668/39740739264_c87a7679b1.jpg" width="500" height="233" alt="访问后数据"></a>


##### afterNodeInsertion源码

```java
void afterNodeInsertion(boolean evict) { // possibly remove eldest
    Entry<K,V> first;

    // 根据条件判断是否移除最近最少被访问的节点
    if (evict && (first = head) != null && removeEldestEntry(first)) {
        K key = first.key;
        removeNode(hash(key), key, null, false, true);
    }
}

// 移除最近最少被访问条件之一，通过覆盖此方法可实现不同策略的缓存
protected boolean removeEldestEntry(Map.Entry<K,V> eldest) {
    return false;
}
```
上面的代码做的事情比较简单，就是通过一些条件，判断是否移除最近最少被访问的节点。根据该特性，可以实现自定义策略的 LRU(最近最少被访问) 缓存。

##### 获取元素

```java
public V get(Object key) {
    Node<K,V> e;
    if ((e = getNode(hash(key), key)) == null)
        return null;
    if (accessOrder)
        afterNodeAccess(e);
    return e.value;
}
```
在访问了元素后，如果 `accessOrder为true`，则会调用 `afterNodeAccess()`进行访问顺序的调整。

##### 删除元素

```java
final Node<K,V> removeNode(int hash, Object key, Object value,
                boolean matchValue, boolean movable) {
    Node<K,V>[] tab; Node<K,V> p; int n, index;

    //查找key所对应的键值对
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (p = tab[index = (n - 1) & hash]) != null) {

        //查找对应的元素
        Node<K,V> node = null, e; K k; V v;
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            node = p;
        else if ((e = p.next) != null) {
            if (p instanceof TreeNode)
                node = ((TreeNode<K,V>)p).getTreeNode(hash, key);
            else {
                do {
                    if (e.hash == hash &&
                        ((k = e.key) == key ||
                         (key != null && key.equals(k)))) {
                        node = e;
                        break;
                    }
                    p = e;
                } while ((e = e.next) != null);
            }
        }


        //找到对应的节点后，进行移除操作
        if (node != null && (!matchValue || (v = node.value) == value ||
                             (value != null && value.equals(v)))) {

            //树结构移除元素
            if (node instanceof TreeNode)
                ((TreeNode<K,V>)node).removeTreeNode(this, tab, movable);

            //链表移除元素
            else if (node == p)
                tab[index] = node.next;
            else
                p.next = node.next;


            ++modCount;
            --size;
            afterNodeRemoval(node);
            return node;
        }
    }
    return null;
}
```
关于 `removeNode()方法`我们也已分析过，我们只要分析元素删除成功后的一些操作。在元素移除成功后，会调用`afterNodeRemoval()` 方法来调整 LinkedHashMap 内部的双向链表结构。


##### afterNodeRemoval源码

```java
//去除节点e
void afterNodeRemoval(Node<K,V> e) { // unlink
    Entry<K,V> p = (Entry<K,V>)e, b = p.before, a = p.after;

    // 将 p 节点的前驱后后继引用置空
    p.before = p.after = null;

    // b 为 null，表明 p 是头节点
    if (b == null)
        head = a;
    else
        b.after = a;

    // a 为 null，表明 p 是尾节点
    if (a == null)
        tail = b;
    else
        a.before = b;
}
```
从实现中，可以看出是将节点从链表中删除。

假如我们要删除下图键值为 3 的节点。

<a data-flickr-embed="true"  href="https://www.flickr.com/photos/157389715@N05/39740894614/in/dateposted-public/" title="待移除元数据"><img src="https://farm5.staticflickr.com/4614/39740894614_b74afb90fd.jpg" width="500" height="238" alt="待移除元数据"></a>

根据 hash 定位到该节点属于3号桶，然后在对3号桶保存的单链表进行遍历。找到要删除的节点后，先从单链表中移除该节点。如下：

<a data-flickr-embed="true"  href="https://www.flickr.com/photos/157389715@N05/39740894544/in/dateposted-public/" title="单链表移除"><img src="https://farm5.staticflickr.com/4669/39740894544_4b313d7e4a.jpg" width="500" height="121" alt="单链表移除"></a>

然后再双向链表中移除该节点：

<a data-flickr-embed="true"  href="https://www.flickr.com/photos/157389715@N05/39740896584/in/dateposted-public/" title="双链表移除"><img src="https://farm5.staticflickr.com/4662/39740896584_bb8b8e98fa.jpg" width="500" height="239" alt="双链表移除"></a>

#### 总结
在日常开发中，LinkedHashMap 的使用频率虽不及 HashMap，但它也个重要的实现。在 Java 集合框架中，HashMap、LinkedHashMap 和 TreeMap 三个映射类基于不同的数据结构，并实现了不同的功能。HashMap 底层基于拉链式的散列结构，并在 JDK 1.8 中引入红黑树优化过长链表的问题。基于这样结构，HashMap 可提供高效的增删改查操作。LinkedHashMap 在其之上，通过维护一条双向链表，实现了散列数据结构的有序遍历。TreeMap 底层基于红黑树实现，利用红黑树的性质，实现了键值对排序功能。


#### 参考文章
[LinkedHashMap 源码详细分析(JDK1.8)](https://segmentfault.com/a/1190000012964859)
