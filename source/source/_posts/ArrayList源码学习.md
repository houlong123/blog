---
title: ArrayList源码学习
date: 2018-01-23 15:32:21
tags: java基础集合
---

#### 前言
ArrayList是开发人员使用最为普遍的类。该类是一个可调整大小的`List接口`实现。该类实现了所有可选的列表操作，并允许存储包括 `null` 在内的所有元素。

每个ArrayList实例都有一个容量。 容量是用于存储列表中的元素的数组的大小。 它总是至少与列表大小一样大。 当元素被添加到ArrayList时，其容量会自动增长。

需要注意的是：<font color=red>ArrayList不是同步的，即非线程安全的的类</font>。

#### 源码解析

<!-- more -->

##### 数据结构

```java
public class ArrayList<E> extends AbstractList<E>
        implements List<E>, RandomAccess, Cloneable, java.io.Serializable {
    //默认数组大小为10
    private static final int DEFAULT_CAPACITY = 10;

    //在创建ArrayList时，如果传递的数据容量为0，则使用该空实例
    private static final Object[] EMPTY_ELEMENTDATA = {};
    
    //在创建ArrayList时，如果没有传递参数，则使用该空实例。在添加元素时，如果数组为DEFAULTCAPACITY_EMPTY_ELEMENTDATA，则默认数组大小为10
    private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};

    //存储数组的底层结构
    transient Object[] elementData; // non-private to simplify nested class access

    //数组大小
    private int size;
    
    //数组最多存储容量
    private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;

    //内部迭代器
    private class Itr implements Iterator<E> {}
    
    private class ListItr extends Itr implements ListIterator<E> {}
    
    //内部类
    private class SubList extends AbstractList<E> implements RandomAccess {}
}
```
由类的声明中，我们也可以看出，ArrayList继承了[AbstractList抽象类](https://houlong123.github.io/2018/01/19/java基础集合源码学习之AbstractList学习/)，并实现了List接口。其内部还有一个迭代器类，用于创建列表的迭代器。这是`迭代器模式`的应用。`DEFAULT_CAPACITY` 变量是在创建ArrayList对象时，默认的存储容量。底层使用数组用与存储元素。


##### 构造函数

```java
public ArrayList(int initialCapacity) {
    //指定数组大小initialCapacity大于0时，创建一个指定大小的数组
    if (initialCapacity > 0) {
        this.elementData = new Object[initialCapacity];
    } else if (initialCapacity == 0) {  //如果指定数组大小为0，则使用默认空数组
        this.elementData = EMPTY_ELEMENTDATA;
    } else {
        throw new IllegalArgumentException("Illegal Capacity: "+
                                           initialCapacity);
    }
}


//默认数组大小为10
public ArrayList() {
    this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
}


public ArrayList(Collection<? extends E> c) {
    elementData = c.toArray();
    if ((size = elementData.length) != 0) {
        // c.toArray might (incorrectly) not return Object[] (see 6260652)
        if (elementData.getClass() != Object[].class)
            elementData = Arrays.copyOf(elementData, size, Object[].class);
    } else {
        // replace with empty array.
        this.elementData = EMPTY_ELEMENTDATA;
    }
}
```
从源码中，可以看出：ArrayList类有三个构造函数。 其目的都是对ArrayList底层的存储容器进行初始化。`ArrayList(int initialCapacity)`创建了一个指定大小为`initialCapacity`的ArrayList对象；`ArrayList()`创建了一个默认大小的ArrayList对象，在添加元素的时候可以看到默认容量大小为10；`ArrayList(Collection<? extends E> c)`则是将集合c转换为数组的形式，然后以ArrayList对象的形式展现。


##### add源码

```java
public boolean add(E e) {
    //在添加元素之前，先判断数组是否需要扩容
    ensureCapacityInternal(size + 1);  // Increments modCount!!
    //扩容后，将元素添加进去
    elementData[size++] = e;
    return true;
}
```
代码逻辑实现很简单。就是在添加元素前，判断是否需要扩容，然后将元素添加到底层数组中。


##### ensureCapacityInternal源码

```java
private void ensureCapacityInternal(int minCapacity) {
    //如果在创建ArrayList时，内部数组为DEFAULTCAPACITY_EMPTY_ELEMENTDATA，则默认大小为10
    if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
        minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity);
    }

    ensureExplicitCapacity(minCapacity);
}
```
我们可以回顾一下，我们以前都知道在创建ArrayList对象时，如果没有指定容量大小，则默认大小为10。那么在ArrayList中是如何实现的呢？在看到ensureCapacityInternal源码的源码时，我们就明朗了。在上面讲解ArrayList的构造函数时，我们已经知道了，在没有指定容量大小时，会有`this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;`这个操作。而在添加元素进行扩容时，会先判断`elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA`，如果成立，则`minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity)`。其中`DEFAULT_CAPACITY=10`，到此，我们就知道了为什么创建ArrayList对象的默认大小为10了。

##### ensureExplicitCapacity源码

```java
private void ensureExplicitCapacity(int minCapacity) {
    modCount++;

    // overflow-conscious code
    if (minCapacity - elementData.length > 0)   //在新数组大小大于目前数组中的元素个数时，进行扩容
        grow(minCapacity);
}
```
该方法是判断在添加元素前，是否需要对底层数组进行扩容。在扩容前，我们可以看到`modCount++;`其中，modCount是ArrayList的父类AbstractList中的一个属性， 该属性是记录：集合的修改次数。<font color=red>在对列表进行增删的时候，都是修改该属性值</font>。其中关于常见的`ConcurrentModificationException`异常，也是与该属性有关。具体可以看[AbstractList学习](https://houlong123.github.io/2018/01/19/java基础集合源码学习之AbstractList学习/)中的讲解。如果需要扩容，则会调用 grow方法 进行扩容。

##### grow源码

```java
//数组扩容
private void grow(int minCapacity) {
    // overflow-conscious code
    //数组中目前存储的元素个数
    int oldCapacity = elementData.length;

    //数组扩容后，新的数组大小。新增原数组的一半
    int newCapacity = oldCapacity + (oldCapacity >> 1);

    //如果新数组大小小于最低要求，则重新赋值数组大小
    if (newCapacity - minCapacity < 0)
        newCapacity = minCapacity;

    //如果新数组大小超过了最大规定值，则重新赋值数组大小
    if (newCapacity - MAX_ARRAY_SIZE > 0)
        newCapacity = hugeCapacity(minCapacity);
    // minCapacity is usually close to size, so this is a win:
    elementData = Arrays.copyOf(elementData, newCapacity);
}
```
从代码的实现可以看出，底层是通过`Arrays.copyOf`进行数组拷贝实现扩容的。所以该方法的重要逻辑是要确认列表新容量的大小。确认逻辑为：
1. 获取旧的容量大小： oldCapacity
2. 通过旧的容量大小计算新的容量：`newCapacity = oldCapacity + (oldCapacity >> 1)`，即：增加原来容量的一半
3. 判断新容量newCapacity 与参数最小容量minCapacity的大小。取两者中较大的。
4. 判断新容量是否超过了最大规定值。超过的话，调用`hugeCapacity`方法重新确定新容量。
5. 调用`Arrays.copyOf`方法进行扩容。

##### hugeCapacity源码

```java
//确定数组最终大小。如果参数minCapacity大于MAX_ARRAY_SIZE，则数组大小为Integer.MAX_VALUE；否则数组大小为 MAX_ARRAY_SIZE
private static int hugeCapacity(int minCapacity) {
    if (minCapacity < 0) // overflow
        throw new OutOfMemoryError();
    return (minCapacity > MAX_ARRAY_SIZE) ?
        Integer.MAX_VALUE :
        MAX_ARRAY_SIZE;
}
```

##### 重载add源码

```java
public void add(int index, E element) {
    //校验index的合法性。范围为：[0,size]
    rangeCheckForAdd(index);

    //在添加元素之前，先判断数组是否需要扩容
    ensureCapacityInternal(size + 1);  // Increments modCount!!

    //将数组elementData的元素从index位置开始拷贝，赋值到数组elementData的index + 1开始处
    System.arraycopy(elementData, index, elementData, index + 1,
                     size - index);

    //在数组elementData的位置index处添加元素element
    elementData[index] = element;
    size++;
}
```
该添加方法是在指定索引index处添加元素。
1. 校验index是否在合法范围内。合法范围为：[0,size]
2. 判断是否需要进行扩容
3. 通过数组拷贝，将index处以后的所有元素后移，将索引index处置空
4. 将元素element添加到索引index处
5. 增加列表内存储的元素个数size值。


##### rangeCheckForAdd源码

```java
private void rangeCheckForAdd(int index) {
    if (index > size || index < 0)
        throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
}
```


##### addAll源码

```java
public boolean addAll(Collection<? extends E> c) {
    //获取集合的数组形式
    Object[] a = c.toArray();
    int numNew = a.length;

    //数组扩容
    ensureCapacityInternal(size + numNew);  // Increments modCount

    //将数组a中的元素依次拷贝到elementData中
    System.arraycopy(a, 0, elementData, size, numNew);

    //修改数组的大小
    size += numNew;
    return numNew != 0;
}
```
该方法的功能是：将集合c中的元素添加到列表中。代码实现逻辑：
1. 获取集合c的数组表现形式
2. 判断是否需要扩容
3. 通过底层数组拷贝，将集合c中的元素依次添加到列表中
4. 修改存储的元素个数size值并返回。

##### 重载addAll源码

```java
public boolean addAll(int index, Collection<? extends E> c) {
    //检查index的范围。正常范围为：[0, size]
    rangeCheckForAdd(index);

    Object[] a = c.toArray();
    int numNew = a.length;
    ensureCapacityInternal(size + numNew);  // Increments modCount

    //移动元素的个数
    int numMoved = size - index;
    if (numMoved > 0)
        //将数组elementData从index开始的元素全都往后移动。留出numMoved个空位用于存储参数集合c中的元素
        System.arraycopy(elementData, index, elementData, index + numNew,
                         numMoved);

    //将参数集合c中的元素拷贝到数组elementData中
    System.arraycopy(a, 0, elementData, index, numNew);
    size += numNew;
    return numNew != 0;
}
```
该方法与重载的 `addAll(Collection<? extends E> c)`方法类似。不同之处在于，在往列表中添加元素的时候，需要判断index是否位于合法范围内；然后将集合c中的元素依次添加进从index处开始的位置。

##### get源码

```java
public E get(int index) {
    //先校验参数是否合法，合法的话，直接从底层数组中获取索引index处的元素，并返回
    rangeCheck(index);
    return elementData(index);
}
```
该方法是获取索引index处的元素值，在获取之前会先判断index的合法性。然后从数组中获取index处的值。

##### rangeCheck源码

```java
//检查给定的索引是否在范围内。没有检查负数，因为它总是在数组访问之前立即使用，如果索引为负，则抛出ArrayIndexOutOfBoundsException异常。
private void rangeCheck(int index) {
    if (index >= size)
        throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
}
```

##### elementData源码

```java
E elementData(int index) {
    return (E) elementData[index];
}
```

##### set源码

```java
public E set(int index, E element) {
    //校验index的合法性。范围为：(~,size]。之所以没有限制下限，是因为在调用elementData()方法的时候，会抛异常
    rangeCheck(index);

    E oldValue = elementData(index);

    //将新值覆写旧值
    elementData[index] = element;

    //返回旧值
    return oldValue;
}
```
将列表的index位的元素值重新设置为element。代码逻辑：
1. 校验索引index的合法性。合法范围：(~,size]
2. 获取index处的旧值
3. 将列表的索引index处的值重新设置为新值
4. 返回旧值


##### remove源码

```java
public E remove(int index) {
    //校验index的合法性。范围为：(~,size]。之所以没有限制下限，是因为在调用elementData()方法的时候，会抛异常
    rangeCheck(index);

    //ArrayList数组修改的次数加1
    modCount++;

    E oldValue = elementData(index);

    int numMoved = size - index - 1;

    //如果待移除的元素不是最后一个，则实现逻辑是：将数组elementData的index位置之后的元素依次往前移动
    if (numMoved > 0)
        System.arraycopy(elementData, index+1, elementData, index,
                         numMoved);

    //将最后的位置置为null。
    elementData[--size] = null; // clear to let GC do its work

    return oldValue;
}
```
该方法是移除列表中索引index处的元素。代码逻辑：
1. 校验index的合法性。范围为：(~,size]
2. 修改modCount值，表示列表被修改一次
3. 获取索引index处的旧值
4. 判断待移除的元素是否位于列表尾；如果位于列表的最后一位，直接将列表最后一位置为空，然后修改size值；否则，将列表中index后的元素依次向前移动一位，最后将列表最后一位置为空并修改size值。
5. 返回旧值


##### remove源码

```java
//移除首次出现的元素o。实现逻辑为：首先在数组中找到元素o首次出现的索引index，
// 找到索引后，调用fastRemove()方法，移除index处的元素。实现逻辑与remove(int index)方法部分相似。都是将数组elementData的index位置之后的元素依次往前移动
public boolean remove(Object o) {
    if (o == null) {
        for (int index = 0; index < size; index++)
            if (elementData[index] == null) {
                fastRemove(index);
                return true;
            }
    } else {
        for (int index = 0; index < size; index++)
            if (o.equals(elementData[index])) {
                fastRemove(index);
                return true;
            }
    }
    return false;
}
```
该方法是移除列表中首次出现的元素o。由于ArrayList列表允许存储包括`null`在内的所有值，所以在确定元素o的位置时，会分两种情况进行。在找到元素o所在的首次出现的索引后，调用`fastRemove`方法删除元素。该方法逻辑与`remove(int index)`方法部分相似。

##### fastRemove源码

```java
 //与remove(int index)方法不同之处：跳过了范围检查，并且不用返回删除的元素
private void fastRemove(int index) {
    modCount++;
    int numMoved = size - index - 1;
    if (numMoved > 0)
        System.arraycopy(elementData, index+1, elementData, index,
                         numMoved);
    elementData[--size] = null; // clear to let GC do its work
}
```

##### removeRange源码

```java
//移除数组中[fromIndex, toIndex) 范围内的元素
protected void removeRange(int fromIndex, int toIndex) {
    modCount++;

    //待往前移动的元素个数。其中包含索引toIndex处的元素
    int numMoved = size - toIndex;

    //将数组elementData从索引toIndex处开始的元素依次往前移动。
    System.arraycopy(elementData, toIndex, elementData, fromIndex, numMoved);

    // clear to let GC do its work
    //清除移除的元素
    int newSize = size - (toIndex-fromIndex);
    for (int i = newSize; i < size; i++) {
        elementData[i] = null;
    }
    size = newSize;
}
```
源码实现，与上面差不多，此处不做分析，想看代码中的注释

##### removeAll源码

```java
public boolean removeAll(Collection<?> c) {
    //如果集合c为空，则抛异常'
    Objects.requireNonNull(c);
    return batchRemove(c, false);
}
```
内部调用了`batchRemove`方法，第二个参数传递了false；我们在看下类似的方法 `retainAll`。

##### retainAll源码

```java
public boolean retainAll(Collection<?> c) {
    //如果集合c为空，则抛异常
    Objects.requireNonNull(c);
    return batchRemove(c, true);
}
```
内部也是调用了了`batchRemove`方法，第二个参数传递了true。下面分析一下`batchRemove`方法。


##### batchRemove源码

```java
//该方法供removeAll(), 和retainAll()使用。在removeAll()时，complement=false，表示从调用者集合中删除包含c中的元素；
// 在retainAll()时，complement=true，表示调用者集合保留包含c中的元素；
private boolean batchRemove(Collection<?> c, boolean complement) {
    final Object[] elementData = this.elementData;
    int r = 0, w = 0;
    boolean modified = false;
    try {
        //遍历数组，根据complement的值，判断是保留数组elementData中包含集合c的元素还是删除集合c的元素
        for (; r < size; r++)
            if (c.contains(elementData[r]) == complement)
                elementData[w++] = elementData[r];
    } finally {
        // Preserve behavioral compatibility with AbstractCollection,
        // even if c.contains() throws.
        if (r != size) {
            System.arraycopy(elementData, r,
                             elementData, w,
                             size - r);
            w += size - r;
        }
        if (w != size) {
            // clear to let GC do its work
            for (int i = w; i < size; i++)
                elementData[i] = null;
            modCount += size - w;
            size = w;
           modified = true;
        }
    }
    return modified;
}
```
该方法的功能是需要根据参数`complement`的值，进行不同的操作。代码逻辑：
1. 获取方法调用者的内部存储数据
2. 遍历方法调用者的内部存储数据。如果`complement==true`，则保存方法调用者与集合c共有的元素；否则，则保存方法调用者存在，但是集合c中不存在的元素。
3. 如有需要，将方法调用者中的集合相关位置给置空。
4. 方法返回。

下图可以形象说明：

<a data-flickr-embed="true"  href="https://www.flickr.com/photos/157389715@N05/25978113808/in/dateposted-public/" title="1516695035317"><img src="https://farm5.staticflickr.com/4720/25978113808_5b9a55d02d_z.jpg" width="640" height="459" alt="1516695035317"></a>

##### clear源码

```java
//遍历数组，将元素全部置为null，并修改size为0
public void clear() {
    modCount++;

    // clear to let GC do its work
    for (int i = 0; i < size; i++)
        elementData[i] = null;

    size = 0;
}
```

##### toArray源码

```java
// 通过底层将数组中的元素拷贝到新数组中，然后返回
public Object[] toArray() {
    return Arrays.copyOf(elementData, size);
}

public <T> T[] toArray(T[] a) {
    //在数组a的长度小于目前数组的大小时，直接通过底层拷贝一份，类型为a的类型
    if (a.length < size)
        // Make a new array of a's runtime type, but my contents:
        return (T[]) Arrays.copyOf(elementData, size, a.getClass());

    //如果参数a的容量足够容下所有元素，则直接通过System.arraycopy进行复制
    System.arraycopy(elementData, 0, a, 0, size);

    //将数组中的多余位置置为null
    if (a.length > size)
        a[size] = null;

    return a;
}
```
这两个方法都是讲ArrayList列表转换为数组格式，并返回。当需要返回指定指定类型时，如果参数a的长度少于列表内的元素数，则通过`Arrays.copyOf`方法直接进行拷贝；否则通过`System.arraycopy`进行拷贝。


##### indexOf源码

```java
//返回参数 o 在数组中首次出现的索引
public int indexOf(Object o) {
    /**
     * 整体逻辑：
     *  如果参数o为null，则遍历数组，如果有null值，则返回首次出现的位置；
     *  如果参数o不为null。则遍历数组，如果有与参数o相同的元素，则返回首次出现的位置；
     */
    if (o == null) {
        for (int i = 0; i < size; i++)
            if (elementData[i]==null)
                return i;
    } else {
        for (int i = 0; i < size; i++)
            if (o.equals(elementData[i]))
                return i;
    }
    return -1;
}

//返回参数 o 在数组中最后出现的索引。与indexOf()方法不同之处为：lastIndexOf()方法是从数组尾部开始遍历；而ndexOf()方法是从数组头部开始遍历
public int lastIndexOf(Object o) {
    if (o == null) {
        for (int i = size-1; i >= 0; i--)
            if (elementData[i]==null)
                return i;
    } else {
        for (int i = size-1; i >= 0; i--)
            if (o.equals(elementData[i]))
                return i;
    }
    return -1;
}
```
这两个方法都是查找元素o在列表中国首次出现或者最后出现的索引

##### clone源码

```java
//浅拷贝
public Object clone() {
    try {
        ArrayList<?> v = (ArrayList<?>) super.clone();
        v.elementData = Arrays.copyOf(elementData, size);
        v.modCount = 0;
        return v;
    } catch (CloneNotSupportedException e) {
        // this shouldn't happen, since we are Cloneable
        throw new InternalError(e);
    }
}
```
该方法的功能是：通过浅拷贝，返回一个ArrayList实例的拷贝。

    浅拷贝：使用一个已知实例对新创建实例的成员变量逐个赋值，这个方式被称为浅拷贝。

    深拷贝：当一个类的拷贝构造方法，不仅要复制对象的所有非引用成员变量值，还要为引用类型的成员变量创建新的实例，并且初始化为形式参数实例值。这个方式称为深拷贝


所以，如果ArrayList中存储的是对象的引用时，修改拷贝出来的对象同样会影响原对象。

实例代码：
```java
public static void main(String[] args) {
    ArrayList<Map<String, String>> list = new ArrayList<>();
    Map<String, String> map = new HashMap<>();
    map.put("hah", "hah");
    list.add(map);

    ArrayList<Map<String, String>> list1 = (ArrayList<Map<String, String>>) list.clone();
    System.out.println("遍历复制体");
    for (int index = 0; index < list1.size(); index++) {
        System.out.println(JSONObject.toJSONString(list1.get(index)));
    }
    
    list1.get(0).put("xix", "xix");
    System.out.println("修改复制体");
    for (int index = 0; index < list1.size(); index++) {
        System.out.println(JSONObject.toJSONString(list1.get(index)));
    }

    System.out.println("遍历原型");
    for (int index = 0; index < list.size(); index++) {
        System.out.println(JSONObject.toJSONString(list1.get(index)));
    }

}

//输出
遍历复制体
{"hah":"hah"}
修改复制体
{"xix":"xix","hah":"hah"}
遍历原型
{"xix":"xix","hah":"hah"}

```


##### contains源码

```java
//判断元素o是否存在数组中，内部是调用indexOf()方法实现
public boolean contains(Object o) {
    return indexOf(o) >= 0;
}
```

##### isEmpty源码

```java
public boolean isEmpty() {
    return size == 0;
}
```

##### size源码

```java
public int size() {
    return size;
}
```

#### 迭代器

##### 获取迭代器源码

```java
public ListIterator<E> listIterator(int index) {
    if (index < 0 || index > size)
        throw new IndexOutOfBoundsException("Index: "+index);
    return new ListItr(index);
}

public ListIterator<E> listIterator() {
    return new ListItr(0);
}

public Iterator<E> iterator() {
    return new Itr();
}
```
在讲解 AbstractList 源码的时候，已对迭代器进行了相关分析，ArrayList 源码的迭代器与 AbstractList 的大致一样，就不在详细分析。只简单给出源码，参考相关注释。

##### 迭代器类

```java
 /**
 * An optimized version of AbstractList.Itr
 */
private class Itr implements Iterator<E> {
    //下一个返回元素的索引
    int cursor;       // index of next element to return
    //返回最后一个元素的索引
    int lastRet = -1; // index of last element returned; -1 if no such
    int expectedModCount = modCount;

    //判断cursor是否等于size
    public boolean hasNext() {
        return cursor != size;
    }

    @SuppressWarnings("unchecked")
    public E next() {
        //校验。ArrayList常见的坑是：使用forEach循环遍历数组中，有删除操作时，会报ConcurrentModificationException错误
        checkForComodification();
        int i = cursor;

        //如果超过了数组大小，则抛NoSuchElementException异常
        if (i >= size)
            throw new NoSuchElementException();
        Object[] elementData = ArrayList.this.elementData;

        //如果超过了数组元素数，则抛ConcurrentModificationException异常
        if (i >= elementData.length)
            throw new ConcurrentModificationException();

        //修改cursor的值。
        cursor = i + 1;
        //返回原索引cursor处的元素值
        return (E) elementData[lastRet = i];
    }

    //迭代器删除索引lastRet处的元素
    public void remove() {
        if (lastRet < 0)
            throw new IllegalStateException();
        checkForComodification();

        try {
            ArrayList.this.remove(lastRet);
            cursor = lastRet;
            lastRet = -1;
            expectedModCount = modCount;
        } catch (IndexOutOfBoundsException ex) {
            throw new ConcurrentModificationException();
        }
    }

    @Override
    @SuppressWarnings("unchecked")
    public void forEachRemaining(Consumer<? super E> consumer) {
        Objects.requireNonNull(consumer);
        final int size = ArrayList.this.size;
        int i = cursor;
        if (i >= size) {
            return;
        }
        final Object[] elementData = ArrayList.this.elementData;
        if (i >= elementData.length) {
            throw new ConcurrentModificationException();
        }
        while (i != size && modCount == expectedModCount) {
            consumer.accept((E) elementData[i++]);
        }
        // update once at end of iteration to reduce heap write traffic
        cursor = i;
        lastRet = i - 1;
        checkForComodification();
    }

    final void checkForComodification() {
        if (modCount != expectedModCount)
            throw new ConcurrentModificationException();
    }
}

/**
 * An optimized version of AbstractList.ListItr
 */
private class ListItr extends Itr implements ListIterator<E> {
    ListItr(int index) {
        super();
        cursor = index;
    }

    //向前遍历时，cursor不能为0
    public boolean hasPrevious() {
        return cursor != 0;
    }

    public int nextIndex() {
        return cursor;
    }

    public int previousIndex() {
        return cursor - 1;
    }

    @SuppressWarnings("unchecked")
    public E previous() {
        checkForComodification();
        int i = cursor - 1;
        if (i < 0)
            throw new NoSuchElementException();
        Object[] elementData = ArrayList.this.elementData;
        if (i >= elementData.length)
            throw new ConcurrentModificationException();
        cursor = i;
        return (E) elementData[lastRet = i];
    }

    public void set(E e) {
        if (lastRet < 0)
            throw new IllegalStateException();
        checkForComodification();

        try {
            ArrayList.this.set(lastRet, e);
        } catch (IndexOutOfBoundsException ex) {
            throw new ConcurrentModificationException();
        }
    }

    public void add(E e) {
        checkForComodification();

        try {
            int i = cursor;
            ArrayList.this.add(i, e);
            cursor = i + 1;
            lastRet = -1;
            expectedModCount = modCount;
        } catch (IndexOutOfBoundsException ex) {
            throw new ConcurrentModificationException();
        }
    }
}

```



至此，List接口的一个重要的子类ArrayList的相关源码我们已经分析完，下面将分析List接口的另一个重要的子类LinkedList。

