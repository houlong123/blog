#### 前言
Java为数据结构中的映射定义了一个接口java.util.Map，此接口主要有四个常用的实现类，分别是HashMap、Hashtable、LinkedHashMap和TreeMap。其中HashMap是Java程序员使用频率最高的用于映射(键值对)处理的数据类型。<font color=red>HashMap是一种无序的，且允许null作为键值对的Map集合。该类除了不是同步的，且允许null键值对外，其他都与Hashtable基本一致。</font>由于Hashtable是遗留类，且并发性不如ConcurrentHashMap，所以在新代码中基本上不使用。HashMap的底层采用 `Node数组+链表+红黑树` 的存储结构。该类除了没有并发功能外，与 [ConcurrentHashMap](https://houlong123.github.io/2017/11/07/ConcurrentHashMap源码学习/) 类基本一致。


![存储结构](http://incdn1.b0.upaiyun.com/2017/07/665f644e43731ff9db3d341da5c827e1.png)


#### 源码解析

<!-- more -->

##### 数据结构

```java
/** API
 * HashMap是一种无序的，且允许null作为键值对的Map集合。
 * 该类除了不是同步的，且允许null键值对外，其他都与Hashtable基本一致。
 * 
 * 假如通过哈希函数将元素适当的分散在桶中，则例如get，put方法的时间是固定的。
 * 遍历集合视图所需要时间与HashMap表的容量及 Map 中所包含映射的 Set 视图个数之和成正比。
 * 因此如果在应用中，遍历的功能是首要的，则Map集合的初始化容量不易过大
 *
 * HashMap的构造函数有两个重要的参数：初始化容量（initial capacity）和 负载因子（load factor）。
 * 当 （负载因子）x（容量）>（Map 大小）时，则调整 Map 大小
 */
public class HashMap<K,V> extends AbstractMap<K,V>
    implements Map<K,V>, Cloneable, Serializable {
    
    
    //HashMap的默认初始化容量大小。必须为2^N 。主要是在定位元素时用到
    static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16
    //最大容量
    static final int MAXIMUM_CAPACITY = 1 << 30;
    //默认负载因子因子。
    static final float DEFAULT_LOAD_FACTOR = 0.75f;
    //链表转树的临界值。大于8时，转为数结构
    static final int TREEIFY_THRESHOLD = 8;
    //树转链表的临界值
    static final int UNTREEIFY_THRESHOLD = 6;
     //转换为树的最小表容量
    static final int MIN_TREEIFY_CAPACITY = 64;
    
    
    //内部Node节点数组。在首次使用时进行初始化，并在必要时进行扩容。分配时，长度为2的次幂。
    transient Node<K,V>[] table;
    //Map 中所包含映射的 Set 视图
    transient Set<Entry<K,V>> entrySet;
    //Map中包含的key-value键值对的数量
    transient int size;
    //HashMap内部的数据结构被修改的次数。在使用iterators时，会使用该值
    transient int modCount;
    //下一次调整的值(capacity * load factor)。如果Map没有被分配，则此字段表示初始化数组容量。0表示DEFAULT_INITIAL_CAPACITY。
    int threshold;
    //负载因子.所能容纳的key-value对极限。threshold就是在此Load factor和length(数组长度)对应下允许的最大元素数目，超过这个数目就重新resize(扩容)
    final float loadFactor;
    
    
    //基于哈希的bin节点
    static class Node<K,V> implements Entry<K,V> {}
    
    //树节点继承了LinkedHashMap的Entry，而该Entry则继承了 HashMap的 Node
    static final class TreeNode<K,V> extends LinkedHashMap.Entry<K,V> {}
    
}
```
关于HashMap中的一些属性，我们使用图表来展示。

+ 默认属性值
    

属性名 | 默认属性值 | 含义
---|---|---
DEFAULT_INITIAL_CAPACITY | 2^4 | 在创建HashMap对象时，如果没有指定初始化值，则默认的容量大小为该值
MAXIMUM_CAPACITY | 2^30 | 表示HashMap对象所能包含的最大容量值
DEFAULT_LOAD_FACTOR | 0.75f | 决定哈希表在达到什么程度时可以扩容。当（负载因子）x（容量）>（Map 大小）时，则调整 Map 大小
TREEIFY_THRESHOLD | 8 |链表转树的阈值。大于等于8时，转为树结构
UNTREEIFY_THRESHOLD | 6 | 树转链表的阈值。小于等于6时，转为链表结构
MIN_TREEIFY_CAPACITY | 64 | 转换为树结构的最小表容量。当表容量小于该值时，扩容；否则，将对于索引上的链表转为树结构

+ 关键属性

属性名 | 含义
---|---
Node<K,V>[] table | 内部Node节点数组。用于存储键值对。在首次使用时进行初始化，并在必要时进行扩容。分配时，长度为2的次幂。
Set<Entry<K,V>> entrySet | Map 中所包含映射的 Set 视图
 size | HashMap中实际存在的键值对数量
 modCount | 记录HashMap内部结构发生变化的次数，主要用于迭代的快速失败。
threshold | HashMap所能容纳的最大数据量的Node(键值对)个数。threshold = length * Load factor。如果Map没有被分配，则此字段表示初始化数组容量。否则当 `size > threshold` 时，进行扩容。
loadFactor | 负载因子。


###### Node节点

```java
//基于哈希的bin节点
static class Node<K,V> implements Entry<K,V> {
    final int hash;
    //key是不可变对象。不可变对象是该对象在创建后它的哈希值不会被改变。
    final K key;

    //与ConcurrentHashMap不同之处在于：ConcurrentHashMap中的value和next都是volatile的。
    V value;
    Node<K,V> next;

    Node(int hash, K key, V value, Node<K,V> next) {
        this.hash = hash;
        this.key = key;
        this.value = value;
        this.next = next;
    }

    public final K getKey()        { return key; }
    public final V getValue()      { return value; }
    public final String toString() { return key + "=" + value; }

    /**
     * 与ConcurrentHashMap类中的hashCode()方法实现不同的原因是：HashMap中允许null值存在，而ConcurrentHashMap不允许
     */
    public final int hashCode() {
        return Objects.hashCode(key) ^ Objects.hashCode(value);
    }

    //在ConcurrentHashMap类中，setValue() 方法抛异常。不允许直接改变value的值
    public final V setValue(V newValue) {
        V oldValue = value;
        value = newValue;
        return oldValue;
    }

    public final boolean equals(Object o) {
        if (o == this)
            return true;
        if (o instanceof Map.Entry) {
            Entry<?,?> e = (Entry<?,?>)o;
            if (Objects.equals(key, e.getKey()) &&
                Objects.equals(value, e.getValue()))
                return true;
        }
        return false;
    }
}
```
Node是HashMap的一个内部类，实现了Map.Entry接口，本质是就是一个映射(键值对)。可以将该类与ConcurrentHashMap中的Node节点类进行对比。


##### 构造函数

```java
//指定容量大小，但是负载因子使用默认的
public HashMap(int initialCapacity) {
    this(initialCapacity, DEFAULT_LOAD_FACTOR);
}

//默认构造函数。其中负载因子和容量大小都使用默认的
public HashMap() {
    this.loadFactor = DEFAULT_LOAD_FACTOR;
}

//指定容量大小和负载因子进行初始化
public HashMap(int initialCapacity, float loadFactor) {
    //如果初始化容量小于0，抛异常
    if (initialCapacity < 0)
        throw new IllegalArgumentException("Illegal initial capacity: " +
                                           initialCapacity);

    //如果初始化容量大于最大容量MAXIMUM_CAPACITY，则重新赋值为MAXIMUM_CAPACITY
    if (initialCapacity > MAXIMUM_CAPACITY)
        initialCapacity = MAXIMUM_CAPACITY;

    //如果负载因子非法，抛异常
    if (loadFactor <= 0 || Float.isNaN(loadFactor))
        throw new IllegalArgumentException("Illegal load factor: " +
                                           loadFactor);
    //设置HashMap的负载因子和初始化容量
    this.loadFactor = loadFactor;

    //所说在创建HashMap时，给定了初始化容量大小为initialCapacity，
    // 但是最终的容量并不一定是所指定的initialCapacity，而且一个不小于initialCapacity的最少2次幂
    this.threshold = tableSizeFor(initialCapacity);
}

//使用另个Map集合进行初始化
public HashMap(Map<? extends K, ? extends V> m) {
    this.loadFactor = DEFAULT_LOAD_FACTOR;
    putMapEntries(m, false);
}
```
HashMap类提供了四个构造函数。但是其内部都是对其两个属性：`loadFactor` 和 `threshold` 进行赋值。在确认`threshold`时，使用了 `tableSizeFor`方法，该方法是获取不小于参数的最小2的次幂。具体实现，在下面给出源码。所以从这里可知，在只用指定容量对HashMap进行初始化时，HashMap的实际容量不一定就是你所指定的值。

##### tableSizeFor源码

```java
//返回大于指定值cap的最少的2次幂。
static final int tableSizeFor(int cap) {
    int n = cap - 1;
    n |= n >>> 1;
    n |= n >>> 2;
    n |= n >>> 4;
    n |= n >>> 8;
    n |= n >>> 16;
    return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
}
```
该方法是获取大于指定值cap的最少的2次幂。下图为该方法执行的结果：


cap | tableSizeFor(cap) 的结果 | cap | tableSizeFor(cap) 的结果 |
---|---|---|---  
0 | 0 |5 | 8 |
1 | 1 |6 | 8 |
2 | 2 |7 | 8 |
3 | 4 |8 | 8 |
4 | 4 |9 | 16 |


##### put源码

```java
public V put(K key, V value) {
    //计算出key的哈希值后，直接调用putVal方法将键值对添加进Map集合
    return putVal(hash(key), key, value, false, true);
}
```
在该方法的内部，首先通过`hash`函数获取键值key的哈希值，然后调用 `putVal`方法添加一个键值对。

##### hash源码

```java
static final int hash(Object key) {
    int h;
    //返回key对像哈希值的高16位与低16位相异或的值
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```
在JDK1.8的实现中，优化了高位运算的算法，通过hashCode()的高16位异或低16位实现的。这么做可以在数组table的length比较小的时候，也能保证考虑到高低Bit都参与到Hash的计算中，同时不会有太大的开销。


##### putVal源码

```java
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
该方法是HashMap的核心方法。执行过程可以通过下图来理解：

![put方法](https://tech.meituan.com/img/java-hashmap/hashMap%20put%E6%96%B9%E6%B3%95%E6%89%A7%E8%A1%8C%E6%B5%81%E7%A8%8B%E5%9B%BE.png)

代码逻辑大致为：
1. 判断键值对数组table是否初始化，否的话，执行resize()进行初始化
2. 根据键值key计算hash值得到插入的数组索引i，如果table[i]==null，直接新建节点添加，添加成功后，执行7步；否则，执行第3步。
3. 判断table[i]的首个元素是否和key一样，如果相同获取对应的 Node节点，然后执行第6步。（通过hashCode和equals()方法判断是否相同）；否则，执行第4步。
4. 判断table[i] 是否为树结构，即table[i] 是否是红黑树，如果是红黑树，则直接在树中插入键值对，否则转向第5步
5. 遍历table[i]位置处的链表。如果遍历过程中发现key已经存在则获取对应的 Node 节点，返回跳出循环，执行第6步；否则，将key-value键值组装成Node节点，然后添加到链表的最尾部。添加成功后，判断链表长度是否大于8，大于8的话把链表转换为红黑树。转换完成后，执行第6步。
6. 如果有需要的话，覆盖key对应节点value的值。
7. 判断实际存在的键值对数量size是否超多了最大容量threshold，如果超过，进行扩容。


##### resize源码

```java
final Node<K,V>[] resize() {
    //获取旧数组
    Node<K,V>[] oldTab = table;

    //获取旧数组的大小
    int oldCap = (oldTab == null) ? 0 : oldTab.length;


    int oldThr = threshold;
    int newCap, newThr = 0;


    /**
     * 下面一段语句的功能是：确认HashMap的本次扩容的大小（capacity）及下次扩容的大小（threshold）
     *  1. 如果内部Node节点数组尚未初始化，则capacity的值为创建HashMap时通过指定的容量大小计算的threshold（threshold=0的话，capacity = 16）；新的threshold = threshold * 负载因子
     *
     *  2. 如果内部Node节点数组已初始化，则capacity为threshold，新的threshold = threshold * 负载因子
     */
    //oldCap > 0,说明是进行数组扩容
    if (oldCap > 0) {

        // 超过最大值就不再扩充了，就只好随你碰撞去吧
        if (oldCap >= MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;  //修改阈值为int的最大值(2^31-1)，这样以后就不会扩容了
            return oldTab;
        }

        // 没超过最大值，就扩充为原来的2倍
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY && oldCap >= DEFAULT_INITIAL_CAPACITY)
            newThr = oldThr << 1; // double threshold

        //在创建HashMap时，指定了initialCapacity且大于0
    } else if (oldThr > 0) // initial capacity was placed in threshold
        newCap = oldThr;

        //在创建HashMap时，没有指定initialCapacity 或者 initialCapacity=0
    else {               // zero initial threshold signifies using defaults
        newCap = DEFAULT_INITIAL_CAPACITY;
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }


    //在创建HashMap时，指定的容量大小大于0时，newThr会为0
    if (newThr == 0) {
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                  (int)ft : Integer.MAX_VALUE);
    }
    threshold = newThr;


    //使用新容量newCap 初始化Node新数组
    @SuppressWarnings({"rawtypes","unchecked"})
    Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    table = newTab;

    //旧数组不为空时，遍历
    if (oldTab != null) {
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e;
            if ((e = oldTab[j]) != null) {  //获取索引j处的Node节点

                //将索引j处的Node节点置空
                oldTab[j] = null;


                if (e.next == null) //如果索引j处的Node节点没有next节点，则直接将该节点放入新Node数组中
                    newTab[e.hash & (newCap - 1)] = e;

                //如果索引j处的Node节点e有next节点，且e为树结构，则使用数的相关方法实现将节点放入新Node数组中
                else if (e instanceof TreeNode)
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);

                //如果索引j处的Node节点e有next节点，且为链表结构
                else { // preserve order

                    //保持原来元素的顺序不变
                    Node<K,V> loHead = null, loTail = null;
                    Node<K,V> hiHead = null, hiTail = null;
                    Node<K,V> next;
                    do {
                        next = e.next;

                        /**
                         * 因为新容量都是扩为原来2倍。所以，元素的新位置要么是在原位置，要么是在原位置的基础上再移动原容量的位置。
                         * 分析：
                         * 因为新容量都是扩为原来2倍，所以对应的其二进制就是将1的位置往前移动1位。在计算元素的插入索引index = e.hash & (newCap - 1)时，可知，
                         * newCap - 1 的二进制将会在高1位上多一个1。在进行索引的时候，可知，e.hash & (newCap - 1)要么为原索引位置；要么为原索引位置在移动原容量大小
                         *
                         *  实例： 前提：原容量大小 n = 16, key1的哈希值的二进制为：1111 1111 1111 1111 0000 1111 0000 0101   key2的哈希值的二进制为：111 1111 1111 1111 0000 1111 0001 0101
                         *  则原位置：
                         *  index1 = 1111 1111 1111 1111 0000 1111 0000 0101 & 0000 0000 0000 0000 0000 0000 0000 1111 =  0000 0000 0000 0000 0000 0000 0000 0101
                         *  index2 = 111 1111 1111 1111 0000 1111 0001 0101  & 0000 0000 0000 0000 0000 0000 0000 1111 =  0000 0000 0000 0000 0000 0000 0000 0101
                         *
                         *  将容量扩容，则此时 n = 32
                         *  则新位置：
                         *  new_index1 = 1111 1111 1111 1111 0000 1111 0000 0101 & 0000 0000 0000 0000 0000 0000 0001 1111 = 0000 0000 0000 0000 0000 0000 0000 0101
                         *  new_index2 = 111 1111 1111 1111 0000 1111 0001 0101  & 0000 0000 0000 0000 0000 0000 0001 1111 = 0000 0000 0000 0000 0000 0000 0001 0101
                         *
                         *  可以发现，new_index1 = index1，扩容后，key1对于的位置没有发生变化；但是 new_index2 = index2 + 16，而16就是原容量的大小。
                         */
                        // 原索引
                        if ((e.hash & oldCap) == 0) {
                            if (loTail == null)
                                loHead = e;
                            else
                                loTail.next = e;
                            loTail = e;
                        }

                        // 原索引+oldCap
                        else {
                            if (hiTail == null)
                                hiHead = e;
                            else
                                hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null);

                    // 原索引放到bucket里
                    if (loTail != null) {
                        loTail.next = null;
                        newTab[j] = loHead;
                    }

                    // 原索引+oldCap放到bucket里
                    if (hiTail != null) {
                        hiTail.next = null;
                        newTab[j + oldCap] = hiHead;
                    }
                }
            }
        }
    }
    return newTab;
}
```
该方法的功能是：对HashMap对象进行扩容。JDK1.8 中的扩容实现逻辑与JDK1.7的实现逻辑有所不同。本文是针对JDK1.8的源码进行分析。 中的相关讲解。再次简单介绍大体流程：
1. 确认HashMap的本次扩容的大小（capacity）及下次扩容的大小（threshold）
2. 创建新容量数组table，然后修改内部属性table的值
3. 判断原键值对数组table是否为空。为空，直接返回新数组；否则，往下执行。
4. 遍历原键值对数组table。
    1. 如果原键值对数组table的索引j处链表不空，赋值给e变量，然后往下执行。
    2. 如果变量e的next域为空，则直接将变量e添加进新数组的指定索引处。否则，执行下一步
    3. 如果变量e为树结构，则通过树操作将树结构中的节点添加到新数组
    4. 如果变量e为链表结构，则遍历链表结构，将各个元素添加到新数组。
    

<font color=red>备注：关于将链表上的节点重新复制到新数组的实现逻辑，JDK1.8中的实现可谓是非常巧妙，跪服了😂😂😂，具体讲解可以详看 [Java 8系列之重新认识HashMap](https://tech.meituan.com/java-hashmap.html) </font>。下面是引用该文章的内容讲解。


```
下面我们讲解下JDK1.8做了哪些优化。经过观测可以发现，我们使用的是2次幂的
扩展(指长度扩为原来2倍)，所以，元素的位置要么是在原位置，要么是在原位置
再移动2次幂的位置。看下图可以明白这句话的意思，n为table的长度，
图（a）表示扩容前的key1和key2两种key确定索引位置的示例，
图（b）表示扩容后key1和key2两种key确定索引位置的示例，
其中hash1是key1对应的哈希与高位运算结果。
```

![image](https://tech.meituan.com/img/java-hashmap/hashMap%201.8%20%E5%93%88%E5%B8%8C%E7%AE%97%E6%B3%95%E4%BE%8B%E5%9B%BE1.png)


```
元素在重新计算hash之后，因为n变为2倍，那么n-1的mask范围在高位多1bit(红色)，
因此新的index就会发生这样的变化：
```

![image](https://tech.meituan.com/img/java-hashmap/hashMap%201.8%20%E5%93%88%E5%B8%8C%E7%AE%97%E6%B3%95%E4%BE%8B%E5%9B%BE2.png)


```
因此，我们在扩充HashMap的时候，不需要像JDK1.7的实现那样重新计算hash，只需要看看原来的hash值
新增的那个bit是1还是0就好了，是0的话索引没变，是1的话索引变成“原索引+oldCap”，
可以看看下图为16扩充为32的resize示意图：
```

![image](https://tech.meituan.com/img/java-hashmap/jdk1.8%20hashMap%E6%89%A9%E5%AE%B9%E4%BE%8B%E5%9B%BE.png)

<font color=red>注意区别，JDK1.7中rehash的时候，旧链表迁移新链表的时候，如果在新表的数组索引位置相同，则链表元素会倒置，但是JDK1.8不会倒置。</font>

##### split源码
在对HashMap内部table进行扩展的时候，如果原来索引处的元素为树结构，则会调用`split`方法对结构进行调整。

```java
final void split(HashMap<K,V> map, Node<K,V>[] tab, int index, int bit) {

    //this指的是Map中索引为index处的树结构
    TreeNode<K,V> b = this;
    // Relink into lo and hi lists, preserving order
    TreeNode<K,V> loHead = null, loTail = null;
    TreeNode<K,V> hiHead = null, hiTail = null;
    int lc = 0, hc = 0;

    //遍历树结构
    for (TreeNode<K,V> e = b, next; e != null; e = next) {

        //数节点的next节点
        next = (TreeNode<K,V>)e.next;
        e.next = null;

        //bit为原数组的容量。其处理逻辑与链表结构的一致
        if ((e.hash & bit) == 0) {
            if ((e.prev = loTail) == null)
                loHead = e;
            else
                loTail.next = e;
            loTail = e;
            ++lc;
        }
        else {
            if ((e.prev = hiTail) == null)
                hiHead = e;
            else
                hiTail.next = e;
            hiTail = e;
            ++hc;
        }
    }

    // 原索引放到bucket里
    if (loHead != null) {

        //如果树结构的节点数小于等于6，则将树结构转换为链表结构
        if (lc <= UNTREEIFY_THRESHOLD)
            tab[index] = loHead.untreeify(map);
        else {
            tab[index] = loHead;
            if (hiHead != null) // (else is already treeified)
                loHead.treeify(tab);
        }
    }

    // 原索引+oldCap放到bucket里
    if (hiHead != null) {
        if (hc <= UNTREEIFY_THRESHOLD)
            tab[index + bit] = hiHead.untreeify(map);
        else {
            tab[index + bit] = hiHead;
            if (loHead != null)
                hiHead.treeify(tab);
        }
    }
}
```
其中逻辑与链表的大体一致。是把原来位置处的一个树结构拆分为两个树结构。在拆分成功后，会判断树节点的个数是否小于等于6，如果是的话，则会调用 `untreeify`方法将树结构转为链表结构。

##### untreeify源码

```java
final Node<K,V> untreeify(HashMap<K,V> map) {
    Node<K,V> hd = null, tl = null;
    for (Node<K,V> q = this; q != null; q = q.next) {
        Node<K,V> p = map.replacementNode(q, null);
        if (tl == null)
            hd = p;
        else
            tl.next = p;
        tl = p;
    }
    return hd;
}
```
该方法是：<font color=red>在树结构中节点的个数不大于6的时候，将树结构转为链表结构</font>。


##### treeifyBin源码

```java
final void treeifyBin(Node<K,V>[] tab, int hash) {
    int n, index; Node<K,V> e;

    //在Map的容量大小小于64（MIN_TREEIFY_CAPACITY）的时候，只简单进行扩容
    if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY)
        resize();

    //否则，将指定索引处的链表转换为树结构
    else if ((e = tab[index = (n - 1) & hash]) != null) {
        TreeNode<K,V> hd = null, tl = null;

        //下面的do-while语句的功能是：遍历链表结构，依次把链表中的每个节点转换为树结构，然后组装成树
        do {
            //将节点结构转换为树结构
            TreeNode<K,V> p = replacementTreeNode(e, null);
            if (tl == null)
                hd = p;
            else {
                p.prev = tl;
                tl.next = p;
            }
            tl = p;
        } while ((e = e.next) != null);

        //转换为树结构
        if ((tab[index] = hd) != null)
            hd.treeify(tab);
    }
}
```
在成功添加后，如果指定位置上的链表节点个数不小于8的时候，则要将链表结构转换为树结构。其中基本逻辑是：将Node类型的节点转换为TreeNode类型的节点，然后在通过TreeNode 类中的 `treeify` 方法实现树形转换的功能。


##### treeify源码

```java
//树化
final void treeify(Node<K,V>[] tab) {
    TreeNode<K,V> root = null;

    //遍历组装的TreeNode节点链表
    for (TreeNode<K,V> x = this, next; x != null; x = next) {
        next = (TreeNode<K,V>)x.next;
        x.left = x.right = null;

        //设置树的根节点
        if (root == null) {
            x.parent = null;
            x.red = false;
            root = x;
        }
        else {
            K k = x.key;
            int h = x.hash;
            Class<?> kc = null;

            //组装树结构
            for (TreeNode<K,V> p = root;;) {
                int dir, ph;
                K pk = p.key;

                //确定节点x是节点p的左节点还是右节点。dir<0，则为左节点；否则为右节点。
                // 在树结构中，左节点的key对应的哈希值小于其父节点的key对应的哈希值。
                if ((ph = p.hash) > h)
                    dir = -1;
                else if (ph < h)
                    dir = 1;

                //在节点x的key对应的哈希值与节点p相同时，则通过比较器比较key
                else if ((kc == null &&(kc = comparableClassFor(k)) == null) ||   // comparableClassFor方法是获取节点x的key的比较器（如果key继承了Comparable，否则返回null）
                         (dir = compareComparables(kc, k, pk)) == 0)

                    //在节点x的key没有继承Comparable接口  或者  通过比较器比较节点x的key与节点p的key相同
                    dir = tieBreakOrder(k, pk);     //该方法先通过类名比较，如果类名相同，在通过哈希比较


                //如果节点p的左右子节点不为空，则继续往深层遍历，直至将节点x插入到叶子节点
                TreeNode<K,V> xp = p;
                if ((p = (dir <= 0) ? p.left : p.right) == null) {
                    x.parent = xp;
                    if (dir <= 0)
                        xp.left = x;
                    else
                        xp.right = x;

                    //平衡树
                    root = balanceInsertion(root, x);
                    break;
                }
            }
        }
    }
    moveRootToFront(tab, root);
}
```
该方法的功能就是将链表结构转换为树结构。整体逻辑就是：遍历链表结构上的所有节点，然后进行组装。关于数结构的相关操作，本文不做讲述，可以看下源码中的注释。关于树结构可以[点击学习](http://blog.csdn.net/v_july_v/article/details/6105630)。

---

##### get源码

```java
public V get(Object key) {
    Node<K,V> e;
    return (e = getNode(hash(key), key)) == null ? null : e.value;
}
```
在调用该方法获取指定键值key对应的value值时，首先获取键值key的哈希值，然后通过`getNode`方法获取对应的Node节点。如果该节点存在，则返回节点的value值；否则，说明键值key对应的节点不存在，返回null。

---

##### getNode源码

```java
final Node<K,V> getNode(int hash, Object key) {
    Node<K,V>[] tab; Node<K,V> first, e; int n; K k;

    //在Map中hash值所对应的索引上有节点
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (first = tab[(n - 1) & hash]) != null) {

        //第一个节点就是键为key的节点，则直接返回节点
        if (first.hash == hash && // always check first node
            ((k = first.key) == key || (key != null && key.equals(k))))
            return first;

        //遍历节点查找
        if ((e = first.next) != null) {

            //树结构查找
            if (first instanceof TreeNode)
                return ((TreeNode<K,V>)first).getTreeNode(hash, key);

            //链表方式查找
            do {
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    return e;
            } while ((e = e.next) != null);
        }
    }
    return null;
}
```
该方法是获取键值key对应的Node节点。实现逻辑为：
1. 判断HashMap是否存有键值对，如果有数据且对应的位置 `(n - 1) & hash`（即键值key的哈希值所在的位置） 上有存储节点，赋值 `first节点 = tab[(n - 1) & hash]` ，则执行下一步；否则返回null，方法结束。
2. 如果first节点与key一样，则直接返回first节点。
3. 如果first节点为树结构，则通过树的相关查找逻辑查找，返回返回
4. 否则通过遍历链表节点，进行查找。找到返回指定节点，找不到，返回null。


---

##### getTreeNode节点
在为树结构的时候，通过该方法进行查找指定值

```java
final TreeNode<K,V> getTreeNode(int h, Object k) {
    //如果当前节点不是树的根节点，则通过root()方法找到根节点，然后从根节点开始，调用find方法进行查找
    return ((parent != null) ? root() : this).find(h, k, null);
}
```
该方法的功能是：从树结构的根节点开始查找元素。如果当前节点不是树的根节点，则首先需要找到树的根节点，然后在进行查找。

##### root源码

```java
final TreeNode<K,V> root() {
    for (TreeNode<K,V> r = this, p;;) {
        if ((p = r.parent) == null)
            return r;
        r = p;
    }
}
```
查找树结构的根节点的条件是：根节点的parent属性为null。所以只需要从当前节点开始向上查找其父节点，如果父节点为null，说明该节点为树结构的根节点。

##### find源码

```java
//从根节点开始进行查询
final TreeNode<K,V> find(int h, Object k, Class<?> kc) {
    TreeNode<K,V> p = this;

    /**
     * 在HashMap对象转为树结构的时候，我们知道，父节点key的哈希值是大于左节点key的哈希值，并不大于右节点的key的哈希值
     * 所以在查找的时候，就类似于二分法查找
     */
    do {
        int ph, dir; K pk;
        TreeNode<K,V> pl = p.left, pr = p.right, q;
        if ((ph = p.hash) > h)
            p = pl;
        else if (ph < h)
            p = pr;
        else if ((pk = p.key) == k || (k != null && k.equals(pk)))
            return p;
        else if (pl == null)
            p = pr;
        else if (pr == null)
            p = pl;
        else if ((kc != null ||
                  (kc = comparableClassFor(k)) != null) &&
                 (dir = compareComparables(kc, k, pk)) != 0)
            p = (dir < 0) ? pl : pr;
        else if ((q = pr.find(h, k, kc)) != null)
            return q;
        else
            p = pl;
    } while (p != null);
    return null;
}
```
该方法就是对树结构进行查找操作。


##### remove源码

```java
public V remove(Object key) {
    Node<K,V> e;
    return (e = removeNode(hash(key), key, null, false, true)) == null ?
        null : e.value;
}
```
在计算到键值key对应的哈希值后，调用`removeNode`方法进行移除。


##### removeNode源码

```java
final Node<K,V> removeNode(int hash, Object key, Object value, 
    boolean matchValue, boolean movable) {
    
    Node<K,V>[] tab; Node<K,V> p; int n, index;

    if ((tab = table) != null && (n = tab.length) > 0 &&
        (p = tab[index = (n - 1) & hash]) != null) {

        //查找key所对应的键值对
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
在移除节点的时候，首先要找到键值key所对应的Node节点。其内部查找Node节点的代码逻辑与 `getNode`方法一致。在找到对应的Node节点后，会进行移除操作。如果为树结构，则通过 `removeTreeNode` 方法进行树移除操作；否则进行节点的相关移除操作。


##### clear源码

```java
public void clear() {
    Node<K,V>[] tab;
    modCount++;
    //将内部Node节点全部置为null
    if ((tab = table) != null && size > 0) {
        size = 0;
        for (int i = 0; i < tab.length; ++i)
            tab[i] = null;
    }
}
```

##### containsValue源码

```java
public boolean containsValue(Object value) {
    Node<K,V>[] tab; V v;

    //遍历内部Node节点数组
    if ((tab = table) != null && size > 0) {
        for (int i = 0; i < tab.length; ++i) {

            //遍历索引i处的链表，通过next索引遍历
            for (Node<K,V> e = tab[i]; e != null; e = e.next) {
                if ((v = e.value) == value ||
                    (value != null && value.equals(v)))
                    return true;
            }
        }
    }
    return false;
}
```
由源码可知，在查找指定的value值是否存在时，会整体扫描内部的数据结构，此操作的效率明显是非常低的。


##### containsKey源码

```java
public boolean containsKey(Object key) {
    return getNode(hash(key), key) != null;
}
```
在根据键值key进行查找时，内部会调用 `getNode`查找，该在上面已分析过，可知该方法比 `containsValue` 方法效率高的多。

--- 

#### 视图展示

##### keySet源码

```java
public Set<K> keySet() {
    Set<K> ks;
    return (ks = keySet) == null ? (keySet = new KeySet()) : ks;
}

//键值key的视图类
final class KeySet extends AbstractSet<K> {
    public final int size()                 { return size; }
    public final void clear()               { HashMap.this.clear(); }
    public final Iterator<K> iterator()     { return new KeyIterator(); }
    public final boolean contains(Object o) { return containsKey(o); }
    public final boolean remove(Object key) {
        return removeNode(hash(key), key, null, false, true) != null;
    }
    public final Spliterator<K> spliterator() {
        return new KeySpliterator<>(HashMap.this, 0, -1, 0, 0);
    }
    public final void forEach(Consumer<? super K> action) {
        Node<K,V>[] tab;
        if (action == null)
            throw new NullPointerException();
        if (size > 0 && (tab = table) != null) {
            int mc = modCount;
            for (int i = 0; i < tab.length; ++i) {
                for (Node<K,V> e = tab[i]; e != null; e = e.next)
                    action.accept(e.key);
            }
            if (modCount != mc)
                throw new ConcurrentModificationException();
        }
    }
}

//遍历器
final class KeyIterator extends HashIterator
    implements Iterator<K> {
    public final K next() { return nextNode().key; }
}
```

---

##### values源码

```java
public Collection<V> values() {
    Collection<V> vs;
    return (vs = values) == null ? (values = new Values()) : vs;
}

//value的视图类
final class Values extends AbstractCollection<V> {
    public final int size()                 { return size; }
    public final void clear()               { HashMap.this.clear(); }
    public final Iterator<V> iterator()     { return new ValueIterator(); }
    public final boolean contains(Object o) { return containsValue(o); }
    public final Spliterator<V> spliterator() {
        return new ValueSpliterator<>(HashMap.this, 0, -1, 0, 0);
    }
    public final void forEach(Consumer<? super V> action) {
        Node<K,V>[] tab;
        if (action == null)
            throw new NullPointerException();
        if (size > 0 && (tab = table) != null) {
            int mc = modCount;
            for (int i = 0; i < tab.length; ++i) {
                for (Node<K,V> e = tab[i]; e != null; e = e.next)
                    action.accept(e.value);
            }
            if (modCount != mc)
                throw new ConcurrentModificationException();
        }
    }
}

//迭代器
final class ValueIterator extends HashIterator
    implements Iterator<V> {
    public final V next() { return nextNode().value; }
}
```

---

##### entrySet源码

```java
public Set<Entry<K,V>> entrySet() {
    Set<Entry<K,V>> es;
    return (es = entrySet) == null ? (entrySet = new EntrySet()) : es;
}

//key-value键值对视图
final class EntrySet extends AbstractSet<Entry<K,V>> {
    public final int size()                 { return size; }
    public final void clear()               { HashMap.this.clear(); }
    public final Iterator<Entry<K,V>> iterator() {
        return new EntryIterator();
    }
    public final boolean contains(Object o) {
        if (!(o instanceof Map.Entry))
            return false;
        Entry<?,?> e = (Entry<?,?>) o;
        Object key = e.getKey();
        Node<K,V> candidate = getNode(hash(key), key);
        return candidate != null && candidate.equals(e);
    }
    public final boolean remove(Object o) {
        if (o instanceof Map.Entry) {
            Entry<?,?> e = (Entry<?,?>) o;
            Object key = e.getKey();
            Object value = e.getValue();
            return removeNode(hash(key), key, value, true, true) != null;
        }
        return false;
    }
    public final Spliterator<Entry<K,V>> spliterator() {
        return new EntrySpliterator<>(HashMap.this, 0, -1, 0, 0);
    }
    public final void forEach(Consumer<? super Entry<K,V>> action) {
        Node<K,V>[] tab;
        if (action == null)
            throw new NullPointerException();
        if (size > 0 && (tab = table) != null) {
            int mc = modCount;
            for (int i = 0; i < tab.length; ++i) {
                for (Node<K,V> e = tab[i]; e != null; e = e.next)
                    action.accept(e);
            }
            if (modCount != mc)
                throw new ConcurrentModificationException();
        }
    }
}

//迭代器
final class EntryIterator extends HashIterator
    implements Iterator<Entry<K,V>> {
    public final Entry<K,V> next() { return nextNode(); }
}
```

从上面的视图相关代码中，可知无论是哪个迭代器，其内部在遍历的时候都会调用 `nextNode`方法获取下个节点，然后返回指定值。

好了，HashMap相关学习到此结束，总之，跪服了😂😂。多亏了[Java 8系列之重新认识HashMap](https://tech.meituan.com/java-hashmap.html)这个文章，不然JDK1.8的优化，不会了解的这么详细😂😂😂



#### 参考文章
[Java 8系列之重新认识HashMap](https://tech.meituan.com/java-hashmap.html)

[Java Map 集合类简介 ](http://www.oracle.com/technetwork/cn/articles/maps1-100947-zhs.html)