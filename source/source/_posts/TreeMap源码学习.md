---
title: TreeMap源码学习
date: 2018-02-26 18:06:58
tags: java基础集合
---

#### 前言
TreeMap 是 Java集合框架中比较重要一个的实现。TreeMap 底层基于<font color=red>红黑树</font>实现，可保证在log(n)时间复杂度内完成 containsKey、get、put 和 remove 操作，效率很高。另一方面，由于TreeMap 基于红黑树实现，这为 TreeMap 保持键的有序性打下了基础。

由于 TreeMap 底层基于 红黑树 实现，因此，在学习 TreeMap 源码之前，我们需要先学习一下[红黑树](https://zh.wikipedia.org/wiki/%E7%BA%A2%E9%BB%91%E6%A0%91)的基础知识。

#### 红黑树知识学习
##### 简介
红黑树是一种自平衡的二叉查找树，是一种高效的查找树。
<!-- more -->
##### 性质
红黑树是每个节点都带有颜色属性的二叉查找树，颜色为红色或黑色。在二叉查找树强制一般要求以外，对于任何有效的红黑树我们增加了如下的额外要求：
1. 节点是红色或黑色。
2. 根是黑色。
3. 所有叶子都是黑色（叶子是NIL节点）。
4. 每个红色节点必须有两个黑色的子节点。（从每个叶子到根的所有路径上不能有两个连续的红色节点。）
5. 从任一节点到其每个叶子的所有简单路径都包含相同数目的黑色节点。

![image](https://upload.wikimedia.org/wikipedia/commons/thumb/6/66/Red-black_tree_example.svg/900px-Red-black_tree_example.svg.png)

##### 旋转
在修改二叉树的结构后，会涉及到对树结构的调整，此时会有旋转操作。旋转操作分为左旋和右旋，左旋是将某个节点旋转为其右孩子的左孩子，而右旋是节点旋转为其左孩子的右孩子。

1. 左旋

    ![左旋](https://upload-images.jianshu.io/upload_images/1637925-fbb109dbaa9b2c48.gif?imageMogr2/auto-orient/strip%7CimageView2/2/w/329)

2. 右旋

    ![image](https://upload-images.jianshu.io/upload_images/1637925-185dea52a871f85e.gif?imageMogr2/auto-orient/strip%7CimageView2/2/w/301)

##### 遍历
二叉树的遍历，简单可以划分为：
1. 深度优先
2. 广度优先

其中，深度优先根据根节点相对于左右子节点的访问先后来划分为：
1. 先序遍历：指先访问根，然后访问子树的遍历方式

    ![先序遍历](https://upload.wikimedia.org/wikipedia/commons/8/8a/%E5%89%8D%E5%BA%8F%E9%81%8D%E5%8E%86.png)

2. 中序遍历：指先访问左（右）子树，然后访问根，最后访问右（左）子树的遍历方式

    ![中序遍历](https://upload.wikimedia.org/wikipedia/commons/c/c4/%E4%B8%AD%E5%BA%8F%E9%81%8D%E5%8E%86.png)

3. 后序遍历：指先访问子树，然后访问根的遍历方式

    ![后序遍历](https://upload.wikimedia.org/wikipedia/commons/7/7f/%E5%90%8E%E5%BA%8F%E9%81%8D%E5%8E%86.png)

关于二叉树的插入，删除等操作，由于太过复杂，本文不做讲述，可以通过下文参考文章进行学习。


#### 源码学习
##### 数据结构

```java
public class TreeMap<K,V>
    extends AbstractMap<K,V>
    implements NavigableMap<K,V>, Cloneable, Serializable {
    //维持TreeMap内部元素顺序的比较器。在为null的时候，使用键自身的自然顺序。
    private final Comparator<? super K> comparator;

    private transient Entry<K,V> root;
    private transient int size = 0;
    private transient int modCount = 0;
}
```

![继承图](http://t.cn/RQ0ic37)


从 TreeMap 类的声明中，可知该类继承了 `AbstractMap抽象类，并实现了NavigableMap 接口`。且内部有个名为 `comparator` 的属性，用于在插入元素时进行元素排序使用。由于底层为红黑树，因此有个 `root` 属性用于表示树根。

NavigableMap 接口声明了一系列具有导航功能的方法。

```java
public interface NavigableMap<K,V> extends SortedMap<K,V> {
    //返回键值小于参数key的最大键值对
    Entry<K,V> lowerEntry(K key);
    
    //返回键值小于参数key的最大键值
    K lowerKey(K key);
    
    //返回键值不大于参数key的最大键值对
    Entry<K,V> floorEntry(K key);
    
    //返回键值不大于参数key的最大键值
    K floorKey(K key);
    
    //返回键值不小于参数key的最小键值对
    Entry<K,V> ceilingEntry(K key);
    
    ...
}
```

##### 构造函数

```java
//创建一个新的空树形键值对映射集合。使用键的自然顺序进行排序。由此构造函数创建的映射集合中存储的元素必须实现Comparable接口。
public TreeMap() {
    comparator = null;
}

//使用指定比较器创建一个新的空树形键值对映射集合。
public TreeMap(Comparator<? super K> comparator) {
    this.comparator = comparator;
}

public TreeMap(Map<? extends K, ? extends V> m) {
    comparator = null;
    putAll(m);
}

public TreeMap(SortedMap<K, ? extends V> m) {
    comparator = m.comparator();
    try {
        buildFromSorted(m.size(), m.entrySet().iterator(), null, null);
    } catch (java.io.IOException cannotHappen) {
    } catch (ClassNotFoundException cannotHappen) {
    }
}
```
TreeMap 的构造函数主要功能是设置元素比较器。在比较器为空的情况下，使用键的自然顺序进行排序。由此映射集合中存储的键值必须实现Comparable接口。

##### 添加元素

```java
public V put(K key, V value) {
    Entry<K,V> t = root;

    // 如果根节点为 null，将新节点设为根节点
    if (t == null) {

        //类型校验
        compare(key, key); // type (and possibly null) check

        root = new Entry<>(key, value, null);
        size = 1;
        modCount++;
        return null;
    }


    int cmp;
    Entry<K,V> parent;
    // split comparator and comparable paths
    Comparator<? super K> cpr = comparator;

    //比较器不为空的情况下，使用指定比较器进行比较，找到待插入元素在红黑树中合适的位置
    if (cpr != null) {
        do {
            parent = t;
            cmp = cpr.compare(key, t.key);
            if (cmp < 0)
                t = t.left;
            else if (cmp > 0)
                t = t.right;
            else    //指定键值存在的情况下，重新设置对应的value，并返回旧的value
                return t.setValue(value);
        } while (t != null);
    }

    //比较器为空的情况下，使用键值自身顺序进行比较，找到待插入元素在红黑树中合适的位置
    else {
        if (key == null)
            throw new NullPointerException();
        @SuppressWarnings("unchecked")
        Comparable<? super K> k = (Comparable<? super K>) key;
        do {
            parent = t;
            cmp = k.compareTo(t.key);
            if (cmp < 0)
                t = t.left;
            else if (cmp > 0)
                t = t.right;
            else
                return t.setValue(value);
        } while (t != null);
    }

    //由以上代码找到待插入节点的父节点后，组装节点，并将新节点插入红黑树
    Entry<K,V> e = new Entry<>(key, value, parent);
    if (cmp < 0)
        parent.left = e;
    else
        parent.right = e;

    //插入新节点可能会破坏红黑树性质，这里修正一下
    fixAfterInsertion(e);
    size++;
    modCount++;
    return null;
}
```
在插入元素之前，如果树还没初始化，则直接将新节点设置为树的根节点；否则，在树结构中根据比较器找到新节点在红黑树中的适当位置，然后将新节点插入数中；在插入成功后，调用 `fixAfterInsertion方法` 调整一下红黑树结构。

在确认新节点插入的位置时，如果新节点的key值比父节点的小，则将新节点放在树结构的左节点位置；否则，为右节点位置。最终的结构是：<font color=red>树结构中，父节点的key值大于其左节点的key值，且小于其右节点的key值。</font>

##### fixAfterInsertion源码

```java
/**
 *  红黑树的性质：
 *      1. 节点是红色或黑色。
        2. 根是黑色。
        3. 所有叶子都是黑色（叶子是NIL节点）。
        4. 每个红色节点必须有两个黑色的子节点。（从每个叶子到根的所有路径上不能有两个连续的红色节点。）
        5. 从任一节点到其每个叶子的所有简单路径都包含相同数目的黑色节点。
 * 在新节点成功插入后，可能会破坏红黑树性质，因为需要使用 fixAfterInsertion 方法进行修正一下
 */
private void fixAfterInsertion(Entry<K,V> x) {
    //先插入的节点变为红色
    x.color = RED;

    while (x != null && x != root && x.parent.color == RED) {
        if (parentOf(x) == leftOf(parentOf(parentOf(x)))) {  //判断节点x的父节点是左节点还是右节点

            //获取节点x的叔父节点
            Entry<K,V> y = rightOf(parentOf(parentOf(x)));

            //节点x的叔父节点为红色，变色处理
            if (colorOf(y) == RED) {

                //将节点x的父节点及其叔父节点变为黑色
                setColor(parentOf(x), BLACK);
                setColor(y, BLACK);

                //将节点x的祖父节点变成红色
                setColor(parentOf(parentOf(x)), RED);

                //向上调整祖父节点
                x = parentOf(parentOf(x));


            } else {    //节点x的叔父节点为黑色，则调整结构

                //节点x为右节点，左旋
                if (x == rightOf(parentOf(x))) {
                    x = parentOf(x);
                    rotateLeft(x);
                }

                //右旋，变色
                setColor(parentOf(x), BLACK);
                setColor(parentOf(parentOf(x)), RED);
                rotateRight(parentOf(parentOf(x)));
            }
        } else {
            Entry<K,V> y = leftOf(parentOf(parentOf(x)));
            if (colorOf(y) == RED) {
                setColor(parentOf(x), BLACK);
                setColor(y, BLACK);
                setColor(parentOf(parentOf(x)), RED);
                x = parentOf(parentOf(x));
            } else {
                if (x == leftOf(parentOf(x))) {
                    x = parentOf(x);
                    rotateRight(x);
                }
                setColor(parentOf(x), BLACK);
                setColor(parentOf(parentOf(x)), RED);
                rotateLeft(parentOf(parentOf(x)));
            }
        }
    }
    root.color = BLACK;
}
```
上述方法的功能是：在红黑树的结构发生改变后，需要对其进行相应调整，已满足红黑树的性质。

下图为代码逻辑所对应的图解：

![调整逻辑图](http://t.cn/RQ0ic3v)

---

##### 获取元素

```java
public V get(Object key) {
    Entry<K,V> p = getEntry(key);
    return (p==null ? null : p.value);
}


final Entry<K,V> getEntry(Object key) {
    // Offload comparator-based version for sake of performance

    //如果比较器不为null，则使用特定比较器进行查找
    if (comparator != null)
        return getEntryUsingComparator(key);

    //在比较器为null的情况下，键值不能为null，否则会抛出空指针异常
    if (key == null)
        throw new NullPointerException();

    //在比较器为null的情况下，集合中所存储的键值必须实现了Comparable接口，因为可以直接进行类型转换
    @SuppressWarnings("unchecked")
    Comparable<? super K> k = (Comparable<? super K>) key;

    //利用比较器的特性开始比较大小  相同 return 小于 从左子树开始 大了 从右子树开始
    Entry<K,V> p = root;
    while (p != null) {
        int cmp = k.compareTo(p.key);
        if (cmp < 0)
            p = p.left;
        else if (cmp > 0)
            p = p.right;
        else
            return p;
    }
    return null;
}

```
在查找元素的时候，总体逻辑是：从根节点开始，如果目标值小于根节点的值，则再和根节点的左孩子进行比较。如果目标值大于根节点的值，则继续和根节点的右孩子比较。在查找过程中，如果目标值和二叉树中的某个节点值相等，则返回true，否则返回 false。

##### containsValue源码

```java
public boolean containsValue(Object value) {
    for (Entry<K,V> e = getFirstEntry(); e != null; e = successor(e))
        if (valEquals(value, e.value))
            return true;
    return false;
}
```
在判断映射中是否包含特定的值时，会根据树结构的 `中序遍历` 进行查找。在根据中序遍历，首先需要调用 `getFirstEntry方法` 查找最小的节点；然后调用 `successor方法` 依次遍历树结构进行判断。

##### getFirstEntry源码

```java
final Entry<K,V> getFirstEntry() {
    Entry<K,V> p = root;
    if (p != null)
        while (p.left != null)
            p = p.left;
    return p;
}
```
由于是根据树结构的中序遍历进行，所以遍历到的第一个元素应该是树结构中最左边子节点。

##### successor源码

```java
/**
 *
 * 二叉树的遍历，简单可以划分为：
 *  1. 深度优先
 *  2. 广度优先
 *
 *  其中，深度优先根据根节点相对于左右子节点的访问先后来划分为：
 *      1. 先序遍历：指先访问根，然后访问子树的遍历方式
 *      2. 中序遍历：指先访问左（右）子树，然后访问根，最后访问右（左）子树的遍历方式
 *      3. 后序遍历：指先访问子树，然后访问根的遍历方式
 *
 *  方法 successor() 是按照中序遍历
 */
static <K,V> Entry<K,V> successor(Entry<K,V> t) {
    if (t == null)
        return null;

    //如果节点t的右节点不为空，
    else if (t.right != null) {
        Entry<K,V> p = t.right;

        // while 循环找到中序后继结点  一直往左找
        while (p.left != null)
            p = p.left;
        return p;
    } else {
        Entry<K,V> p = t.parent;
        Entry<K,V> ch = t;
        while (p != null && ch == p.right) {
            ch = p;
            p = p.parent;
        }
        return p;
    }
}
```
方法 successor() 是按照中序遍历树结构。

![中序遍历](https://upload.wikimedia.org/wikipedia/commons/c/c4/%E4%B8%AD%E5%BA%8F%E9%81%8D%E5%8E%86.png)


关于TreeMap的移除操作，有点复杂，不在讲述了，看参考文章吧🤦




#### 参考文章
[红黑树简介](http://www.cnblogs.com/nullllun/p/8214599.html)

[树的遍历](https://zh.wikipedia.org/wiki/%E6%A0%91%E7%9A%84%E9%81%8D%E5%8E%86)

[红黑树](https://zh.wikipedia.org/wiki/%E7%BA%A2%E9%BB%91%E6%A0%91)

[TreeMap 源码分析](https://www.cnblogs.com/nullllun/p/8251212.html)