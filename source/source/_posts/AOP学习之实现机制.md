---
title: AOP学习之实现机制
date: 2017-07-31 13:51:07
tags: java
---

### 概述
> Spring AOP是Spring核心框架的重要组成部分。采用动态代理机制和字节码生成技术实现。

![image](https://farm5.staticflickr.com/4301/36147092481_99db67114d_z.jpg)

### 实现机制

+ ****实现逻辑****

  动态代理机制和字节码生成都是在运行期间为目标对象生成一个 <i><b><font color=red size=2 face="黑体">代理对象</font></b></i>，而将横切逻辑织入到这个代理对象中，系统最终使用的是织入了横切逻辑的代理对象，而不是真正的目标对象。
  
  <!-- more -->

+ ****实现方式一：静态代理****

    1. ****类图****
    
    ![image](http://thumbsnap.com/s/ehEt4yED.jpg)
    **ISubject**：被访问者或者被访问资源的抽象。
    
    **SubjectImpl**：被访问者或者被访问资源的具体实现。
    
    **SubjectProxy**：被访问者或者被访问资源的代理实现类，该类持有一个`ISubject`接口的具体实例。
    
    由类图可知，实现静态代理的前提是目标对象和代理对象都实现了相同的接口，且代理对象内部持有一个目标对象引用。
    
    2. ****优缺点****
    
    2.1 优点
    + 可以做到在不修改目标对象的功能前提下，对目标功能扩展
    
    2.2 缺点
    + 因为代理对象需要与目标对象实现一样的接口，所以会有很多代理类，类太多。同时，一旦接口增加方法，目标对象与代理对象都要维护。
    + 当代理对象所要添加的横切逻辑一样，但是目标对象的类型不同时，需要为不同类型的目标对象创建代理对象。
    
    3. ****具体代码****

  
```java
 //公共接口
 interface ISubject {
    
    public String request();
 }
```

```java
//目标对象
public class SubjectImpl implements ISubject {

    @Override
    public String request() {
        //process logic
        return  "OK";
    }
}
```

    
```java
//代理对象
public class SubjectProxy implements ISubject {

    private ISubject target;

    public SubjectProxy(ISubject target) {
        this.target = target;
    }
    
    @Override
    public String request() {
        // add pre-process logic if necessary

        String originalResult = target.request();

        //add post process logic if necessary
        return  "Proxy：" + originalResult;
    }
}

```

```java
public class ProxyTest {

    public static void main(String[]args){
        ISubject target = new ISubjectImpl();
        ISubject finalSubject = new ISubjectProxy(target);
        finalSubject.request();
    }
}
```


+ ****实现方式二：动态代理****
    1. **Java动态代理构成**
    
        使用该机制，我们可以为指定的接口在系统运行期间动态的生成代理对象。该机制的实现主要由一个类和一个接口组成。即`java.lang.reflect.Proxy`类 和 `java.lang.reflect.InvocationHandler`接口。当Proxy动态生成的代理对象上的相应接口方法被调用时，InvocationHandler就会拦截相应的方法调用，并进行相应处理。

    2. **特点**
    + 代理对象，不需要实现接口
    + 代理对象不需要实现接口，但是目标对象一定要实现接口，否则不能用动态代理。
    + 代理对象的生成，是利用JDK的API，动态的在内存中构建代理对象。
    + 动态代理也叫做：JDK代理，接口代理。
    
    3. **具体代码**
    
    
```java
//横切逻辑。InvocationHandler就是我们实现横切逻辑的地方
public class RequestCtrlInvocationHandler implements InvocationHandler {

    private Object target;

    public RequestCtrlInvocationHandler(Object target) {
        this.target = target;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        if (method.getName().equals("request")) {
            // add pre-process logic if necessary
            
            String originalResult = method.invoke(target, args);

            //add post process logic if necessary
            return  "Proxy：" + originalResult;
        }
        return null;
    }
}
```

```java
//不同目标对象织入相同的横切逻辑
public class ProxyTest {

    public static void main(String[] args) {
        ISubject subject = (ISubject)Proxy.newProxyInstance(ProxyTest.class.getClassLoader(), new Class[]{ISubject.class}, new RequestCtrlInvocationHandler(new SubjectImpl()));
        subject.request();

        
        IRequestable requestable = (IRequestable)Proxy.newProxyInstance(ProxyTest.class.getClassLoader(), new Class[]{IRequestable.class}, new RequestCtrlInvocationHandler(new RequestableImpl()));
        requestable.request();
    }
}
```

+ ****实现方式三：动态字节码生成****
  1. **原理**
    
        对目标对象进行扩展，为其生成相应子类，子类通过覆写来扩展父类的行为。将横切逻辑的实现放在子类中，然后使用扩展后的目标对象的子类，即可实现相应代理。
    2. **不足**
    + 使用CGLIB对类进行扩展的唯一限制就是无法对final方法进行覆写。
    
    3. **具体代码**

```java
//目标对象
public class Requestable {
    public void request() {
        System.out.println("request in Requestable without implement any interfave");
    }
}
```


```java
//横切逻辑。需要引入net.sf.cglib包。
public class RequestCtrlCallBack implements MethodInterceptor {
    public Object intercept(Object object, Method method, Object[] args, MethodProxy proxy) throws Throwable {
        if (method.getName().equals("request")) {
            // add pre-process logic if necessary

            String originalResult = proxy.invokeSuper(object, args);

            //add post process logic if necessary
            return  "Proxy：" + originalResult;
        }
        return null;
    }
}
```


```java
public class ProxyTest {
    public static void main(String[] args) {
        Enhancer enhancer = new Enhancer();
        enhancer.setSuperclass(Requestable.class);
        enhancer.setCallBack(new RequestCtrlCallBack());

        //通过Enhancer为目标对象动态生成一个子类，并将横切逻辑加入子类。
        Requestable proxy = (Requestable)enhancer.create();
        proxy.request();
    }
}
```

默认情况下，Spring AOP发现了目标对象实现了相应的Interface，则采用动态代理机制来生成代理对象；如果目标对象没有实现任何Interface，Spring AOP会尝试使用名为CGLIB(code generation Library)的开源的动态字节码生成类库，为目标对象生成代理对象。
  
  ### 参考文献
  
  [代理模式](https://www.zybuluo.com/pastqing/note/174679)