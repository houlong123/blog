---
title: String讲解
date: 2017-07-11 13:38:02
tags: java
---


#### API文档描述
<font color=gray><i>Strings are constant; their values cannot be changed after they are created. String buffers support mutable strings. Because String objects are immutable they can be shared.
You’ll see that every method in the class that appears to modify a String actually creates and returns a brand new String object containing the modification. The original String is left untouched.</i></font>

#### “+” 重载
在String中，`+`被赋予了新的功能。该功能可以实现String字符串的连接。在对String的+操作进行反编译时，可知其内部是如何工作的。

```java
//源码
public class Concatenation {
   public static void main(String[] args) {
     String mango = "mango";
     String s = "abc" + mango + "def" + 47;
     System.out.println(s);
  }
}
```

使用`javac Concatenation.java`命令获取class文件，然后使用`javap -c Concatenation`命令得到：

<!-- more -->

```java
public class Concatenation {
  public Concatenation();
    Code:
       0: aload_0
       1: invokespecial #1                  // Method java/lang/Object."<init>":()V
       4: return
  public static void main(java.lang.String[]);
    Code:
       0: ldc           #2                  // String mango
       2: astore_1
       3: new           #3                  // class java/lang/StringBuilder
       6: dup
       7: invokespecial #4                  // Method java/lang/StringBuilder."<init>":()V
      10: ldc           #5                  // String abc
      12: invokevirtual #6                  // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
      15: aload_1
      16: invokevirtual #6                  // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
      19: ldc           #7                  // String def
      21: invokevirtual #6                  // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
      24: bipush        47
      26: invokevirtual #8                  // Method java/lang/StringBuilder.append:(I)Ljava/lang/StringBuilder;
      29: invokevirtual #9                  // Method java/lang/StringBuilder.toString:()Ljava/lang/String;
      32: astore_2
      33: getstatic     #10                 // Field java/lang/System.out:Ljava/io/PrintStream;
      36: aload_2
      37: invokevirtual #11                 // Method java/io/PrintStream.println:(Ljava/lang/String;)V
      40: return
}
```
由上可知，编译器创建了`StringBuilder`对象去构建源码中的字符串s。其实现是调用`append()`方法实现字符串的连接，最后调用`toString()`方法生成结果，把它存储作为s。

简单程序中，可以发现使用`+`或者 `StringBuilder` 进行字符串连接时，并没有什么区别，使用哪个都可以。但是在有循环操作时，`StringBuilder` 比较有优势。看代码：

```java
public class WhitherStringBuilder {
  //循环内，使用+操作进行字符串连接
  public String implicit(String[] fields) {
    String result = "";
    for(int i = 0; i < fields.length; i++)
      result += fields[i];
    return result;
  }
  //循环内，使用StringBuilder操作进行字符串连接
  public String explicit(String[] fields) {
    StringBuilder result = new StringBuilder();
    for(int i = 0; i < fields.length; i++)
      result.append(fields[i]);
    return result.toString();
  }
}
```

---

反编译


```java
public Concatenation();
    Code:
       0: aload_0
       1: invokespecial #1                  // Method java/lang/Object."<init>":()V
       4: return
   //使用+操作进行字符串连接
  public java.lang.String implicit(java.lang.String[]);
    Code:
       0: ldc           #2                  // String
       2: astore_2
       3: iconst_0
       4: istore_3
       5: iload_3
       6: aload_1
       7: arraylength
       8: if_icmpge     38
      11: new           #3                  // class java/lang/StringBuilder
      14: dup
      15: invokespecial #4                  // Method java/lang/StringBuilder."<init>":()V
      18: aload_2
      19: invokevirtual #5                  // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
      22: aload_1
      23: iload_3
      24: aaload
      25: invokevirtual #5                  // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
      28: invokevirtual #6                  // Method java/lang/StringBuilder.toString:()Ljava/lang/String;
      31: astore_2
      32: iinc          3, 1
      35: goto          5    //重点，跳转到5
      38: aload_2
      39: areturn
 
  //使用StringBuilder操作进行字符串连接
  public java.lang.String explicit(java.lang.String[]);
    Code:
       0: new           #3                  // class java/lang/StringBuilder
       3: dup
       4: invokespecial #4                  // Method java/lang/StringBuilder."<init>":()V
       7: astore_2
       8: iconst_0
       9: istore_3
      10: iload_3
      11: aload_1
      12: arraylength
      13: if_icmpge     30
      16: aload_2
      17: aload_1
      18: iload_3
      19: aaload
      20: invokevirtual #5                  // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
      23: pop
      24: iinc          3, 1
      27: goto          10          //重点，跳转到第10 步
      30: aload_2
      31: invokevirtual #6                  // Method java/lang/StringBuilder.toString:()Ljava/lang/String;
      34: areturn
}
```

在反编译代码中，在`implicit()`(即使用+操作字符串)方法中，重点注意第  `8:`和`35:`。8:表示：如果循环完毕，在执行38:，否则往下执行。35:表示：跳转到循环开始出5:。重点是`StringBuilder`的创建发生在`11:`，位于循环之内，因此可知在使用+操作字符串时，会产生许多StringBuilder临时对象，性能不太好。

在`explicit()`(即使用StringBuilder操作字符串)方法中，StringBuilder的创建发生在循环之外，性能好些。

因此得知，在涉及到循环时，建议使用StringBuilder连接字符串。


####  switch对字符串支持的实现
在java 7之前，switch语句的判断条件可以接受`int`，`byte`，`char`，`short`，不能接受其他类型。但是在Java 7 之后，switch语句可以支持String类型了。`switch对String的支持是通过使用equals()方法和hashcode()方法实现的`


Java代码
```java
public class Test {

        public static void main(String[] args) {
            String str = "world";
            switch (str) {
                case "hello":
                    System.out.println("hello");
                    break;
                case "world":
                    System.out.println("world");
                    break;
                default: break;
            }
        }
}
```

编译后代码

```java
public class Test {
    public Test() {
    }

    public static void main(String[] args) {
        String str = "world";
        byte var3 = -1;
        switch(str.hashCode()) { //在switch语句中使用了String的hashCode()方法
        case 99162322:
            if(str.equals("hello")) {   //在哈希值相同的情况下，可能会发生哈希碰撞，因此通过使用equals()方法比较进行安全检查，这个检查是必要的
                var3 = 0;
            }
            break;
        case 113318802:
            if(str.equals("world")) {
                var3 = 1;
            }
        }

        switch(var3) {
        case 0:
            System.out.println("hello");
            break;
        case 1:
            System.out.println("world");
        }

    }
}

```
从上面编译后的代码看，可以知道字符串的switch是通过equals()和hashCode()方法来实现的。switch语句中只能使用整型。使用hashCode()的返回值进行switch语句判断，在哈希值匹配后，由于哈希碰撞，在使用equals()进行精确判断，实现流程控制。
