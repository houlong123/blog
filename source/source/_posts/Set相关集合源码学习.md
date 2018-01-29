---
title: Set相关集合源码学习
date: 2018-01-29 14:06:44
tags: java基础集合
---

### 前言
在前面几个章节中，已经对集合中的 `List`列表一支进行了相关的学习。在我们日常编程中，使用最多的集合还有 `Set` 。本章我们将学习Set的相关知识。


#### java基础集合框架
<a data-flickr-embed="true"  href="http://img.blog.csdn.net/20140628144205625" title="java基础集合框架"><img src="http://img.blog.csdn.net/20140628144205625" width="640" height="453" alt="java基础集合框架"></a>

#### HashSet源码学习
在上图中，我们可以看到，`Collection接口` 的另一分支：`Set` 集合，常用的有：HashSet，TreeSet等。在此，我们先学习一下平时最常用的 HashSet。

##### 数据结构

<!-- more -->

```java
/** API文档描述
 * This class implements the <tt>Set</tt> interface, backed by a hash table
 * (actually a <tt>HashMap</tt> instance).  It makes no guarantees as to the
 * iteration order of the set; in particular, it does not guarantee that the
 * order will remain constant over time.  This class permits the <tt>null</tt>
 * element.
 * 简单翻译为：HashSet是一个实现了Set接口，内部由一个哈希表支持，并允许存储null值的无序集合
 */
public class HashSet<E>
    extends AbstractSet<E>
    implements Set<E>, Cloneable, java.io.Serializable {
    
    //内部维持一个HaahMap对象
    private transient HashMap<E,Object> map;
    // 内部HaahMap对象存储的值
    private static final Object PRESENT = new Object();
}
```
从 HashSet 的源码中，我们可以看出，该类继承了 `AbstractSet抽象类`，并实现了 `Set接口`。在其内部，维持了一个 `HashMap`类型的对象。由此可知，<font color=red>HashSet的内部是通过HashMap类型对象来实现具体操作的。</font>

###### Set接口

```java
public interface Set<E> extends Collection<E> {
    
    //增删操作
    boolean add(E e);
    boolean addAll(Collection<? extends E> c);
    boolean remove(Object o);
    boolean removeAll(Collection<?> c);
    void clear();
    boolean retainAll(Collection<?> c);
    
    //通用方法
    int size();
    boolean isEmpty();
    boolean contains(Object o);
    Object[] toArray();
    <T> T[] toArray(T[] a);
    boolean containsAll(Collection<?> c);
    boolean equals(Object o);
    int hashCode();
    
    //迭代器
    Iterator<E> iterator();
}
```
在 `Set接口`中，定义了Set集合的相关方法。具体实现需由子类进行完成。


###### AbstractSet源码

```java
public abstract class AbstractSet<E> extends AbstractCollection<E> implements Set<E> {

    protected AbstractSet() {
    }
    
    
    public boolean equals(Object o) {
        if (o == this)
            return true;

        if (!(o instanceof Set))
            return false;
        Collection<?> c = (Collection<?>) o;
        if (c.size() != size())
            return false;
        try {
            //调用父类的AbstractCollection的containsAll方法。具体实现是：遍历集合c中的元素，然后一个个判断是否存在
            return containsAll(c);
        } catch (ClassCastException unused)   {
            return false;
        } catch (NullPointerException unused) {
            return false;
        }
    }
    
    //计算集合的hashCode。集合内所有元素的hashCode之和
    public int hashCode() {
        int h = 0;
        Iterator<E> i = iterator();
        while (i.hasNext()) {
            E obj = i.next();
            if (obj != null)
                h += obj.hashCode();
        }
        return h;
    }

    //从调用者中删除集合c中的元素
    public boolean removeAll(Collection<?> c) {
        //待删除集合不能为null
        Objects.requireNonNull(c);
        boolean modified = false;

        if (size() > c.size()) {    //在待删除集合的个数小于方法调用者的时候，遍历待删除集合
            for (Iterator<?> i = c.iterator(); i.hasNext(); )
                modified |= remove(i.next());
        } else {
            for (Iterator<?> i = iterator(); i.hasNext(); ) {
                if (c.contains(i.next())) {
                    i.remove();
                    modified = true;
                }
            }
        }
        return modified;
    }
}
```
在抽象类 AbstractSet 中，只是实现了`equals`, `hashCode`, `removeAll` 这三个方法。内部最终还是会调用父类AbstractCollection的相关方法。

##### 构造方法

```java
//默认构造一个新的空set集合。内部的HashMap对象实现默认构造，其中初始化大小为16，加载因子为0.75
public HashSet() {
    map = new HashMap<>();
}

public HashSet(Collection<? extends E> c) {
    //初始化内部HashMap集合。默认加载因子为0.75
    map = new HashMap<>(Math.max((int) (c.size()/.75f) + 1, 16));
    //模板方法，内部循环调用add方法，在Collection类中，add()方法默认抛异常，具体逻辑由子类实现
    addAll(c);
}

//使用指定大小和加载因子创建Set集合。
public HashSet(int initialCapacity, float loadFactor) {
    map = new HashMap<>(initialCapacity, loadFactor);
}

public HashSet(int initialCapacity) {
    map = new HashMap<>(initialCapacity);
}

//构造一个新的空链接哈希Set。为了区别HashSet(int initialCapacity, float loadFactor)，加了个无用参数。该构造函数在构造LinkedHashSet对象时用到。
HashSet(int initialCapacity, float loadFactor, boolean dummy) {
    map = new LinkedHashMap<>(initialCapacity, loadFactor);
}
```
HashSet 类中有5个不同的构造函数。其中最上面4个，内部创建了 `HashMap`类型的对象；而最后一个构造函数，则构建了一个 `LinkedHashMap` 类型的对象。所以最后一个对象在构造LinkedHashSet对象时会用到。下文会分析到。由于HashSet内部维持了一个 HashMap 类型的对象，且各种操作都是通过这个内部对象实现的。所以，HashSet构造函数的主要作用是创建
HashMap对象。关于 Map 的相关子类源码，我们下篇文章接着分析。在此处只要知道HashSet内部操作都是围绕着HashMap进行的。各种构造函数，也是为了初始化HashMap的大小和设置HashMap的加载因子。

##### 基本方法

```java

public boolean add(E e) {
    return map.put(e, PRESENT)==null;
}

public boolean remove(Object o) {
    return map.remove(o)==PRESENT;
}

public void clear() {
    map.clear();
}

//浅拷贝
public Object clone() {
    try {
        HashSet<E> newSet = (HashSet<E>) super.clone();
        newSet.map = (HashMap<E, Object>) map.clone();
        return newSet;
    } catch (CloneNotSupportedException e) {
        throw new InternalError(e);
    }
}

public int size() {
    return map.size();
}
    
public boolean isEmpty() {
    return map.isEmpty();
} 

public boolean contains(Object o) {
    return map.containsKey(o);
}


```
可见，HashMap 的相关操作都是通过 HashMap对象实现的。

关于 `Set` 集合的其他子类，在此不做详细讲述，因为实现逻辑和 HashMap 大致一样，只不过 <font color=red>LinkedHashSet内部维持一个 LinkedHashMap，TreeSet内部维持的是一个TreeMap</font>。


###### LinkedHashSet源码

```java
public class LinkedHashSet<E>
extends HashSet<E>
implements Set<E>, Cloneable, java.io.Serializable {

    /**
     *  内部全是调用HashSet的构造函数，内部创建了一个LinkedHashMap集合
     *
     HashSet(int initialCapacity, float loadFactor, boolean dummy) {
        map = new LinkedHashMap<>(initialCapacity, loadFactor);
    }
     */
    public LinkedHashSet(int initialCapacity, float loadFactor) {
        super(initialCapacity, loadFactor, true);
    }
    
    public LinkedHashSet(int initialCapacity) {
        super(initialCapacity, .75f, true);
    }
    
    public LinkedHashSet() {
        super(16, .75f, true);
    }
}
```

###### TreeSet源码

```java
public class TreeSet<E> extends AbstractSet<E>
    implements NavigableSet<E>, Cloneable, java.io.Serializable {
    
    //内部维持了一个NavigableMap对象，TreeMap为其子类
    private transient NavigableMap<E,Object> m;
    
    /**
     * 内部构建了一个TreeMap对象。该对象实现相关操作
     */
    public TreeSet() {
        this(new TreeMap<E,Object>());
    }
    
    TreeSet(NavigableMap<E,Object> m) {
        this.m = m;
    }
}
```
