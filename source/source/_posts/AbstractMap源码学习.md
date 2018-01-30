---
title: AbstractMap源码学习
date: 2018-01-30 18:08:24
tags: java基础集合
---

#### 前言
关于 java 的集合框架，我们前面已经学习了 `Collection接口` 及其子类，比如：[ArrayList](https://houlong123.github.io/2018/01/23/ArrayList源码学习/)， [LinkedList](https://houlong123.github.io/2018/01/26/LinkedList源码学习/), [HashSet](https://houlong123.github.io/2018/01/29/Set相关集合源码学习/) 等。在本篇中，我们将学习日常编程中最常用的另一集合分支：Map。我们最常用的Map集合为HashMap，从类图中，我们可知，它的父类为`AbstractMap`，所以，我们从`AbstractMap`入手分析Map的相关源码。

#### 源码分析

<!-- more -->

##### 数据结构

```java
/**
 *  API文档描述
 * To implement an unmodifiable map, the programmer needs only 
 * to extend this class and provide an implementation for the entrySet method, 
 * which returns a set-view of the map's mappings. Typically, 
 * the returned set will, in turn, be implemented atop AbstractSet. 
 * This set should not support the add or remove methods, and its iterator 
 * should not support the remove method. 
 *（为了实现一个不可修改的Map，需扩展该类并实现entrySet方法，返回的集合不支持add或remove方法，它的迭代器不应该支持remove方法。）
 *
 * To implement a modifiable map, the programmer must additionally override
 * this class's put method (which otherwise throws an
 * UnsupportedOperationException), and the iterator returned by entrySet().iterator() 
 * must additionally implement its remove method.  
 *（为了实现一个可修改的Map，需重写该类put方法，且entrySet().iterator()方法的迭代器必须另外实现它的remove方法。）
 */
public abstract class AbstractMap<K,V> implements Map<K,V> {
    //keySet，values都是volatile的，保证了内存可见性。（1. 当写入主存时，使其值无效；2. 防止内存重排序）
    transient volatile Set<K>        keySet;
    transient volatile Collection<V> values;
    
    //An Entry maintaining a key and a value. 维持一个value可变的key-value键值对
    public static class SimpleEntry<K,V>
        implements Entry<K,V>, java.io.Serializable {
        
        private final K key;
        private V value;
    }
        
    //An Entry maintaining an immutable key and value  维持一个不可变的key-value键值对
    public static class SimpleImmutableEntry<K,V>
        implements Entry<K,V>, java.io.Serializable {
         
        private final K key;
        private final V value;   
    }
}
```
由 AbstractMap 的源码可知，该类继承了 `Map接口`。内部只有两个属性：keySet，values。这两个属性是Map集合中用于存储元素的Entry实体的视图展示形式。

###### Map源码

```java
/**
 * An object that maps keys to values.  A map cannot contain duplicate keys;
 * each key can map to at most one value.  (Map集合中不能保持重复的key)
 */
 public interface Map<K,V> {
    int size();
    boolean isEmpty();
    boolean containsKey(Object key);
    boolean containsValue(Object value);
    V get(Object key);
    V put(K key, V value);
    V remove(Object key);
    void putAll(Map<? extends K, ? extends V> m);
    void clear();
    Set<K> keySet();
    Collection<V> values();
    Set<Entry<K, V>> entrySet();
    boolean equals(Object o);
    int hashCode();
    
    //1.8版本新增方法
    default V getOrDefault(Object key, V defaultValue) {
        V v;
        //如果key存在，在返回key对应的value值；否则返回默认值
        return (((v = get(key)) != null) || containsKey(key))
            ? v
            : defaultValue;
    }
    
    default V putIfAbsent(K key, V value) {
        V v = get(key);
        if (v == null) {
            v = put(key, value);
        }

        return v;
    }
    
    default boolean remove(Object key, Object value) {
        Object curValue = get(key);
        if (!Objects.equals(curValue, value) ||
            (curValue == null && !containsKey(key))) {
            return false;
        }
        remove(key);
        return true;
    }
    ...
    
    
    //内部Entry类
    interface Entry<K,V> {
    
        //基本方法
        K getKey();
        V getValue();
        V setValue(V value);
        boolean equals(Object o);
        int hashCode();
        
        //获取比较器。1.8版本
        public static <K extends Comparable<? super K>, V> Comparator<Entry<K,V>> comparingByKey() {
            return (Comparator<Entry<K, V>> & Serializable)
                (c1, c2) -> c1.getKey().compareTo(c2.getKey());
        }
        
        public static <K, V extends Comparable<? super V>> Comparator<Entry<K,V>> comparingByValue() {
            return (Comparator<Entry<K, V>> & Serializable)
                (c1, c2) -> c1.getValue().compareTo(c2.getValue());
        }
        
        public static <K, V> Comparator<Entry<K, V>> comparingByKey(Comparator<? super K> cmp) {
            Objects.requireNonNull(cmp);
            return (Comparator<Entry<K, V>> & Serializable)
                (c1, c2) -> cmp.compare(c1.getKey(), c2.getKey());
        }
        
        public static <K, V> Comparator<Entry<K, V>> comparingByValue(Comparator<? super V> cmp) {
            Objects.requireNonNull(cmp);
            return (Comparator<Entry<K, V>> & Serializable)
                (c1, c2) -> cmp.compare(c1.getValue(), c2.getValue());
        }
    }
 }
```
关于 Map 接口，它里面定义了集合 Map所有涉及到的操作。并且内部有个 `Entry` 类的实体，该实体用于存储key-value键值对。且其内部有抽象方法，也有具体实现的方法。需要注意的是，具体实现的方法也是只定义了一个模板，具体逻辑还是由相应子类实现。

##### size源码

```java
//内部调用entrySet()方法，返回entrySet()结果的大小
public int size() {
    return entrySet().size();
}
```
内部调用 `entrySet()` 方法。`entrySet()` 方法是返回内部Entry实体的Set集合。

##### entrySet源码

```java
//抽象方法，由子类实现。从上面的代码可以发现，其他方法的实现都是基于该方法
public abstract Set<Entry<K,V>> entrySet();
```
该方法为抽象方法。具体需要子类去实现。从后面的代码中，我们可以发现，其他方法的 实现都是基于该方法的。


##### isEmpty源码

```java
public boolean isEmpty() {
    return size() == 0;
}
```

##### containsValue源码

```java
public boolean containsValue(Object value) {
    //通过entrySet()方法，获取内部元素的迭代器
    Iterator<Entry<K,V>> i = entrySet().iterator();

    //由于Map集合的value允许储存null值，因此，在查找指定value是否存在时，需要分两种情况
    if (value==null) {
        while (i.hasNext()) {
            Entry<K,V> e = i.next();
            if (e.getValue()==null)
                return true;
        }
    } else {
        while (i.hasNext()) {
            Entry<K,V> e = i.next();
            if (value.equals(e.getValue()))
                return true;
        }
    }
    return false;
}
```
该方法功能：判断Map集合中是否存在元素value。内部大体逻辑是：首先通过 `entrySet()`方法获取Map中Entry对象集合的迭代器，然后通过迭代器遍历Entry对象集合，如果其value值与参数value值相等，则返回true；否则，返回false。


##### containsKey源码

```java
public boolean containsKey(Object key) {
    Iterator<Entry<K,V>> i = entrySet().iterator();

    //由于Map集合的key允许储存null值，因此，在查找指定key是否存在时，需要分两种情况
    if (key==null) {
        while (i.hasNext()) {
            Entry<K,V> e = i.next();
            if (e.getKey()==null)
                return true;
        }
    } else {
        while (i.hasNext()) {
            Entry<K,V> e = i.next();
            if (key.equals(e.getKey()))
                return true;
        }
    }
    return false;
}
```
该方法的功能：判断Map集合中是否存在相应的key。大体实现逻辑与 `containsValue` 类似。


##### get源码

```java
public V get(Object key) {
    //通过entrySet()方法，获取内部元素的迭代器
    Iterator<Entry<K,V>> i = entrySet().iterator();

    //由于Map集合的key-value允许存储null值，所以分两种情况查找

    //遍历，然后找到与key所在的Entry对应的value
    if (key==null) {
        while (i.hasNext()) {
            Entry<K,V> e = i.next();
            if (e.getKey()==null)
                return e.getValue();
        }
    } else {
        while (i.hasNext()) {
            Entry<K,V> e = i.next();
            if (key.equals(e.getKey()))
                return e.getValue();
        }
    }
    return null;
}
```
该方法与 `containsKey` 方法的实现逻辑基本类似，只不过该方法是返回key在Map集合中所对应的value值。

##### remove源码

```java
//遍历，通过内部迭代器实现删除操作
public V remove(Object key) {
    //通过entrySet()方法，获取内部元素的迭代器
    Iterator<Entry<K,V>> i = entrySet().iterator();

    //遍历数据，找到key所对应的Entry实体
    Entry<K,V> correctEntry = null;
    if (key==null) {
        while (correctEntry==null && i.hasNext()) {
            Entry<K,V> e = i.next();
            if (e.getKey()==null)
                correctEntry = e;
        }
    } else {
        while (correctEntry==null && i.hasNext()) {
            Entry<K,V> e = i.next();
            if (key.equals(e.getKey()))
                correctEntry = e;
        }
    }

    //找到key所对应的Entry实体后，使用迭代器进行删除
    V oldValue = null;
    if (correctEntry !=null) {
        oldValue = correctEntry.getValue();
        i.remove();
    }

    //返回key对应的value值，如果key所对应的Entry实体不存在，则返回null
    return oldValue;
}
```
功能：在Map集合中删除key所对应的key-value键值对，并返回对应的value值，如果不存在，返回null。代码实现逻辑可以参考`containsValue`的源码解析。

##### clear源码

```java
public void clear() {
    entrySet().clear();
}
```

##### putAll源码

```java
public void putAll(Map<? extends K, ? extends V> m) {

    //遍历m，然后将值存储在调用者的Map中
    for (Entry<? extends K, ? extends V> e : m.entrySet())
        put(e.getKey(), e.getValue());
}
```
通过 `entrySet()` 方法获取Map中Entry对象集合的迭代器，然后依次将集合m中的key-value键值对通过调用 `put` 方法 存放到Map中。

##### put源码

```java
//抛出UnsupportedOperationException异常，对于可修改的Map，需要实现该方法
public V put(K key, V value) {
    throw new UnsupportedOperationException();
}
```
该方法为空实现。功能是往Map集合中添加key-value键值对。

##### equals源码

```java
public boolean equals(Object o) {
    //如果o == this，则直接返回true
    if (o == this)
        return true;

    //如果对象o非Map类型，则直接返回false
    if (!(o instanceof Map))
        return false;

    //如果对象o的大小与调用者的不等，则直接返回false
    Map<?,?> m = (Map<?,?>) o;
    if (m.size() != size())
        return false;


    try {
        //获取entrySet()的迭代器
        Iterator<Entry<K,V>> i = entrySet().iterator();

        //遍历调用者Map，依次判断是否相等
        while (i.hasNext()) {
            Entry<K,V> e = i.next();
            K key = e.getKey();
            V value = e.getValue();
            if (value == null) {
                if (!(m.get(key)==null && m.containsKey(key)))
                    return false;
            } else {
                if (!value.equals(m.get(key)))
                    return false;
            }
        }
    } catch (ClassCastException unused) {
        return false;
    } catch (NullPointerException unused) {
        return false;
    }

    return true;
}
```
方法功能：判断两个对象是否相等。实现逻辑也是首先通过 `entrySet()` 方法获取迭代器，然后将两个Map集合中的元素进行对比。

<font color=red>备注：在判断两个Map集合是否相等时，与内部元素的顺序无关</font>

###### 例子

```java
public class MapTest {
    public static void main(String[] args) {
        Map<String, String> map1 = new LinkedHashMap<>();
        map1.put("hah", "hah");
        map1.put("xix", "xix");

        System.out.println(JSONObject.toJSONString(map1));

        Map<String, String> map2 = new LinkedHashMap<>();
        map2.put("xix", "xix");
        map2.put("hah", "hah");

        System.out.println(JSONObject.toJSONString(map2));

        System.out.println("map1.equals(map2) = " + map1.equals(map2));

    }
}

// output
{"hah":"hah","xix":"xix"}
{"xix":"xix","hah":"hah"}
map1.equals(map2) = true
```

##### hashCode源码

```java
public int hashCode() {
    int h = 0;
    Iterator<Entry<K,V>> i = entrySet().iterator();
    while (i.hasNext())
        h += i.next().hashCode();
    return h;
}
```

从上面的源码可以看出，AbstractMap 中的非抽象方法都是 `模板方法` 设计模式的典型使用，内部都会调用抽象方法 `entrySet()` ，该方法需要子类进行实现。

##### clone源码

```java
//AbstractMap实例的浅拷贝。且内部的keySet和values不进行拷贝
protected Object clone() throws CloneNotSupportedException {
    AbstractMap<?,?> result = (AbstractMap<?,?>)super.clone();
    result.keySet = null;
    result.values = null;
    return result;
}
```

##### keySet源码

```java
/**
 * This implementation returns a set that subclasses AbstractSet. 
 * The subclass's iterator method returns a "wrapper object" over this map's entrySet() iterator. The size 
 * method delegates to this map's size method and the contains method delegates to this map's containsKey method.
 */
public Set<K> keySet() {
    //在该方法首次调用的时候，会创建keySet
    if (keySet == null) {
        keySet = new AbstractSet<K>() {

            //返回了该map的entrySet()对象的迭代器的包装类
            public Iterator<K> iterator() {
                return new Iterator<K>() {

                    //获取entrySet()的迭代器
                    private Iterator<Entry<K,V>> i = entrySet().iterator();

                    //里面的相关方法都有entrySet()的迭代器代理
                    public boolean hasNext() {
                        return i.hasNext();
                    }

                    public K next() {
                        return i.next().getKey();
                    }

                    public void remove() {
                        i.remove();
                    }
                };
            }

            //方法实现都是AbstractMap的相关方法
            public int size() {
                return AbstractMap.this.size();
            }

            public boolean isEmpty() {
                return AbstractMap.this.isEmpty();
            }

            public void clear() {
                AbstractMap.this.clear();
            }

            public boolean contains(Object k) {
                return AbstractMap.this.containsKey(k);
            }
        };
    }
    return keySet;
}
```
该方法返回了一个`AbstractSet`类的子类。子类的迭代器方法返回一个通过此Map的 `entrySet()`对象获取的迭代器的 “包装器对象”（翻译的很拗口，看方法英文注释吧，理解就好😂😂）。 且其内部的所有方法都委托为了Map对象。


##### values源码

```java
public Collection<V> values() {
    //实现逻辑同keySet()方法一致
    if (values == null) {
        values = new AbstractCollection<V>() {
            public Iterator<V> iterator() {
                return new Iterator<V>() {
                    private Iterator<Entry<K,V>> i = entrySet().iterator();

                    public boolean hasNext() {
                        return i.hasNext();
                    }

                    public V next() {
                        return i.next().getValue();
                    }

                    public void remove() {
                        i.remove();
                    }
                };
            }

            public int size() {
                return AbstractMap.this.size();
            }

            public boolean isEmpty() {
                return AbstractMap.this.isEmpty();
            }

            public void clear() {
                AbstractMap.this.clear();
            }

            public boolean contains(Object v) {
                return AbstractMap.this.containsValue(v);
            }
        };
    }
    return values;
}
```
不说了，与 `keySet()` 方法实现逻辑基本一致。

