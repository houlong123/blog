---
title: CopyOnWriteArrayList源码学习
date: 2017-11-14 14:56:59
tags: java并发集合
---

#### 前言
CopyOnWriteArrayList是一个线程安全的，**`读操作无锁，但是写操作有锁`** 的ArrayList。是 `读写分离` 思想的体现。从名字中，我们可以推断出, 这个是<font color=red>`在 Write 时进行 Copy 数组元素的 ArrayList`</font>。是一种 **`空间换时间`** 的策略，适合读多写少的场景。

#### 源码解析
##### 内部数据结构

```java
public class CopyOnWriteArrayList<E>
    implements List<E>, RandomAccess, Cloneable, java.io.Serializable {
    
    final transient ReentrantLock lock = new ReentrantLock();
    /** The array, accessed only via getArray/setArray. */
    //底层数据结构。并被volatile修饰
    private transient volatile Object[] array;
}
```
这两个成员变量是该类的核心，对集合所有的`修改操作都需要使用lock加锁`，array则是整个集合的数据储存部分，关键在于该array被声明为volatile，保证了内存可见性。

##### 构造函数

```java
//默认创建一个元素数为0的数组，ArrayList则默认为10
public CopyOnWriteArrayList() {
    setArray(new Object[0]);
}

public CopyOnWriteArrayList(Collection<? extends E> c) {
    Object[] elements;
    if (c.getClass() == CopyOnWriteArrayList.class)
        elements = ((CopyOnWriteArrayList<?>)c).getArray();
    else {
        elements = c.toArray();
        // c.toArray might (incorrectly) not return Object[] (see 6260652)
        if (elements.getClass() != Object[].class)
            elements = Arrays.copyOf(elements, elements.length, Object[].class);
    }
    setArray(elements);
}

public CopyOnWriteArrayList(E[] toCopyIn) {
    setArray(Arrays.copyOf(toCopyIn, toCopyIn.length, Object[].class));
}

```
CopyOnWriteArrayList类提供了三个构造方法，值得一提的是无参构造方法，在无参构造方法中，会默认创建一个元素数为0的数组，而ArrayList则默认为10。其他两个构造函数会将传进来的集合通过Arrays.copyOf()方法转换成一个Object类型的数组，并用array成员变量存储起来。

##### add源码

<!-- more -->

```java
public boolean add(E e) {
    final ReentrantLock lock = this.lock;
    lock.lock();    //上锁
    try {
        Object[] elements = getArray(); //获取当前数组
        int len = elements.length;  //获取当前数组的长度
        //创建一个比当前数组长度多1的数组
        Object[] newElements = Arrays.copyOf(elements, len + 1);    //每增加一个新元素，都要进行一次数组的复制消耗，所以是一种空间换时间的策略
        newElements[len] = e;   //将新元素置于新数组的末尾
        setArray(newElements);  //将newElements设置为全局array
        return true;
    } finally {
        lock.unlock();  //释放锁
    }
}
```
代码逻辑：

1. 获取可重入锁
2. 获取当前数组及长度len
3. 将当前数组的元素拷贝到长度为len + 1 的新数组中
4. 将新元素置于新数组的尾部
5. 将新数组设置为全局array


##### add源码

```java
public void add(int index, E element) {
    final ReentrantLock lock = this.lock;
    lock.lock();    //上锁
    try {
        Object[] elements = getArray(); //获取当前数组
        int len = elements.length;      //获取当前数组的长度
        if (index > len || index < 0)   //参数判断
            throw new IndexOutOfBoundsException("Index: "+index+
                                                ", Size: "+len);
        Object[] newElements;
        int numMoved = len - index; //需要移位的数目
        if (numMoved == 0)  //插入的元素为原数组的最后一个元素
            newElements = Arrays.copyOf(elements, len + 1); //将数据插入到elements.length位置上
        else {
            newElements = new Object[len + 1];
            System.arraycopy(elements, 0, newElements, 0, index);    // 将原数组 index 前的数组都拷贝到新的数组里面
            System.arraycopy(elements, index, newElements, index + 1,
                             numMoved); // 将原数组 index 以后的元素都 copy到新的数组里面(包括index位置的元素)
        }
        newElements[index] = element;
        setArray(newElements);
    } finally {
        lock.unlock();
    }
}
```
该方法与上个添加方法大体实现逻辑一致，只不过是多了确认待插入元素位置的逻辑。如果待插入元素要插入的位置为数组最尾部，和上个添加方法逻辑一样。如果不是，则需要创建新数组（分两部分创建），最后将元素插入指定位置。

##### remove源码

```java
public E remove(int index) {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        Object[] elements = getArray(); //获取原数组
        int len = elements.length;  //获取原数组长度
        E oldValue = get(elements, index);  //在index > len时，会报数组越界错误
        int numMoved = len - index - 1; //由上可知，numMoved >= 0
        if (numMoved == 0)  // 说明删除的元素的位置在 len - 1 上, 直接拷贝原数组的前 len - 1 个元素
            setArray(Arrays.copyOf(elements, len - 1));
        else {
            Object[] newElements = new Object[len - 1];
            System.arraycopy(elements, 0, newElements, 0, index);   // 拷贝原数组 0 - index之间的元素 (index 不拷贝)
            System.arraycopy(elements, index + 1, newElements, index,
                             numMoved);     // 拷贝原数组 index+1 到末尾之间的元素 (index＋1也进行拷贝)
            setArray(newElements);
        }
        return oldValue;
    } finally {
        lock.unlock();
    }
}
```
该方法为删除指定索引上的元素，与在指定位置上添加元素逻辑有点相似。

##### remove源码

```java
public boolean remove(Object o) {
    Object[] snapshot = getArray();
    int index = indexOf(o, snapshot, 0, snapshot.length);   //获取元素o在数组中的位置。-1表示不存在
    return (index < 0) ? false : remove(o, snapshot, index);
}
```
该方法为删除指定元素。处理逻辑为：

1. 获取当前数组
2. 获取指定元素在数组中的位置index
3. 如果 `index < 0 ` 说明在数组中没有找到指定元素，则直接返回；如果 `index >= 0 `，则调用内部重载方法remove

##### getArray源码

```java
final Object[] getArray() {
    return array;
}
```
很简单，直接返回全局array。由于修饰符`volatile`保证其内存可见性。

##### indexOf源码

```java
private static int indexOf(Object o, Object[] elements,
                           int index, int fence) {
    if (o == null) {
        for (int i = index; i < fence; i++)
            if (elements[i] == null)
                return i;
    } else {
        for (int i = index; i < fence; i++)
            if (o.equals(elements[i]))
                return i;
    }
    return -1;
}
```
由源码可知，在确认元素的位置时，只是简单的遍历数组。在找到时，返回元素所在的位置；没有找到时，返回-1。


##### remove源码

```java
private boolean remove(Object o, Object[] snapshot, int index) {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        Object[] current = getArray();
        int len = current.length;

        //下面的if语句的主要功能是：在数组发生变化后，重新定位要移除的元素
        if (snapshot != current) findIndex: {   //snapshot != current，表示数组被另一个线程操作过，数组元素有变化
            int prefix = Math.min(index, len);
            for (int i = 0; i < prefix; i++) {
                if (current[i] != snapshot[i] && eq(o, current[i])) {   // 找出 current 数组里面 元素 o 所在的位置 i, 并且赋值给 index
                    index = i;
                    break findIndex;    //直接跳到findIndex:表示的括号外面
                }
            }
            if (index >= len)   //则说明 元素 o 在另外的线程中已经被删除, 直接 return
                return false;
            if (current[index] == o)    //index 位置上的元素 o 还在那边, 直接 break
                break findIndex;
            index = indexOf(o, current, index, len);    // 在 index 与 len 之间寻找元素, 找到位置直接接下来的代码, 没找到 直接 return
            if (index < 0)
                return false;
        }
        Object[] newElements = new Object[len - 1];
        System.arraycopy(current, 0, newElements, 0, index);
        System.arraycopy(current, index + 1,
                         newElements, index,
                         len - index - 1);
        setArray(newElements);
        return true;
    } finally {
        lock.unlock();
    }
}
```
该方法为remove的重载方法。在调用 `remove(Object o)` 方法时，上个方法会先获取待移除元素所在的位置，然后调用重载方法 `remove(Object o, Object[] snapshot, int index)` 。在重载方法中，会判断数组是否发生了变化（snapshot != current），如果发生变化的话，则重新定位待删除元素的位置。然后执行移除操作。


##### set源码

```java
public E set(int index, E element) {
    final ReentrantLock lock = this.lock;
    lock.lock();    //获取锁
    try {
        Object[] elements = getArray();
        E oldValue = get(elements, index);  //获取原数组中对应index位置的元素

        if (oldValue != element) {  //在新值与原有值不相同时
            int len = elements.length;
            Object[] newElements = Arrays.copyOf(elements, len);
            newElements[index] = element;
            setArray(newElements);
        } else {
            // Not quite a no-op; ensures volatile write semantics  确保volatile写语义
            setArray(elements);
        }
        return oldValue;
    } finally {
        lock.unlock();
    }
}
```
<font color = red>`到目前为止，上面所介绍的方法都是常见的修改方法。它们内部实现原理的共性都是：
1）加锁
2）进行一次数组的复制消耗。` </font>

##### get源码

```java
public E get(int index) {
    return get(getArray(), index);  //读操作没有上锁，因为array数组是volatile修饰的
}

private E get(Object[] a, int index) {
    return (E) a[index];
}
```
get方法是直接取得该线程当前拥有的array数组，不需要额外的同步开销。因为array数组是volatile修饰的，也就是当你<font color=red>`对volatile进行写操作后，会将写过后的array数组强制刷新到主内存`</font>，从而在读操作getArray()时会强制从主内存将array读出来。


##### addIfAbsent源码

```java
/**
 * Appends the element, if not present.
 *
 * @param e element to be added to this list, if absent
 * @return {@code true} if the element was added
 */
public boolean addIfAbsent(E e) {
    Object[] snapshot = getArray();
    return indexOf(e, snapshot, 0, snapshot.length) >= 0 ? false :
        addIfAbsent(e, snapshot);
}
```
该方法的功能为：若列表中没有待插入的元素，则进行插入操作。`CopyOnWriteArraySet的add方法内部原理就是调用该方法`。处理逻辑：
1. 在原数组中查找待插入元素的位置index
2. 若`index >= 0`，直接返回false
3. 若`index < 0`，调用重载方法 `addIfAbsent(E e, Object[] snapshot)`


##### addIfAbsent源码

```java
private boolean addIfAbsent(E e, Object[] snapshot) {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        Object[] current = getArray();  //获取当前数组
        int len = current.length;   //获取当前数组的长度
        if (snapshot != current) {  //期间数组发生了变化。即其他线程有操作了该集合
            // Optimize for lost race to another addXXX operation
            int common = Math.min(snapshot.length, len);
            //在for循环中，找到，则返回false
            for (int i = 0; i < common; i++)
                if (current[i] != snapshot[i] && eq(e, current[i]))
                    return false;

            //重新定位待插入元素，如果定位到，则说明已存在，返回FALSE
            if (indexOf(e, current, common, len) >= 0)
                    return false;
        }

        //插入元素
        Object[] newElements = Arrays.copyOf(current, len + 1);
        newElements[len] = e;
        setArray(newElements);
        return true;
    } finally {
        lock.unlock();
    }
}
```
该方法的功能是：不重复的插入元素。代码逻辑：
1. 获取锁
2. 获取当前数组current及当前数组长度len
3. 如果期间数组发生了改变，首先通过for循环查询待插入元素是否已在数组中，是返回FALSE；若没有，则通过indexOf方法定位带插入元素的位置，定位到，返回FALSE。否则，执行第4步。
4. 创建新数组，并将原数组元素拷贝到新数组
5. 在新数组尾部添加待插入元素
6. 设置全局array数组
7. 返回true。


##### 备注：
<font color="#723638">*CopyOnWriteArraySet的原理场景和CopyOnWriteArrayList一样，只不过是不能有重复的元素放入，它里面包含一个CopyOnWriteArrayList，真正调用的方法都 是CopyOnWriteArrayList的方法*</font>



#### 参考文章
[CopyOnWriteArrayList 源码分析 (基于Java 8)](http://www.jianshu.com/p/8c66e2e7411c)

[Java技术——CopyOnWriteArrayList源码解析](http://blog.csdn.net/seu_calvin/article/details/70198220)

[CopyOnWriteArrayList源码分析](https://my.oschina.net/summerpxy/blog/405728)

[Java并发编程与技术内幕:CopyOnWriteArrayList、CopyOnWriteArraySet源码解析](http://blog.csdn.net/Evankaka/article/details/51832938)
