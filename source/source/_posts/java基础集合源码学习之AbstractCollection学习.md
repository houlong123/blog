---
title: java基础集合源码学习之AbstractCollection学习
date: 2018-01-17 15:26:28
tags: java基础集合
---


#### 前言
**Java集合框架**是指java的集合类。<font color=pick>`Collection`</font> 接口是一组<font color=pick>`允许重复`</font>的对象。<font color=red>`Set`</font> 接口继承 Collection，但<font color=red>`不允许重复`</font>，使用自己内部的一个排列机制。 <font color=gray>`List`</font> 接口继承 Collection，<font color=gray>`允许重复`</font>，以元素安插的次序来放置元素，不会重新排列。Map接口是一组成对的键－值对象，即所持有的是key-value pairs。`Map中不能有重复的key`。拥有自己的内部排列机制。

#### java基础集合框架图
<a data-flickr-embed="true"  href="http://img.blog.csdn.net/20140628144205625" title="java基础集合框架"><img src="http://img.blog.csdn.net/20140628144205625" width="640" height="453" alt="java基础集合框架"></a>

从上面的集合类继承图可以看出，集合类主要分为两大类：Collection和Map。


#### Collection 接口

<!-- more -->

```java
public interface Collection<E> extends Iterable<E> {

    //集合的基本操作
    int size();
    boolean isEmpty();
    boolean contains(Object o);
    Object[] toArray();
    <T> T[] toArray(T[] a);
    boolean add(E e);
    boolean remove(Object o);
    boolean containsAll(Collection<?> c);
    boolean addAll(Collection<? extends E> c);
    boolean removeAll(Collection<?> c);
    boolean retainAll(Collection<?> c);
    void clear();
    boolean equals(Object o);
    int hashCode();
    
    //获取迭代器
    Iterator<E> iterator();
}
```
Collection是List、Set等集合高度抽象出来的接口，它包含了这些集合的基本操作，它主要又分为两大部分：List和Set。


#### AbstractCollection 源码

```java
public abstract class AbstractCollection<E> implements Collection<E> {
    
    //数组最大容量
    private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;
    
    /**             抽象方法，由子类实现          **/
    public abstract Iterator<E> iterator();
    public abstract int size();
    
    
    /**            抛异常实现，由子类实现      **/
    public boolean add(E e) {
        throw new UnsupportedOperationException();
    }
    
    
    /**           以下方法都进行了默认实现      **/
    
    public boolean isEmpty() {
        return size() == 0;
    }
    
    //通过获取迭代器，然后依次与待比较对象进行比较
    public boolean contains(Object o) {
        Iterator<E> it = iterator();
        if (o==null) {
            while (it.hasNext())
                if (it.next()==null)
                    return true;
        } else {
            while (it.hasNext())
                if (o.equals(it.next()))
                    return true;
        }
        return false;
    }
    
    //将列表转换为数组
    public Object[] toArray() {
        // Estimate size of array; be prepared to see more or fewer elements
        //根据当前数组大小创建数组
        Object[] r = new Object[size()];

        //获取迭代器
        Iterator<E> it = iterator();

        //根据新建数组大小，依次将元素放入新数组中
        for (int i = 0; i < r.length; i++) {
            //在遍历的过程中，出现元素个数比预计的少，则直接重新拷贝（可能出现其他线程操作了该集合的元素）
            if (! it.hasNext()) // fewer elements than expected
                return Arrays.copyOf(r, i);
            r[i] = it.next();
        }

        //在将新数组填充完后，如果集合中依然有元素，则扩容继续填充；否则，直接返回新建数组
        return it.hasNext() ? finishToArray(r, it) : r;
    }
    
    public <T> T[] toArray(T[] a) {
        // Estimate size of array; be prepared to see more or fewer elements
        int size = size();

        //如果参数a 的大小大于集合大小size的话，则直接使用a ;否则，通过反射，新创建一个大小为size，类型与a相同的数组
        T[] r = a.length >= size ? a :
                  (T[])java.lang.reflect.Array
                  .newInstance(a.getClass().getComponentType(), size);

        Iterator<E> it = iterator();

        for (int i = 0; i < r.length; i++) {

            //比预想中的元素个数少
            if (! it.hasNext()) { // fewer elements than expected
                //如果不是新建的数组，则使用null值填充
                if (a == r) {
                    r[i] = null; // null-terminate
                } else if (a.length < i) {  //如果使用的是新建数组，则截取部分数组返回
                    return Arrays.copyOf(r, i);
                } else {    //a数组可以容下所有元素
                    //从新建数组中拷贝元素到a中
                    System.arraycopy(r, 0, a, 0, i);
                    //将a数组空余的位置null
                    if (a.length > i) {
                        a[i] = null;
                    }
                }
                return a;
            }

            r[i] = (T)it.next();
        }
        //比预期有更多元素，扩容继续填充   more elements than expected
        return it.hasNext() ? finishToArray(r, it) : r;
    }
    
    private static <T> T[] finishToArray(T[] r, Iterator<?> it) {
        int i = r.length;
        while (it.hasNext()) {  //如果遍历器中依然有元素
            int cap = r.length;
            //该处的作用是：首次遍历时，将原数组大小进行扩容
            if (i == cap) {
                //将元数组大小扩容：新增加(cap >> 1) + 1
                int newCap = cap + (cap >> 1) + 1;
                // overflow-conscious code
                //如果新扩容后的数组大小大于MAX_ARRAY_SIZE，则重新设置新扩容后的数组大小
                if (newCap - MAX_ARRAY_SIZE > 0)
                    newCap = hugeCapacity(cap + 1);

                //重新分配数组
                r = Arrays.copyOf(r, newCap);
            }
            r[i++] = (T)it.next();
        }
        // trim if overallocated  如果分配空间过多，则去除多余的
        return (i == r.length) ? r : Arrays.copyOf(r, i);
    }
    
    private static int hugeCapacity(int minCapacity) {
        if (minCapacity < 0) // overflow
            throw new OutOfMemoryError
                ("Required array size too large");
        return (minCapacity > MAX_ARRAY_SIZE) ?
            Integer.MAX_VALUE :
            MAX_ARRAY_SIZE;
    }

    //该方法的实现，与contains方法的逻辑一致，只不过在包含的时候，会删除元素
    public boolean remove(Object o) {
        Iterator<E> it = iterator();
        if (o==null) {
            while (it.hasNext()) {
                if (it.next()==null) {
                    it.remove();
                    return true;
                }
            }
        } else {
            while (it.hasNext()) {
                if (o.equals(it.next())) {
                    it.remove();
                    return true;
                }
            }
        }
        return false;
    }
    
    //内部循环调用contains方法，有一个元素不包含，返回FALSE
    public boolean containsAll(Collection<?> c) {
        for (Object e : c)
            if (!contains(e))
                return false;
        return true;
    }
    
    //模板方法，内部循环调用add方法，在Collection类中，add()方法默认抛异常，具体逻辑由子类实现
    public boolean addAll(Collection<? extends E> c) {
        boolean modified = false;
        for (E e : c)
            if (add(e))
                modified = true;
        return modified;
    }
    
    //该方法功能：获取调用者集合中所有不在参数集合c的元素（调用者的差集）
    public boolean removeAll(Collection<?> c) {
        Objects.requireNonNull(c);
        boolean modified = false;
        Iterator<?> it = iterator();
        //遍历集合
        while (it.hasNext()) {
            //参数集合c中包含元素，则从调用者集合中删除该元素
            if (c.contains(it.next())) {
                it.remove();
                modified = true;
            }
        }
        return modified;
    }
    
    //该方法实现功能为：获取两集合的交集（交集）
    public boolean retainAll(Collection<?> c) {
        Objects.requireNonNull(c);
        boolean modified = false;
        Iterator<E> it = iterator();
        while (it.hasNext()) {
            //调用者集合中的元素不在参数集合c中，则将该元素从调用者集合中移除
            if (!c.contains(it.next())) {
                it.remove();
                modified = true;
            }
        }
        return modified;
    }
    
    //清空集合
    public void clear() {
        Iterator<E> it = iterator();
        while (it.hasNext()) {
            it.next();
            it.remove();
        }
    }
    
    public String toString() {
        Iterator<E> it = iterator();
        if (! it.hasNext())
            return "[]";

        StringBuilder sb = new StringBuilder();
        sb.append('[');
        for (;;) {
            E e = it.next();
            sb.append(e == this ? "(this Collection)" : e);
            if (! it.hasNext())
                return sb.append(']').toString();
            sb.append(',').append(' ');
        }
    }
}
```
抽象类AbstractCollection实现了Collection接口，并实现了若干方法，子类只需要实现抽象方法`iterator()`，`size()`及默认实现方法`add(E e)`。在方法`add(E e)`中，只是简单的抛出了异常，具体实现逻辑还需要子类去实现。AbstractCollection是典型的`缺省适配器模式`的应用，子类只需直接继承该抽象类，并实现自己需要的方法即可，而不用实现接口中的全部抽象方法。


#### 参考文章
[Java 集合类详解（含类图）](http://www.cnblogs.com/Berryxiong/p/6134012.html)

[Java集合框架](http://blog.csdn.net/ns_code/article/details/35564663)