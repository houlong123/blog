---
title: java接口
date: 2017-07-05 18:59:44
tags: java
---

<i><font color=red>接口和内部类为我们提供了一种将接口与实现分离的更加结构化的方法。</font></i>

#### 抽象类与抽象方法

+ 抽象方法

    仅有声明而没有方法体的方法叫抽象方法。

+ 抽象类

    包含抽象方法的类叫做抽象类。

```
为抽象类创建对象是不安全的。即：抽象类不能创建对象。
```

#### 接口

<i><font color=red>interface是一个极度抽象的类，interface这个关键字产生一个安全抽象的类，根本没有提供任何具体实现。</i></font>

##### 注意点

+ 在interface中，所有方法在声明时即使不加public，默认是public
+ interface中可以包含域，这些域隐式的为 <font color=red>`static final`</font>的。这些域不是接口的一部分，他们的值被存储在该接口的静态存储区域内。
+ 在Java中，不支持多重继承。但是，interface可以支持多重继承。

##### 使用接口的原因

+ 核心原因：为了能够向上转型为多个基类型（策略模式实现的基础）
+ 为了防止客户端程序员创建该类的对象

#### 抽象类与接口的区别

interface是 <font color=red>`like a`</font>关系，定义该体系的额外功能；抽象类是 <font color=red>`is a`</font>关系，定义该体系的基本共性内容。

