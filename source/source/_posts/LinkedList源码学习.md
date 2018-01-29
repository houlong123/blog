---
title: LinkedList源码学习
date: 2018-01-26 17:56:18
tags: java基础集合
---




#### 前言
在上篇文章 [ArrayList源码学习](https://houlong123.github.io/2018/01/23/ArrayList源码学习/) 中，我们队 ArrayList 的源码进行了相应的分析。在学习的过程中，我们已经知道，`ArrayList的底层是数组结构`。所有操作都是针对数组进行的。在本篇中，我们学习一下List接口的另一个重要的子类LinkedList。最后在总结一下二者的区别。

#### 源码解析

##### 数据结构

```java
//LinkedList是双链表结构，继承了List和Deque接口
public class LinkedList<E>
    extends AbstractSequentialList<E>
    implements List<E>, Deque<E>, Cloneable, java.io.Serializable {
    
    //元素个数
    transient int size = 0;
    //双链表的头结点。不变性：(first == null && last == null) || (first.prev == null && first.item != null)
    transient Node<E> first;
    //双链表的尾结点。不变性：(first == null && last == null) || (last.next == null && last.item != null)
    transient Node<E> last;
    
    //迭代器
    private class ListItr implements ListIterator<E> {}
    
    //内部节点类
    //列表内部节点类
    private static class Node<E> {}
}
```
从LinkedList的源码中，我们可以发现：LinkedList 不仅继承了List接口，而且还继承了Deque，即：Queue的子类。因此可知，LinkedList不仅可以当做列表使用，而且还能实现队列的功能。

与`ArrayList底层数组结构`不同的是，`LinkedList的底层数据结构是链接`，即由节点构成。底层数据结构的差异自然而然就会导致对存储对象的操作差异。

###### Node节点类

<!--more-->

```java
//列表内部节点类
private static class Node<E> {
    E item;
    Node<E> next;
    Node<E> prev;

    Node(Node<E> prev, E element, Node<E> next) {
        this.item = element;
        this.next = next;
        this.prev = prev;
    }
}
```
LinkedList 的底层数据结构为链表。而链表是由一个个节点连接而成的。LinkedList内部的Node节点类很简单，内部只有三个属性：

属性名|描述
---|---|---|
item|当前节点中存储的元素值
next|当前节点的子节点
prev|当前节点的父节点

在上面分析LinkedList的数据结构时，我们可以看到LinkedList 中有两个重要属性：

```java
//双链表的头结点。不变性：(first == null && last == null) || (first.prev == null && first.item != null)
transient Node<E> first;
//双链表的尾结点。不变性：(first == null && last == null) || (last.next == null && last.item != null)
transient Node<E> last;
```

##### 构造函数

```java
public LinkedList() {
}

public LinkedList(Collection<? extends E> c) {
    this();
    addAll(c);
}
```
和 ArrayList 的 构造函数不同，LinkedList 的构造函数内部没有做任何操作。由于是链表结构，所以也就没有初始化容量大小这一概念，所以也就不会像 ArrayList 那样在构造函数中去初始化底层数据结构 。


##### addAll源码

```java
public boolean addAll(Collection<? extends E> c) {
    return addAll(size, c);
}

public boolean addAll(int index, Collection<? extends E> c) {
    //插入位置校验。合法范围：[0, size]
    checkPositionIndex(index);

    //如果待插入集合内没有元素，则直接返回FALSE
    Object[] a = c.toArray();
    int numNew = a.length;
    if (numNew == 0)
        return false;

    //确定首次新插入节点的父节点和最后插入节点的子节点
    Node<E> pred, succ;
    if (index == size) {    //index == size，说明是从链表的尾部进行集合元素的插入，所以首次新插入节点的父节点为last；最后插入节点的子节点为null
        succ = null;
        pred = last;
    } else {    //index != size，说明是从链表的中间进行集合元素的插入，所以首次新插入节点的父节点为索引index处的元素的父节点；最后插入节点的子节点为索引index处的元素
        succ = node(index);
        pred = succ.prev;
    }

    //遍历集合c中的元素，并依次链接到pred的next节点
    for (Object o : a) {
        @SuppressWarnings("unchecked") E e = (E) o;
        Node<E> newNode = new Node<>(pred, e, null);
        if (pred == null)
            first = newNode;
        else
            pred.next = newNode;
        pred = newNode;
    }

    //集合c中的元素遍历完后，设置链表的尾节点。此时pred是集合c中最后一个元素组成的节点
    if (succ == null) {
        last = pred;
    } else {
        pred.next = succ;
        succ.prev = pred;
    }

    size += numNew;
    modCount++;
    return true;
}
```
在 LinkedList 的有参构造函数中，内部会调用`addAll()`方法将集合中的元素构造一个 LinkedList 对象。addAll方法的整理逻辑为：根据待插入位置，找到待插入元素的父节点与子节点，然后设置节点的引用即可。

addAll 方法的逻辑为：
1. 校验插入元素的位置。合法范围：[0, size]
2. 判断集合中是否有元素，没有的话，直接返回false；否则执行下一步。
3. 确定首次新插入节点的父节点和最后插入节点的子节点
4. 遍历集合c中的元素，并依次链接到pred的next节点
5. 集合c中的元素遍历完后，设置链表的尾节点。
6. 修改size值及modCount值，然后返回。

##### checkPositionIndex源码

```java
private void checkPositionIndex(int index) {
    if (!isPositionIndex(index))
        throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
}

private boolean isPositionIndex(int index) {
    return index >= 0 && index <= size;
}
```
该方法是用来校验指定索引index是否合法。合法范围为：[0, size]。

##### node源码

```java
//返回索引index处的Node节点。类似于折半查找。在index大于链表数的一半时，从尾节点开始查找；否则，从头节点开始查找
Node<E> node(int index) {
    // assert isElementIndex(index);

    if (index < (size >> 1)) {
        Node<E> x = first;
        for (int i = 0; i < index; i++)
            x = x.next;
        return x;
    } else {
        Node<E> x = last;
        for (int i = size - 1; i > index; i--)
            x = x.prev;
        return x;
    }
}
```
在指定位置index处插入元素时，所以需要先把链表中index处的元素oldElement查找出来。将指定元素插入的逻辑就是设置新插入节点的next节点为oldElement；将oldElement的父节点设置为新插入节点。

方法`node()` 的功能就是查找index处的节点并返回。其内部的实现逻辑有点类似于折半查找法：如果index少于元素个数的一半，则从链表的头节点开始往后查找；否则，从尾节点开始向前查找。

##### 添加元素

###### addFirst源码

```java
public void addFirst(E e) {
    linkFirst(e);
}
```
该方法是从链表的头部插入元素e。即把元素e链接到链表的头部。

###### linkFirst源码

```java
//在链表头部添加元素
private void linkFirst(E e) {
    //获取链表头部指针
    final Node<E> f = first;
    //将元素组装成一个Node节点
    final Node<E> newNode = new Node<>(null, e, f);

    //将first指针指向新的节点
    first = newNode;

    //如果双链表的头节点为null，则说明该值为列表中首次添加的元素，此时将链表的头尾节点都指向同一元素。
    if (f == null)
        last = newNode;
    else
        f.prev = newNode;

    //修改size和modCount
    size++;
    modCount++;
}
```
如果了解链表的相关操作的话，该方法其实很简单。就是将链表的头节点的引用重新指到新节点上；设置好头结点后，如果原头结点为null，说明新节点为链表总的第一次节点，所以链表的尾节点也需要指向新节点；否则，将原头结点的父节点设置为新节点。

> 元素添加

+ 队列初始态
<a data-flickr-embed="true"  href="https://www.flickr.com/photos/157389715@N05/28123993599/in/dateposted-public/" title="链表初始态"><img src="https://farm5.staticflickr.com/4615/28123993599_9ee81c3faf.jpg" width="487" height="302" alt="链表初始态"></a>

+ 新元素入队
<a data-flickr-embed="true"  href="https://www.flickr.com/photos/157389715@N05/26031363388/in/dateposted-public/" title="新元素添加"><img src="https://farm5.staticflickr.com/4706/26031363388_74213ec72e.jpg" width="500" height="233" alt="新元素添加"></a>


###### addLast源码

```java
public void addLast(E e) {
    linkLast(e);
}
```
同  addFirst方法 的逻辑差不多，该方法是在链表的尾节点添加元素。

###### linkLast源码

```java
//在链表尾部添加元素
void linkLast(E e) {
    //双链表的尾结点
    final Node<E> l = last;
    //将元素组装成一个Node节点
    final Node<E> newNode = new Node<>(l, e, null);

    //将last指针指向新的节点
    last = newNode;

    //如果双链表的尾节点为null，则说明该值为列表中首次添加的元素，此时将链表的头尾节点都指向同一元素。
    if (l == null)
        first = newNode;
    else
        l.next = newNode;

    //修改size和modCount
    size++;
    modCount++;
}
```

###### add源码

```java
//在列表的尾部添加元素e
public boolean add(E e) {
    linkLast(e);
    return true;
}
```
同直接调用 addLast() 方法逻辑一样，都是在链表的尾部添加元素。

###### 重载add源码

```java
public void add(int index, E element) {
    checkPositionIndex(index);

    //说明在链表的尾部添加元素，相当于直接调用linkLast
    if (index == size)
        linkLast(element);
    else
        linkBefore(element, node(index));
}
```
该方法的功能是在指定索引index处添加元素。在添加之前会先调用`checkPositionIndex`方法判断index是否合法；合法的情况下，如果`index=size`，则说明在链表的尾部添加元素，则直接调用`linkLast`方法进行元素添加；否则，先查找出index出的元素succ，然后调用`linkBefore`方法在元素succ之前添加节点。

###### linkBefore源码

```java
//在节点succ之前添加元素
void linkBefore(E e, Node<E> succ) {
    // assert succ != null;
    //获取节点succ的父结点
    final Node<E> pred = succ.prev;
    //组装节点
    final Node<E> newNode = new Node<>(pred, e, succ);

    //设置节点succ的父节点为新节点
    succ.prev = newNode;

    //如果succ的父结点为null，说明是在链表的头部插入了新元素
    if (pred == null)
        first = newNode;
    else
        pred.next = newNode;
    size++;
    modCount++;
}
```

> 指定位置元素添加

+ 队列初始态

<a data-flickr-embed="true"  href="https://www.flickr.com/photos/157389715@N05/28123993599/in/dateposted-public/" title="链表初始态"><img src="https://farm5.staticflickr.com/4615/28123993599_9ee81c3faf.jpg" width="487" height="302" alt="链表初始态"></a>

+ 新元素入队

<a data-flickr-embed="true"  href="https://www.flickr.com/photos/157389715@N05/28124193199/in/dateposted-public/" title="指定位置插入元素"><img src="https://farm5.staticflickr.com/4755/28124193199_db0fc91d86.jpg" width="500" height="393" alt="指定位置插入元素"></a>


###### push源码

```java
//从头部添加元素
public void push(E e) {
    addFirst(e);
}
```
该方法是在队列的头部添加元素。在分析LinkedList的数据结构的时候，我们已经说过，LinkedList 不仅继承了 List接口，而且还继承了 Queue 接口，因此 LinkedList 具有队列的特征。其中该方法就是模拟队列添加元素的。


###### offer源码

```java
//从尾部添加元素
public boolean offer(E e) {
    return add(e);
}
```
该方法是在队列的尾部添加元素。

###### offerFirst源码

```java
//从头部添加元素
public boolean offerFirst(E e) {
    addFirst(e);
    return true;
}
```

###### offerLast源码

```java
//从尾部添加元素
public boolean offerLast(E e) {
    addLast(e);
    return true;
}
```





##### 删除元素

###### removeFirst源码

```java
//移除链表中的头结点
public E removeFirst() {
    final Node<E> f = first;
    if (f == null)
        throw new NoSuchElementException();
    return unlinkFirst(f);
}
```
该方法是从链表中移除头结点。如果列表为空时，会报错；否则，调用`unlinkFirst()`方法将节点从链表移除。


###### unlinkFirst源码

```java
//与unlinkLast()方法类似， 但是移除的是非空的头节点l
private E unlinkFirst(Node<E> f) {
    // assert f == first && f != null;
    final E element = f.item;
    final Node<E> next = f.next;
    f.item = null;
    f.next = null; // help GC
    first = next;
    if (next == null)
        last = null;
    else
        next.prev = null;
    size--;
    modCount++;
    return element;
}
```
该方法的功能：从链表中移除头节点f，并返回头节点f中存储的item值。大体逻辑为：首先获取头节点f的item值和next子节点A。然后将这两个域置为null；然后重新设置链表的头结点为A；如果A为null，说明队列中已经没有其他节点，则设置尾节点为null；否则，将节点A的父节点置为null，最后返回头结点f的item域值。

###### removeLast源码

```java
//移除链表中的尾节点
public E removeLast() {
    final Node<E> l = last;
    if (l == null)
        throw new NoSuchElementException();
    return unlinkLast(l);
}
```
该方法的功能为：从链表的尾部移除节点。

###### unlinkLast源码

```java
//移除非空的尾节点l
private E unlinkLast(Node<E> l) {
    // assert l == last && l != null;
    //获取节点l内的item值
    final E element = l.item;
    //获取节点l的父节点prev
    final Node<E> prev = l.prev;
    //将节点l的域值，父节点置空，用于GC
    l.item = null;
    l.prev = null; // help GC

    //重新赋值last节点
    last = prev;

    //如果父节点prev为空，说明删除的为链表中的最后一个元素；否则将父节点prev的next置空
    if (prev == null)
        first = null;
    else
        prev.next = null;

    //修改size和modCount值
    size--;
    modCount++;
    return element;
}
```
该方法与`unlinkFirst`方法大致相同，不在细述，详情看代码注释。

###### remove源码

```java
//从链表中删除元素o
public boolean remove(Object o) {
    //整体逻辑：遍历链表中的所有节点，如果找到节点的item值等于元素o，则调用unlink()方法将该节点移除
    if (o == null) {
        for (Node<E> x = first; x != null; x = x.next) {
            if (x.item == null) {
                unlink(x);
                return true;
            }
        }
    } else {
        for (Node<E> x = first; x != null; x = x.next) {
            if (o.equals(x.item)) {
                unlink(x);
                return true;
            }
        }
    }
    return false;
}
```
该方法是从链表中移除元素o（移除首次出现的）。大致逻辑：从头结点开始遍历链表，如果找到元素o对应的节点A，则调用`unlink`方法删除节点A。

###### unlink源码

```java
//移除非空节点
E unlink(Node<E> x) {
    // assert x != null;
    //获取节点x的item值，父节点prev及子节点next
    final E element = x.item;
    final Node<E> next = x.next;
    final Node<E> prev = x.prev;

    //如果节点x的父节点为null。则说明节点x为头结点，则直接将first指向节点x的next节点
    if (prev == null) {
        first = next;
    } else {    //如果节点x的父节点不为null。则需要重新设置节点x的父节点prev的子节点next指向节点x的子节点；节点x的父节点prev置为空。即：将节点x与其父节点断开连接
        prev.next = next;
        x.prev = null;
    }

    //如果节点x的子节点为null。则说明节点x为尾结点，则直接将last指向节点x的prev节点
    if (next == null) {
        last = prev;
    } else {    //如果节点x的子节点不为null。则需要重新设置节点x的子节点next的父节点prev指向节点x的父节点；节点x的子节点next置为空。即：将节点x与其子节点断开连接
        next.prev = prev;
        x.next = null;
    }

    //将节点x的item值置为空
    x.item = null;

    //修改size和modCount值
    size--;
    modCount++;
    return element;
}
```
该方法是从链表中删除节点x。代码逻辑：
1. 获取待删除节点x的父节点prev，子节点next及item值element。
2. 重新设置待删除节点x的父子节点的引用
3. 置空待删除节点的相关属性
4. 返回待删除节点的item值。

###### remove源码

```java
//remove(Object o)方法的简化版
public E remove(int index) {
    checkElementIndex(index);
    return unlink(node(index));
}
```
找到index处的节点，然后删除。

###### remove源码

```java
//从链表的头部移除元素，与poll()方法不同之处在于：链表中没有元素时，poll()方法返回null，而remove()抛异常
public E remove() {
    return removeFirst();
}
```
从链表的头节点删除节点。

###### clear源码

```java
public void clear() {
    // Clearing all of the links between nodes is "unnecessary", but:
    // - helps a generational GC if the discarded nodes inhabit
    //   more than one generation
    // - is sure to free memory even if there is a reachable Iterator

    //从头结点开始遍历，依次将节点的item，prev和next值都置为空
    for (Node<E> x = first; x != null; ) {
        Node<E> next = x.next;
        x.item = null;
        x.next = null;
        x.prev = null;
        x = next;
    }
    first = last = null;
    size = 0;
    modCount++;
}
```
该方法的功能：清空列表中的所有元素。

###### poll源码

```java
//从链表的头部移除元素
public E poll() {
    final Node<E> f = first;
    return (f == null) ? null : unlinkFirst(f);
}
```
LinkedList在扮演队列的角色时，`push`方法用于往队列中添加元素，`poll`方法用于从队列中移除元素。这两个方法一起用的时候，可以充当`先进先出队列`。

###### pollFirst源码

```java
//头部删除节点
public E pollFirst() {
    final Node<E> f = first;
    return (f == null) ? null : unlinkFirst(f);
}
```

###### pollLast源码

```java
//删除尾节点
public E pollLast() {
    final Node<E> l = last;
    return (l == null) ? null : unlinkLast(l);
}
```

###### pop源码

```java
//从头部移除元素
public E pop() {
    return removeFirst();
}
```

###### removeFirstOccurrence源码

```java
public boolean removeFirstOccurrence(Object o) {
    return remove(o);
}
```
由方法名可以知道，是删除链表中首次出现的元素。和直接使用`remove(Object o)`方法效果一样。

###### removeLastOccurrence源码

```java
public boolean removeLastOccurrence(Object o) {
    if (o == null) {
        for (Node<E> x = last; x != null; x = x.prev) {
            if (x.item == null) {
                unlink(x);
                return true;
            }
        }
    } else {
        for (Node<E> x = last; x != null; x = x.prev) {
            if (o.equals(x.item)) {
                unlink(x);
                return true;
            }
        }
    }
    return false;
}
```
和方法`removeFirstOccurrence`的逻辑相反，是删除链表中最后出现的元素o。核心是遍历链表，然后删除。

##### 获取元素

###### getFirst源码

```java
//获取头节点的item值
public E getFirst() {
    final Node<E> f = first;
    if (f == null)
        throw new NoSuchElementException();
    return f.item;
}
```
功能：直接拿取头结点的item值，然后返回。

###### getLast

```java
//获取尾节点的item值
public E getLast() {
    final Node<E> l = last;
    if (l == null)
        throw new NoSuchElementException();
    return l.item;
}
```
功能：直接拿取尾结点的item值，然后返回。

###### get源码

```java
//内部通过遍历链表获取
public E get(int index) {
    checkElementIndex(index);
    return node(index).item;
}
```
功能：获取索引index处的节点的item值并返回。

###### indexOf源码

```java
//获取元素o在链表中的索引
public int indexOf(Object o) {
    int index = 0;
    //因为链表运行允许存储null元素，所以在查找元素o时，会分两种情况。
    // 但整理逻辑为：遍历链表中的节点，如果节点内的item值等于元素o，则返回对应的索引；否则，返回-1
    if (o == null) {
        for (Node<E> x = first; x != null; x = x.next) {
            if (x.item == null)
                return index;
            index++;
        }
    } else {
        for (Node<E> x = first; x != null; x = x.next) {
            if (o.equals(x.item))
                return index;
            index++;
        }
    }
    return -1;
}
```
功能：获取元素o在链表中的索引。


###### peek源码

```java
//获取链表中的头元素，但不从链表中去除
public E peek() {
    final Node<E> f = first;
    return (f == null) ? null : f.item;
}
```

###### element源码

```java
//检索但不删除此列表的头（第一个元素）。
public E element() {
    return getFirst();
}
```

###### peekFirst源码

```java
//获取头元素
public E peekFirst() {
    final Node<E> f = first;
    return (f == null) ? null : f.item;
}
```

##### 更新元素

###### set源码

```java
//先通过node()方法获取index位置上的节点，然后将其item值设置为新值
public E set(int index, E element) {
    checkElementIndex(index);
    Node<E> x = node(index);
    E oldVal = x.item;
    x.item = element;
    return oldVal;
}
```
功能：通过node()方法获取index位置上的节点，然后将其item值设置为新值。


#### LinkedList 与 ArrayList 对比

类  |底层数据结构|添加效率|删除效率|修改效率|查找效率|
---:|---|---|---|---|---|
|LinkedList |链表，由节点构成|由于底层是操作链表，所以添加效率高|高|低|低
|ArrayList |数组|由于底层是操作数组，在添加的过程中，会涉及到数组拷贝等操作，所以添加效率比LinkedList低|低|高|高





