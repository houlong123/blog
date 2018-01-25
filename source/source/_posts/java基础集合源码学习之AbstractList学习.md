---
title: java基础集合源码学习之AbstractList学习
date: 2018-01-19 15:30:08
tags: java基础集合
---

#### 前言
在日常编程中，ArrayList是使用最多的列表。在上篇文章 [AbstractCollection学习](https://houlong123.github.io/2018/01/17/java基础集合源码学习之AbstractCollection学习/) 的学习中，我们可以从`java基础集合框架图`中得知：ArrayList继承了`AbstractList`，并实现了`List`接口。而其中 AbstractList 则继承了`AbstractCollection`。

#### 内部数据结构

##### 数据结构
```java
public abstract class AbstractList<E> extends AbstractCollection<E> implements List<E> {
    
    // ArrayList集合的修改次数
    protected transient int modCount = 0;
    
    //迭代器类
    private class Itr implements Iterator<E> {}
    
    private class ListItr extends Itr implements ListIterator<E> {}
}
```
AbstractList继承了AbstractCollection抽象类，并实现了List接口。关于AbstractCollection已在[另篇文章](https://houlong123.github.io/2018/01/17/java基础集合源码学习之AbstractCollection学习/)讲解。关于List接口，在下文有简述。AbstractList中唯一的一个属性为`modCount`，表示集合被修改的次数。在AbstractList的子类中，凡是涉及到修改列表的内部数据结构的方法，都是修改`modCount`的值。

##### List 源码分析

<!-- more -->

###### 数据结构

```java
public interface List<E> extends Collection<E> {
    
}
```
由类的声明可知，List接口继承了java集合框架的超类 Collection，该类中定义了列表的所有基本操作。

###### 基本方法

```java
public interface List<E> extends Collection<E> {
    
    //添加
    boolean add(E e);
    void add(int index, E element);
    boolean addAll(Collection<? extends E> c);
    boolean addAll(int index, Collection<? extends E> c);
    
    //修改
    E set(int index, E element);
    
    //移除
    E remove(int index);
    boolean remove(Object o);
    boolean removeAll(Collection<?> c);
    void clear();
    
    
    //查找
    E get(int index);
    boolean containsAll(Collection<?> c);
    int indexOf(Object o);
    int lastIndexOf(Object o);
    
    //基本操作
    int size();
    boolean isEmpty();
    boolean contains(Object o);
    Object[] toArray();
    <T> T[] toArray(T[] a);
    boolean retainAll(Collection<?> c);
    boolean equals(Object o);
    int hashCode();
    List<E> subList(int fromIndex, int toIndex);
    
    //排序
    default void sort(Comparator<? super E> c) {
        Object[] a = this.toArray();
        Arrays.sort(a, (Comparator) c);
        ListIterator<E> i = this.listIterator();
        for (Object e : a) {
            i.next();
            i.set((E) e);
        }
    }
    
    //迭代器
    Iterator<E> iterator();
    ListIterator<E> listIterator();
    ListIterator<E> listIterator(int index);
}
```
由List的源码可知，在接口List中，定义了列表的所有基本操作，比如常见的`增删改查`。还有用于获取遍历列表的迭代器方法。具体实现逻辑，需有子类根据自身需求去实现。


##### 添加方法

```java
//模板方法，内部调用add(int,E)方法，默认是抛异常，具体逻辑由子类实现
public boolean add(E e) {
    add(size(), e);
    return true;
}
    
public boolean addAll(int index, Collection<? extends E> c) {
    //校验index是否有误,索引的index有效范围为：[0, size]
    rangeCheckForAdd(index);

    boolean modified = false;

    //依次遍历参数集合c，将元素添加进调用者集合
    for (E e : c) {
        add(index++, e);
        modified = true;
    }
    return modified;
}

public void add(int index, E element) {
    throw new UnsupportedOperationException();
}
```
AbstractList 中有三个涉及到列表添加元素的方法。其中最底层都是使用声明为: `add(int index, E element)` 的方法。有代码实现可知，该方法的实现是抛出异常。其具体实现逻辑交由子类实现。

##### rangeCheckForAdd源码

```java
private void rangeCheckForAdd(int index) {
    if (index < 0 || index > size())
        throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
}
```
该方法用于添加元素位置进行校验，插入元素的有效范围是[0, size]。

##### 查找元素

###### get源码

```java
abstract public E get(int index);
```
抽象方法，内部逻辑由子类实现。在AbstractList的两个常用实现类：ArrayList 和 LinkedList ，因其内部数据结构的不同，导致其内部实现逻辑完全不同。

###### indexOf源码

```java
//获取参数o在集合中首次出现的索引
public int indexOf(Object o) {

    //ListIterator迭代器既可以向后遍历，也可以向前遍历
    ListIterator<E> it = listIterator();

    //使用迭代器遍历集合，集合中包含元素o，则返回索引
    if (o==null) {
        while (it.hasNext())
            if (it.next()==null)
                return it.previousIndex();
    } else {
        while (it.hasNext())
            if (o.equals(it.next()))
                return it.previousIndex();
    }
    return -1;
}
```
indexOf() 方法的功能：获取参数o在集合中首次出现的索引。具体实现逻辑就是获取向后的迭代器，从列表的头部开始查找，找到后，返回所处的索引。

###### lastIndexOf源码

```java
//获取参数o在集合中最后出现的索引
public int lastIndexOf(Object o) {
    ListIterator<E> it = listIterator(size());
    if (o==null) {
        while (it.hasPrevious())
            if (it.previous()==null)
                return it.nextIndex();
    } else {
        while (it.hasPrevious())
            if (o.equals(it.previous()))
                return it.nextIndex();
    }
    return -1;
}
```
lastIndexOf 方法的功能与 indexOf 方法类似，只不过lastIndexOf 方法是获取参数o在集合中最后出现的索引。


##### 修改元素

```java
public E set(int index, E element) {
    throw new UnsupportedOperationException();
}
```
与add方法一样，也是抽象方法。

##### 删除元素

###### remove源码
```java
 public E remove(int index) {
    throw new UnsupportedOperationException();
}
```
由源码可知，该方法同add，get，set方法一样，都是抽象方法。

###### clear源码

```java
public void clear() {
    removeRange(0, size());
}
```

###### removeRange方法

```java

//移除[fromIndex, toIndex)之间的元素
protected void removeRange(int fromIndex, int toIndex) {
    ListIterator<E> it = listIterator(fromIndex);
    for (int i=0, n=toIndex-fromIndex; i<n; i++) {
        it.next();
        it.remove();
    }
}
```
该方法是移除列表中[fromIndex, toIndex) 范围内的元素。移除操作是通过迭代器的移除功能实现的。


##### 获取迭代器

```java
//获取迭代器。从列表头向后开始遍历
public Iterator<E> iterator() {
    return new Itr();
}

//获取迭代器。从列表尾向前开始遍历
public ListIterator<E> listIterator() {
    return listIterator(0);
}

public ListIterator<E> listIterator(final int index) {
    rangeCheckForAdd(index);

    return new ListItr(index);
}
```
由 AbstractList 的源码可知，其子类会有两种遍历列表中元素的方法。一种是通过 `iterator()` 获取只能从头部开始向后遍历元素的迭代器Iterator对象；一种通过 `listIterator()` 获取既可以从头部开始遍历也可以从尾部开始遍历的ListIterator对象。

##### 基本方法
###### subList源码

```java
public List<E> subList(int fromIndex, int toIndex) {
    return (this instanceof RandomAccess ?
            new RandomAccessSubList<>(this, fromIndex, toIndex) :
            new SubList<>(this, fromIndex, toIndex));
}
```
从代码实现可知，获取列表的子列表其实是创建了另一个类。

###### equels源码

```java
public boolean equals(Object o) {
    if (o == this)
        return true;
    if (!(o instanceof List))
        return false;

    ListIterator<E> e1 = listIterator();
    ListIterator<?> e2 = ((List<?>) o).listIterator();
    while (e1.hasNext() && e2.hasNext()) {
        E o1 = e1.next();
        Object o2 = e2.next();
        if (!(o1==null ? o2==null : o1.equals(o2)))
            return false;
    }
    return !(e1.hasNext() || e2.hasNext());
}
```
该方法是判断两个对象是否相等。前提是两个对象都是列表类型。判断的逻辑是：分别获取两个列表对象的迭代器，然后依次判断各个元素是否相等。

###### hashCode源码

```java
 public int hashCode() {
    int hashCode = 1;
    for (E e : this)
        hashCode = 31*hashCode + (e==null ? 0 : e.hashCode());
    return hashCode;
}
```


##### 迭代器源码分析
在AbstractList的方法中，有三个获取迭代器的方法。其中`iterator()`方法是获取`Iterator`类型的迭代器。该迭代器只能从列表的头部开始遍历。在分析`iterator()`方法的源码时，知道内部是创建了一个名为`Itr`的类。

###### Itr 源码

```java
private class Itr implements Iterator<E> {
    /**
     * Index of element to be returned by subsequent call to next.
     */
    //随后调用下一个元素的索引。当调用next()方法时，返回当前索引的值
    int cursor = 0;

    /**
     * Index of element returned by most recent call to next or
     * previous.  Reset to -1 if this element is deleted by a call
     * to remove.
     */
    //最近调用next()或previous()的返回元素索引。如果删掉此元素，该值置为-1
    int lastRet = -1;

    /**
     * The modCount value that the iterator believes that the backing
     * List should have.  If this expectation is violated, the iterator
     * has detected concurrent modification.
     */
    int expectedModCount = modCount;

    //判断是否还有元素。向后遍历时，cursor不能等于集合大小
    public boolean hasNext() {
        return cursor != size();
    }

    //在调用next时，直接取出集合中cursor位置的元素，然后修改lastRet和cursor值
    public E next() {
        checkForComodification();
        try {
            int i = cursor;
            E next = get(i);
            lastRet = i;
            cursor = i + 1;
            return next;
        } catch (IndexOutOfBoundsException e) {
            checkForComodification();
            throw new NoSuchElementException();
        }
    }

    //移除lastRet位置的元素
    public void remove() {
        if (lastRet < 0)
            throw new IllegalStateException();
        checkForComodification();

        try {
            AbstractList.this.remove(lastRet);
            if (lastRet < cursor)
                cursor--;
            lastRet = -1;
            expectedModCount = modCount;
        } catch (IndexOutOfBoundsException e) {
            throw new ConcurrentModificationException();
        }
    }

    //检查modCount是否改变
    final void checkForComodification() {
        if (modCount != expectedModCount)
            throw new ConcurrentModificationException();
    }
}
```
由内部源码可知，Itr 内部其实是维持了一个名为`cursor`的属性。该属性是调用下一个元素的索引。即：当调用next()方法时，返回当前索引的值。名为`lastRet`的属性，表示最近调用next()或previous()的返回元素索引。如果删掉此元素，该值置为-1。因此可知，在调用next获取下个元素值时，是返回索引`cursor`处的元素；在调用remove删除元素时，会删除索引`lastRet`处的元素。还有一处需要注意的是：在迭代器无论调用哪个方法，在调用之前都会先调用`checkForComodification()`方法进行校验，校验通过后，才会有后续操作。checkForComodification方法会抛出`ConcurrentModificationException`异常。这是我们在使用列表的时候可能会遇到的坑。出现这个异常是由于：在我们创建迭代器的时候，会有`expectedModCount = modCount`这个赋值操作，如果在遍历的过程中，列表的数据结构被修改了，如进行了列表元素的添加或删除，会导致 modCount的值发生变化，所以会导致异常。<font color=red>如果想实现在使用迭代器遍历的过程中既可以修改列表的数据结构又不会产生异常，可以使用迭代器的相关方法。如add，或remove</font>。


###### ListItr 源码

AbstractList中的`listIterator()`方法是获取`ListIterator`类型的迭代器。该迭代器即可以从前往后遍历，也可以从后往前遍历。`listIterator()`方法内部是创建了`ListItr`对象。

```java
private class ListItr extends Itr implements ListIterator<E> {
    ListItr(int index) {
        cursor = index;
    }

    //向前遍历时，cursor不能为0
    public boolean hasPrevious() {
        return cursor != 0;
    }

    public E previous() {
        checkForComodification();
        try {
            int i = cursor - 1;
            E previous = get(i);
            lastRet = cursor = i;
            return previous;
        } catch (IndexOutOfBoundsException e) {
            checkForComodification();
            throw new NoSuchElementException();
        }
    }

    public int nextIndex() {
        return cursor;
    }

    public int previousIndex() {
        return cursor-1;
    }

    public void set(E e) {
        if (lastRet < 0)
            throw new IllegalStateException();
        checkForComodification();

        try {
            AbstractList.this.set(lastRet, e);
            expectedModCount = modCount;
        } catch (IndexOutOfBoundsException ex) {
            throw new ConcurrentModificationException();
        }
    }

    public void add(E e) {
        checkForComodification();

        try {
            int i = cursor;
            AbstractList.this.add(i, e);
            lastRet = -1;
            cursor = i + 1;
            expectedModCount = modCount;
        } catch (IndexOutOfBoundsException ex) {
            throw new ConcurrentModificationException();
        }
    }
}
```
该类继承了 Itr类，并提供了一种从列表尾部向前遍历的方法。并且添加了修改和添加元素的方法。


##### SubList源码
在调用AbstractList类的subList()方法时，其内部返回了SubList或者RandomAccessSubList类。

###### RandomAccessSubList源码

```java
class RandomAccessSubList<E> extends SubList<E> implements RandomAccess {
    RandomAccessSubList(AbstractList<E> list, int fromIndex, int toIndex) {
        super(list, fromIndex, toIndex);
    }

    public List<E> subList(int fromIndex, int toIndex) {
        return new RandomAccessSubList<>(this, fromIndex, toIndex);
    }
}
```
由RandomAccessSubList源码可知，RandomAccessSubList类继承了SubList类。

###### SubList源码

```java
//SubList内部是通过维护一个AbstractList对象来进行相应操作。并包含偏移值offset和SubList的大小
class SubList<E> extends AbstractList<E> {

    /* 从SubList源码可以看出，当需要获得一个子List时，底层并不是真正的返回一个子List，还是原来的List，只不过
    * 在操作的时候，索引全部限定在用户所需要的子List部分而已
    */
    private final AbstractList<E> l;
    private final int offset;
    private int size;

    SubList(AbstractList<E> list, int fromIndex, int toIndex) {
        if (fromIndex < 0)
            throw new IndexOutOfBoundsException("fromIndex = " + fromIndex);
        if (toIndex > list.size())
            throw new IndexOutOfBoundsException("toIndex = " + toIndex);
        if (fromIndex > toIndex)
            throw new IllegalArgumentException("fromIndex(" + fromIndex +
                                               ") > toIndex(" + toIndex + ")");
        l = list;
        offset = fromIndex;
        size = toIndex - fromIndex;
        this.modCount = l.modCount;
    }

    //注意下面所有的操作都在索引上加上偏移量offset，相当于在原来List的副本上操作子List
    public E set(int index, E element) {
        rangeCheck(index);
        checkForComodification();
        return l.set(index+offset, element);
    }

    public E get(int index) {
        rangeCheck(index);
        checkForComodification();
        return l.get(index+offset);
    }

    public int size() {
        checkForComodification();
        return size;
    }

    public void add(int index, E element) {
        rangeCheckForAdd(index);
        checkForComodification();
        l.add(index+offset, element);
        this.modCount = l.modCount;
        size++;
    }

    public E remove(int index) {
        rangeCheck(index);
        checkForComodification();
        E result = l.remove(index+offset);
        this.modCount = l.modCount;
        size--;
        return result;
    }

    protected void removeRange(int fromIndex, int toIndex) {
        checkForComodification();
        l.removeRange(fromIndex+offset, toIndex+offset);
        this.modCount = l.modCount;
        size -= (toIndex-fromIndex);
    }

    public boolean addAll(Collection<? extends E> c) {
        return addAll(size, c);
    }

    public boolean addAll(int index, Collection<? extends E> c) {
        rangeCheckForAdd(index);
        int cSize = c.size();
        if (cSize==0)
            return false;

        checkForComodification();
        l.addAll(offset+index, c);
        this.modCount = l.modCount;
        size += cSize;
        return true;
    }

    public Iterator<E> iterator() {
        return listIterator();
    }

    public ListIterator<E> listIterator(final int index) {
        checkForComodification();
        rangeCheckForAdd(index);

        return new ListIterator<E>() {
            private final ListIterator<E> i = l.listIterator(index+offset);

            public boolean hasNext() {
                return nextIndex() < size;
            }

            public E next() {
                if (hasNext())
                    return i.next();
                else
                    throw new NoSuchElementException();
            }

            public boolean hasPrevious() {
                return previousIndex() >= 0;
            }

            public E previous() {
                if (hasPrevious())
                    return i.previous();
                else
                    throw new NoSuchElementException();
            }

            public int nextIndex() {
                return i.nextIndex() - offset;
            }

            public int previousIndex() {
                return i.previousIndex() - offset;
            }

            public void remove() {
                i.remove();
                SubList.this.modCount = l.modCount;
                size--;
            }

            public void set(E e) {
                i.set(e);
            }

            public void add(E e) {
                i.add(e);
                SubList.this.modCount = l.modCount;
                size++;
            }
        };
    }

    public List<E> subList(int fromIndex, int toIndex) {
        return new SubList<>(this, fromIndex, toIndex);
    }

    private void rangeCheck(int index) {
        if (index < 0 || index >= size)
            throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
    }

    private void rangeCheckForAdd(int index) {
        if (index < 0 || index > size)
            throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
    }

    private String outOfBoundsMsg(int index) {
        return "Index: "+index+", Size: "+size;
    }

    private void checkForComodification() {
        if (this.modCount != l.modCount)
            throw new ConcurrentModificationException();
    }
}
```
由SubList源码可知，其内部维持了一个AbstractList对象和偏移量offset及SubList对象内存储的元素数。从SubList源码可以看出，当需要获得一个子List时，底层并不是真正的返回一个子List，还是原来的List，只不过在操作的时候，索引全部限定在用户所需要的子List部分而已。

