---
title: Integer类学习
date: 2018-03-20 17:39:14
tags: java
---

### 自动装箱机制

自动装箱机制：在将**`int`**类型的数据直接赋值给**`Integer`**类型时，会触发自动装箱机制，自动装箱机制的实质是：调用了**`Integer.valueOf(i) `**方法（通过跟踪debug可知），阅读源码
```java
public static Integer valueOf(int i) {
   if (i >= IntegerCache.low && i <= IntegerCache.high)
      return IntegerCache.cache[i + (-IntegerCache.low)];
   return new Integer(i);
}
```

可知，使用了Integer中的静态内部类**`IntegerCache`**类，其实现为：

```java

/**
 * 使用了享元模式，实现了对象的复用
 */
private static class IntegerCache {
   static final int low = -128;
   static final int high;
   static final Integer cache[];

   static {
      // high value may be configured by property
      int h = 127;
      String integerCacheHighPropValue =sun.misc.VM.getSavedProperty("java.lang.Integer.IntegerCache.high");
     if (integerCacheHighPropValue != null) {
         try {
             int i = parseInt(integerCacheHighPropValue);
             i = Math.max(i, 127);
             // Maximum array size is Integer.MAX_VALUE
             h = Math.min(i, Integer.MAX_VALUE - (-low) -1);
         } catch( NumberFormatException nfe) {
             // If the property cannot be parsed into an int, ignore it.
         }
     }
     high = h;
	 cache = new Integer[(high - low) + 1];
     int j = low;
     for(int k = 0; k < cache.length; k++)
        cache[k] = new Integer(j++);
     // range [-128, 127] must be interned (JLS7 5.1.7)
     assert IntegerCache.high >= 127;
   }
   private IntegerCache() {}
}
```

可知，该静态内部类缓存了从-128到127这256个最常用数的Integer对象，Integer类被加载的时候这些缓存对象就初始化好了。因此可知在-128到127这256个最常用数在发生自动装箱的时候，它们的地址是相同的。

因此：

```java
Integer a1 = new Integer(1);
Integer a2 = new Integer(1);
System.out.println(a1 == a2);	//false

Integer b1 = 1;
Integer b2 = 1;
System.out.println(b1 == b2);	//true

Integer c1 = 200;
Integer c2 = 200;
System.out.println(c1 == c2);	//false
```
