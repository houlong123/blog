---
title: Spring AOP源码学习
date: 2017-08-22 14:13:31
tags: Spring
---

#### AOP基础知识
AOP(Aspect-Oriented Programming)：面向切面编程。OOP(Object-Oriented Programming)面向对象的编程。其实AOP是对OOP的一种补充。OOP面向的是纵向编程，`继承，封装，多态`是其三大特性，而AOP是面向横向编程。在OOP中，模块化的关键是`类(class)`，而在AOP中，模块化的关键是`切面(aspect)`。

##### AOP概念术语
+ **切面(Aspect)**：一个关注点的模块化，这个关注点可能会横切多个对象。
+ **连接点(Joinpoint)**：在程序执行过程中某个特定的点。在Spring AOP中，`一个连接点总是表示一个方法的执行`。
+ **通知(Advice)**：在切面的某个特定的连接点上执行的动作。包括：`around`, `before`, `after`等不同类型的通知。
+ **切入点(Pointcut)**：匹配连接点的断言。通知和一个切入点表达式关联，并在满足这个切入点的连接点上运行。
+ **引入(Introduction)**：用来给一个类型说明额外的方法或者属性。
+ **目标对象(Target Object)**：被一个或者多个切面所通知的对象。既然Spring AOP是通过运行时代理实现的，这个对象永远是一个被代理（proxied）对象。
+ **AOP代理(AOP Proxy)**：AOP框架创建的对象，用来实现切面契约（例如通知方法执行等等）。在Spring中，AOP代理可以是[JDK动态代理或者CGLIB代理](https://houlong123.github.io/2017/07/31/AOP%E5%AD%A6%E4%B9%A0%E4%B9%8B%E5%AE%9E%E7%8E%B0%E6%9C%BA%E5%88%B6/)。
+ **织入(Weaving)**：把切面连接到其他的应用程序类型或者对象上，并创建一个被通知的对象。

<!-- more -->

##### AOP通知类型
+ **前置通知(Before advice)**：在某连接点之前执行的通知，但这个通知不能阻止连接点之前的执行流程（除非它抛出一个异常）。
+ **后置通知(After returning advice)**：在某连接点正常完成后执行 的通知。
+ **异常通知(After throwing advide)**：在方法抛出异常退出时执行的通知。
+ **最终通知(After finally advice)**：当某连接点退出的时候执行的通知(不论是正常返回还是异常退出)。
+ **环绕通知(Around Advice)**：包围一个连接点的通知，环绕通知可以在方法调用前后完成自定义的行为。它也会选择是否继续执行连接点或直接返回它自己的返回值或抛出异常来结束执行。


#### AOP源码解析之xml文件读取

实例：

```java
//接口
public interface Dao {
    public void select();
    public void insert();
}
```


```java
//实现类
public class DaoImpl extends Dao {
    @Override
    public void select() {
        System.out.println("Enter DaoImpl.select()");
    }
    
    @Override
    public void insert() {
        System.out.println("Enter DaoImpl.insert()");
    }
}
```


```java
//在AOP中，扮演横切关注点角色
public class TimeHandler {
    public void printTime() {
        System.out.println("CurrentTime: " + System.currentTimeMillis());
    }
}
```




```xml
<beans xmlns="http://www.springframework.org/schema/beans"
      xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
      xmlns:aop="http://www.springframework.org/schema/aop"
      xmlns:tx="http://www.springframework.org/schema/tx"
      xsi:schemaLocation="http://www.springframework.org/schema/beans
          http://www.springframework.org/schema/beans/spring-beans-3.0.xsd
          http://www.springframework.org/schema/aop
          http://www.springframework.org/schema/aop/spring-aop-3.0.xsd">
 
     <bean id="daoImpl" class="org.xrq.action.aop.DaoImpl" />
     <bean id="timeHandler" class="org.xrq.action.aop.TimeHandler" />
 
     <aop:config proxy-target-class="true">
         <aop:aspect id="time" ref="timeHandler">
             <aop:pointcut id="addAllMethod" expression="execution(* org.xrq.action.aop.Dao.*(..))" />
             <aop:before method="printTime" pointcut-ref="addAllMethod" />
             <aop:after method="printTime" pointcut-ref="addAllMethod" />
         </aop:aspect>
     </aop:config>
     
 </beans>
```


```Java
public class TestAop {
    @Test
    public void testAop() {
        ApplicationContext ac = new ClassPathXmlApplicationContext("spring/aop.xml");
        Dao dao = (Dao) ac.getBean("daoImpl");
        dao.select();
    }
}

//output
CurrentTime: 1231223432
Enter DaoImpl.select()
CurrentTime: 1231223532
```

在文章[Bean加载流程梳理之BeanDefinitions加载](https://houlong123.github.io/2017/08/10/Bean%E5%8A%A0%E8%BD%BD%E6%B5%81%E7%A8%8B%E6%A2%B3%E7%90%86%E4%B9%8BBeanDefinitions%E5%8A%A0%E8%BD%BD/)中，已知xml文件的解析是发生在DefaultBeanDefinitionDocumentReader的parseBeanDefinitions方法中。

##### parseBeanDefinitions源码

```java
protected void parseBeanDefinitions(Element root, BeanDefinitionParserDelegate delegate) {
        if(delegate.isDefaultNamespace(root)) {
            NodeList nl = root.getChildNodes();

            for(int i = 0; i < nl.getLength(); ++i) {
                Node node = nl.item(i);
                if(node instanceof Element) {
                    Element ele = (Element)node;
                    if(delegate.isDefaultNamespace(ele)) {
                        this.parseDefaultElement(ele, delegate);
                    } else {
                        delegate.parseCustomElement(ele);
                    }
                }
            }
        } else {
            delegate.parseCustomElement(root);
        }

    }
```
由源码可知，xml配置文件中的`<import>`, `<alias>`, `<bean>`, `<beans>`元素交由`parseDefaultElement`方法处理，其他标签则交由`parseCustomElement`方法处理。

##### parseCustomElement源码

```java
public BeanDefinition parseCustomElement(Element ele, BeanDefinition containingBd) {
        //<aop:config>这个Node（参数Element是Node接口的子接口）中拿到Namespace="http://www.springframework.org/schema/aop"
        String namespaceUri = this.getNamespaceURI(ele);
        
        //根据这个Namespace获取对应的NamespaceHandler即Namespace处理器，具体到aop这个Namespace的NamespaceHandler是org.springframework.beans.factory.xml.NamespaceHandlerSupport类。
        NamespaceHandler handler = this.readerContext.getNamespaceHandlerResolver().resolve(namespaceUri);
        if(handler == null) {
            this.error("Unable to locate Spring NamespaceHandler for XML schema namespace [" + namespaceUri + "]", ele);
            return null;
        } else {
            return handler.parse(ele, new ParserContext(this.readerContext, this, containingBd));
        }
    }
```
在Spring AOP中，有几个Parser用于具体标签转换的：

标签 | parser处理类
---|---
aop:config | ConfigBeanDefinitionParser
aop:aspectj-autoproxy | AspectJAutoProxyBeanDefinitionParser
aop:scoped-proxy | ScopedProxyBeanDefinitionDecorator

##### NamespaceHandlerSupport类中parse源码

```java
public BeanDefinition parse(Element element, ParserContext parserContext) {
        //由上面的表格可知，具体的parser为ConfigBeanDefinitionParser
        return this.findParserForElement(element, parserContext).parse(element, parserContext);
    }
```

##### ConfigBeanDefinitionParser类中parse源码

```java
public BeanDefinition parse(Element element, ParserContext parserContext) {
        CompositeComponentDefinition compositeDef = new CompositeComponentDefinition(element.getTagName(), parserContext.extractSource(element));
        parserContext.pushContainingComponent(compositeDef);
        
        /**
         * 完成两个功能：
         * 1. 向Spring容器注册了一个BeanName为org.springframework.aop.config.internalAutoProxyCreator的Bean定义，可以自定义也可以使用Spring提供的（根据优先级来）
         * 
         * 2. Spring默认提供的是org.springframework.aop.aspectj.autoproxy.AspectJAwareAdvisorAutoProxyCreator，这个类是AOP的核心类
         * 
         * 3.根据配置proxy-target-class和expose-proxy，设置是否使用CGLIB进行代理以及是否暴露最终的代理。
         */
        this.configureAutoProxyCreator(parserContext, element);
        
        List childElts = DomUtils.getChildElements(element);
        Iterator var5 = childElts.iterator();

        while(var5.hasNext()) {
            Element elt = (Element)var5.next();
            String localName = parserContext.getDelegate().getLocalName(elt);
            if("pointcut".equals(localName)) {
                this.parsePointcut(elt, parserContext);
            } else if("advisor".equals(localName)) {
                this.parseAdvisor(elt, parserContext);
            } else if("aspect".equals(localName)) {  //<aop:config>下的节点为<aop:aspect>,因此会执行下面代码
                this.parseAspect(elt, parserContext);
            }
        }

        parserContext.popAndRegisterContainingComponent();
        return null;
    }
```

##### parseAspect源码
```java
private void parseAspect(Element aspectElement, ParserContext parserContext) {
        //根据配置<aop:aspect id="time" ref="timeHandler">，获取切面ID及切面名
        String aspectId = aspectElement.getAttribute("id");
        String aspectName = aspectElement.getAttribute("ref");

        try {
            this.parseState.push(new AspectEntry(aspectId, aspectName));
            ArrayList beanDefinitions = new ArrayList();
            ArrayList beanReferences = new ArrayList();
            List declareParents = DomUtils.getChildElementsByTagName(aspectElement, "declare-parents");

            for(int nodeList = 0; nodeList < declareParents.size(); ++nodeList) {
                Element adviceFoundAlready = (Element)declareParents.get(nodeList);
                beanDefinitions.add(this.parseDeclareParents(adviceFoundAlready, parserContext));
            }

            // 获取<aop:aspect>的子元素，遍历
            NodeList var17 = aspectElement.getChildNodes();
            boolean var18 = false;

            for(int aspectComponentDefinition = 0; aspectComponentDefinition < var17.getLength(); ++aspectComponentDefinition) {
                Node pointcuts = var17.item(aspectComponentDefinition);
               
                //由isAdviceNode方法可知，只用来处理<aop:aspect>标签下的<aop:before>、<aop:after>、<aop:after-returning>、<aop:after-throwing method="">、<aop:around method="">这五个标签的。
                if(this.isAdviceNode(pointcuts, parserContext)) {
                    if(!var18) {
                        var18 = true;
                        if(!StringUtils.hasText(aspectName)) {
                            parserContext.getReaderContext().error("<aspect> tag needs aspect bean reference via \'ref\' attribute when declaring advices.", aspectElement, this.parseState.snapshot());
                            return;
                        }

                        beanReferences.add(new RuntimeBeanReference(aspectName));
                    }

                    //如果是五种通知类型之一的话，执行下面语句
                    AbstractBeanDefinition advisorDefinition = this.parseAdvice(aspectName, aspectComponentDefinition, aspectElement, (Element)pointcuts, parserContext, beanDefinitions, beanReferences);
                    beanDefinitions.add(advisorDefinition);
                }
            }

            AspectComponentDefinition var19 = this.createAspectComponentDefinition(aspectElement, aspectId, beanDefinitions, beanReferences, parserContext);
            parserContext.pushContainingComponent(var19);
            
            //处理<aop:pointcut>流程
            List var20 = DomUtils.getChildElementsByTagName(aspectElement, "pointcut");
            Iterator var21 = var20.iterator();

            while(var21.hasNext()) {
                Element pointcutElement = (Element)var21.next();
                this.parsePointcut(pointcutElement, parserContext);
            }

            parserContext.popAndRegisterContainingComponent();
        } finally {
            this.parseState.pop();
        }
    }
```

##### parseAdvice源码
```java
private AbstractBeanDefinition parseAdvice(String aspectName, int order, Element aspectElement, Element adviceElement, ParserContext parserContext, List<BeanDefinition> beanDefinitions, List<BeanReference> beanReferences) {
        RootBeanDefinition var12;
        try {
            this.parseState.push(new AdviceEntry(parserContext.getDelegate().getLocalName(adviceElement)));
            
            // create the method factory bean
            RootBeanDefinition methodDefinition = new RootBeanDefinition(MethodLocatingFactoryBean.class);
            methodDefinition.getPropertyValues().add("targetBeanName", aspectName);
            methodDefinition.getPropertyValues().add("methodName", adviceElement.getAttribute("method"));
            methodDefinition.setSynthetic(true);
            
            //create instance factory definition
            RootBeanDefinition aspectFactoryDef = new RootBeanDefinition(SimpleBeanFactoryAwareAspectInstanceFactory.class);
            aspectFactoryDef.getPropertyValues().add("aspectBeanName", aspectName);
            aspectFactoryDef.setSynthetic(true);
            
            //register the pointcut。根据织入方式将<aop:before>、<aop:after>转换成名为adviceDef的RootBeanDefinition
            AbstractBeanDefinition adviceDef = this.createAdviceDefinition(adviceElement, parserContext, aspectName, order, methodDefinition, aspectFactoryDef, beanDefinitions, beanReferences);
            
            //configure the advisor。将名为adviceDef的RootBeanDefinition转换成名为advisorDefinition的RootBeanDefinition
            RootBeanDefinition advisorDefinition = new RootBeanDefinition(AspectJPointcutAdvisor.class);
            advisorDefinition.setSource(parserContext.extractSource(adviceElement));
            advisorDefinition.getConstructorArgumentValues().addGenericArgumentValue(adviceDef);
            if(aspectElement.hasAttribute("order")) {
                advisorDefinition.getPropertyValues().add("order", aspectElement.getAttribute("order"));
            }
            
            //register the final advisor。将BeanDefinition注册到DefaultListableBeanFactory中
            parserContext.getReaderContext().registerWithGeneratedName(advisorDefinition);
            var12 = advisorDefinition;
        } finally {
            this.parseState.pop();
        }

        return var12;
    }
```
功能：

1. 根据织入方式（before、after这些）创建RootBeanDefinition，名为adviceDef即advice定义
2. 将上一步创建的RootBeanDefinition写入一个新的RootBeanDefinition，构造一个新的对象，名为advisorDefinition，即advisor定义
3. 将advisorDefinition注册到DefaultListableBeanFactory中


##### createAdviceDefinition源码
```java
private AbstractBeanDefinition createAdviceDefinition(Element adviceElement, ParserContext parserContext, String aspectName, int order, RootBeanDefinition methodDef, RootBeanDefinition aspectFactoryDef, List<BeanDefinition> beanDefinitions, List<BeanReference> beanReferences) {
        
        //创建的AbstractBeanDefinition实例是RootBeanDefinition，这和普通Bean创建的实例为GenericBeanDefinition不同
        RootBeanDefinition adviceDefinition = new RootBeanDefinition(this.getAdviceClass(adviceElement, parserContext));
        adviceDefinition.setSource(parserContext.extractSource(adviceElement));
        adviceDefinition.getPropertyValues().add("aspectName", aspectName);
        adviceDefinition.getPropertyValues().add("declarationOrder", Integer.valueOf(order));
        if(adviceElement.hasAttribute("returning")) {
            adviceDefinition.getPropertyValues().add("returningName", adviceElement.getAttribute("returning"));
        }

        if(adviceElement.hasAttribute("throwing")) {
            adviceDefinition.getPropertyValues().add("throwingName", adviceElement.getAttribute("throwing"));
        }

        if(adviceElement.hasAttribute("arg-names")) {
            adviceDefinition.getPropertyValues().add("argumentNames", adviceElement.getAttribute("arg-names"));
        }

        ConstructorArgumentValues cav = adviceDefinition.getConstructorArgumentValues();
        cav.addIndexedArgumentValue(0, methodDef);
        Object pointcut = this.parsePointcutProperty(adviceElement, parserContext);
        if(pointcut instanceof BeanDefinition) {
            cav.addIndexedArgumentValue(1, pointcut);
            beanDefinitions.add((BeanDefinition)pointcut);
        } else if(pointcut instanceof String) {
            RuntimeBeanReference pointcutRef = new RuntimeBeanReference((String)pointcut);
            cav.addIndexedArgumentValue(1, pointcutRef);
            beanReferences.add(pointcutRef);
        }

        cav.addIndexedArgumentValue(2, aspectFactoryDef);
        return adviceDefinition;
    }
```

##### getAdviceClass源码
```java
private Class<?> getAdviceClass(Element adviceElement, ParserContext parserContext) {
        String elementName = parserContext.getDelegate().getLocalName(adviceElement);
        if("before".equals(elementName)) {
            return AspectJMethodBeforeAdvice.class;
        } else if("after".equals(elementName)) {
            return AspectJAfterAdvice.class;
        } else if("after-returning".equals(elementName)) {
            return AspectJAfterReturningAdvice.class;
        } else if("after-throwing".equals(elementName)) {
            return AspectJAfterThrowingAdvice.class;
        } else if("around".equals(elementName)) {
            return AspectJAroundAdvice.class;
        } else {
            throw new IllegalArgumentException("Unknown advice kind [" + elementName + "].");
        }
    }
```
既然创建Bean定义，必然该Bean定义中要对应一个具体的Class，不同的切入方式对应不同的Class：

通知类型 |  对应Class
---|---
before | AspectJMethodBeforeAdvice
after | AspectJAfterAdvice
after-returning | AspectJAfterReturningAdvice
after-throwing | AspectJAfterThrowingAdvice
around | AspectJAroundAdvice


##### parsePointcut源码
在ConfigBeanDefinitionParser类的parseAspect方法处理中，在处理完advice后，会执行parsePointcut进行`<aop:pointcut>`的解析。
```java
private AbstractBeanDefinition parsePointcut(Element pointcutElement, ParserContext parserContext) {
        //获取<aop:pointcut>标签下的"id"属性与"expression"属性
        String id = pointcutElement.getAttribute("id");
        String expression = pointcutElement.getAttribute("expression");
        AbstractBeanDefinition pointcutDefinition = null;

        try {
            //推送一个PointcutEntry，表示当前Spring上下文正在解析Pointcut标签
            this.parseState.push(new PointcutEntry(id));
            
            //创建Pointcut的Bean定义
            pointcutDefinition = this.createPointcutDefinition(expression);
            pointcutDefinition.setSource(parserContext.extractSource(pointcutElement));
            String pointcutBeanName = id;
            if(StringUtils.hasText(id)) {   //如果<aop:pointcut>标签中配置了id属性就执行
                parserContext.getRegistry().registerBeanDefinition(id, pointcutDefinition);
            } else {    //如果<aop:pointcut>标签中没有配置id属性就执行,和Bean不配置id属性一样的规则,pointcutBeanName=org.springframework.aop.aspectj.AspectJExpressionPointcut#序号（从0开始累加）
                pointcutBeanName = parserContext.getReaderContext().registerWithGeneratedName(pointcutDefinition);
            }
            
            //向解析工具上下文中注册一个Pointcut组件定义
            parserContext.registerComponent(new PointcutComponentDefinition(pointcutBeanName, pointcutDefinition, expression));
        } finally {
            //在<aop:pointcut>标签解析完毕后，让之前推送至栈顶的PointcutEntry出栈，表示此次<aop:pointcut>标签解析完毕。
            this.parseState.pop();
        }

        return pointcutDefinition;
    }
```

##### createPointcutDefinition源码
```java
protected AbstractBeanDefinition createPointcutDefinition(String expression) {
        RootBeanDefinition beanDefinition = new RootBeanDefinition(AspectJExpressionPointcut.class);
        beanDefinition.setScope("prototype");
        beanDefinition.setSynthetic(true);
        beanDefinition.getPropertyValues().add("expression", expression);
        return beanDefinition;
    }
```
关键点：
1. <aop:pointcut>标签对应解析出来的BeanDefinition是RootBeanDefinition，且RootBenaDefinitoin中的Class是org.springframework.aop.aspectj.AspectJExpressionPointcut
2. <aop:pointcut>标签对应的Bean是prototype即原型的


整个流程下来，就解析了`<aop:pointcut>`标签中的内容并将之转换为RootBeanDefintion存储在Spring容器中。


#### AOP关键类AspectJAwareAdvisorAutoProxyCreator解析

在读取AOP配置文件时，可知`org.springframework.aop.aspectj.autoproxy.AspectJAwareAdvisorAutoProxyCreator`这个类是Spring AOP的核心类，就是AspectJAwareAdvisorAutoProxyCreator完成了【类/接口-->代理】的转换过程。

类图：
![AspectJAwareAdvisorAutoProxyCreator继承图](http://thumbsnap.com/i/NdzbZHJ7.jpg)
由类图，可知：
1. AspectJAwareAdvisorAutoProxyCreator实现了BeanPostProcessor接口。
2. 在每个Bean初始化之后，如果需要，调用AspectJAwareAdvisorAutoProxyCreator中的`postProcessBeforeInitialization`或者`postProcessAfterInitialization`为Bean生成代理。

##### AOP代理类生成
在 [Bean加载流程之BeanWrapper构建](https://houlong123.github.io/2017/08/12/Bean%E5%8A%A0%E8%BD%BD%E6%B5%81%E7%A8%8B%E4%B9%8BBeanWrapper%E6%9E%84%E5%BB%BA/) 文章中，讲解到了对非懒加载Bean的初始化。

##### initializeBean源码

```java
protected Object initializeBean(final String beanName, final Object bean, RootBeanDefinition mbd) {
        if(System.getSecurityManager() != null) {
            AccessController.doPrivileged(new PrivilegedAction() {
                public Object run() {
                    AbstractAutowireCapableBeanFactory.this.invokeAwareMethods(beanName, bean);
                    return null;
                }
            }, this.getAccessControlContext());
        } else {
            //Aware注入。在使用Spring的时候我们将自己的Bean实现BeanNameAware接口、BeanFactoryAware接口等，依赖容器帮我们注入当前Bean的名称或者Bean工厂
            this.invokeAwareMethods(beanName, bean);
        }
        Object wrappedBean = bean;
        if(mbd == null || !mbd.isSynthetic()) {
            //执行BeanPostProcessor接口方法，在bean初始化之前执行
            wrappedBean = this.applyBeanPostProcessorsBeforeInitialization(bean, beanName);
        }
        try {
            //如果bean实现了InitializingBean或者设置了init-method属性，则执行相应的操作
            this.invokeInitMethods(beanName, wrappedBean, mbd);
        } catch (Throwable var6) {
            throw new BeanCreationException(mbd != null?mbd.getResourceDescription():null, beanName, "Invocation of init method failed", var6);
        }
        if(mbd == null || !mbd.isSynthetic()) {
              //执行BeanPostProcessor接口方法，在bean初始化之后执行
            wrappedBean = this.applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
        }
        return wrappedBean;
    }
```
有上面代码可知，在bean初始化之前调用BeanPostProcessor接口的applyBeanPostProcessorsBeforeInitialization方法，初始化之后调用BeanPostProcessor接口的applyBeanPostProcessorsAfterInitialization方法。由源码可知，相应的applyBeanPostProcessorsBeforeInitialization方法没有添加任何多余的实现，代理类生成的逻辑代码在postProcessAfterInitialization方法中。

##### applyBeanPostProcessorsAfterInitialization源码

```java
public Object applyBeanPostProcessorsAfterInitialization(Object existingBean, String beanName) throws BeansException {
        Object result = existingBean;
        Iterator var4 = this.getBeanPostProcessors().iterator();

        do {
            if(!var4.hasNext()) {
                return result;
            }

            BeanPostProcessor beanProcessor = (BeanPostProcessor)var4.next();
            result = beanProcessor.postProcessAfterInitialization(result, beanName);
        } while(result != null);

        return result;
    }
```
功能：循环调用每个BeanPostProcessor接口实现类的postProcessAfterInitialization方法，由之前的分析，可知，AOP代理类生成会进入`AbstractAutoProxyCreator`类

##### postProcessAfterInitialization源码

```
//AbstractAutoProxyCreator类中postProcessAfterInitialization方法
public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
        if(bean != null) {
            Object cacheKey = this.getCacheKey(bean.getClass(), beanName);
            if(!this.earlyProxyReferences.contains(cacheKey)) {
                return this.wrapIfNecessary(bean, beanName, cacheKey);
            }
        }

        return bean;
    }
```

##### wrapIfNecessary源码

```java
protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
        
        
        if(beanName != null && this.targetSourcedBeans.contains(beanName)) { //不需要生成代理的场景判断
            return bean;
        } else if(Boolean.FALSE.equals(this.advisedBeans.get(cacheKey))) { //不需要生成代理的场景判断
            return bean;
        } else if(!this.isInfrastructureClass(bean.getClass()) && !this.shouldSkip(bean.getClass(), beanName)) {
            // 寻找合适的Advisor
            Object[] specificInterceptors = this.getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, (TargetSource)null);
            if(specificInterceptors != DO_NOT_PROXY) {  //需要生成代理的目标对象不为空时
                this.advisedBeans.put(cacheKey, Boolean.TRUE);
                //为需要生成代理的目标对象创建代理
                Object proxy = this.createProxy(bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));
                this.proxyTypes.put(cacheKey, proxy.getClass());
                return proxy;
            } else {
                this.advisedBeans.put(cacheKey, Boolean.FALSE);
                return bean;
            }
        } else {
            this.advisedBeans.put(cacheKey, Boolean.FALSE);
            return bean;
        }
    }
```
功能：上述源码的功能主要就是判断目标对象是否需要生成代理。需要，则创建代理类；不需要，则返回原类。

##### findEligibleAdvisors源码
在目标对象需要被代理时，需要为目标对象去寻找合适的Advisor。

```java
//为指定class寻找合适的Advisor
protected List<Advisor> findEligibleAdvisors(Class<?> beanClass, String beanName) {
        //根据上文的配置文件,寻找所有候选Advisors。在XML解析的时候已经被转换生成了RootBeanDefinition。
        List candidateAdvisors = this.findCandidateAdvisors();
        
        //获取匹配的Advisors
        List eligibleAdvisors = this.findAdvisorsThatCanApply(candidateAdvisors, beanClass, beanName);
       
        //向候选Advisor链的开头（也就是List.get(0)的位置添加一个org.springframework.aop.support.DefaultPointcutAdvisor。
        this.extendAdvisors(eligibleAdvisors);
        
        if(!eligibleAdvisors.isEmpty()) {
            eligibleAdvisors = this.sortAdvisors(eligibleAdvisors);
        }

        return eligibleAdvisors;
    }
```

##### findAdvisorsThatCanApply源码

```java
protected List<Advisor> findAdvisorsThatCanApply(List<Advisor> candidateAdvisors, Class<?> beanClass, String beanName) {
        ProxyCreationContext.setCurrentProxiedBeanName(beanName);

        List var4;
        try {
            //调用AopUtils中相应方法
            var4 = AopUtils.findAdvisorsThatCanApply(candidateAdvisors, beanClass);
        } finally {
            ProxyCreationContext.setCurrentProxiedBeanName((String)null);
        }

        return var4;
    }
```

##### AopUtils类中findAdvisorsThatCanApply源码

```java
//AopUtils类中findAdvisorsThatCanApply源码
public static List<Advisor> findAdvisorsThatCanApply(List<Advisor> candidateAdvisors, Class<?> clazz) {
        if(candidateAdvisors.isEmpty()) {
            return candidateAdvisors;
        } else {
            LinkedList eligibleAdvisors = new LinkedList();
            Iterator hasIntroductions = candidateAdvisors.iterator();

            while(hasIntroductions.hasNext()) {
                Advisor candidate = (Advisor)hasIntroductions.next();
                if(candidate instanceof IntroductionAdvisor && canApply(candidate, clazz)) {
                    eligibleAdvisors.add(candidate);
                }
            }

            boolean hasIntroductions1 = !eligibleAdvisors.isEmpty();
            Iterator candidate2 = candidateAdvisors.iterator();

            while(candidate2.hasNext()) {
                Advisor candidate1 = (Advisor)candidate2.next();
                
                //PointcutAdvisor处理
                if(!(candidate1 instanceof IntroductionAdvisor) && canApply(candidate1, clazz, hasIntroductions1)) {
                    eligibleAdvisors.add(candidate1);
                }
            }

            return eligibleAdvisors;
        }
    }
```
由源码可知，整个方法判断都是围绕`canApply`方法展开

##### canApply源码

```java
public static boolean canApply(Advisor advisor, Class<?> targetClass, boolean hasIntroductions) {
        if(advisor instanceof IntroductionAdvisor) {
            return ((IntroductionAdvisor)advisor).getClassFilter().matches(targetClass);
        } else if(advisor instanceof PointcutAdvisor) {  //由于实例使用的是<aop:pointcut>,advisord的实际类型是AspectJPointcutAdvisor，是PointcutAdvisor的子类，因此下面代码执行
            PointcutAdvisor pca = (PointcutAdvisor)advisor;
            return canApply(pca.getPointcut(), targetClass, hasIntroductions);
        } else {
            return true;
        }
    }
```

##### canApply源码

```java
//实例中实际类型是AspectJPointcutAdvisor，是PointcutAdvisor的子类
public static boolean canApply(Pointcut pc, Class<?> targetClass, boolean hasIntroductions) {
        Assert.notNull(pc, "Pointcut must not be null");
        if(!pc.getClassFilter().matches(targetClass)) {  //是否匹配<aop:pointcut id="addAllMethod" expression="execution(* org.xrq.action.aop.Dao.*(..))" />中的expression表达式，不匹配，返回false
            return false;
        } else {
            //判断目标类中的方法是否匹配<aop:pointcut>中的expression表达式
            MethodMatcher methodMatcher = pc.getMethodMatcher();
            if(methodMatcher == MethodMatcher.TRUE) {
                return true;
            } else {
                IntroductionAwareMethodMatcher introductionAwareMethodMatcher = null;
                if(methodMatcher instanceof IntroductionAwareMethodMatcher) {
                    introductionAwareMethodMatcher = (IntroductionAwareMethodMatcher)methodMatcher;
                }
                
                //获取目标类的所有实现接口
                LinkedHashSet classes = new LinkedHashSet(ClassUtils.getAllInterfacesForClassAsSet(targetClass));
                classes.add(targetClass);
                Iterator var6 = classes.iterator();
                
                //循环遍历目标类所实现接口中的所有方法，判断是否匹配<aop:pointcut>中的expression表达式，只要有一个匹配，则返回true
                while(var6.hasNext()) {
                    Class clazz = (Class)var6.next();
                    Method[] methods = ReflectionUtils.getAllDeclaredMethods(clazz);
                    Method[] var9 = methods;
                    int var10 = methods.length;

                    for(int var11 = 0; var11 < var10; ++var11) {
                        Method method = var9[var11];
                        if(introductionAwareMethodMatcher != null && introductionAwareMethodMatcher.matches(method, targetClass, hasIntroductions) || methodMatcher.matches(method, targetClass)) {
                            return true;
                        }
                    }
                }

                return false;
            }
        }
    }
```
以上源码主要是获取Advisor数组。在Advisor数组不为空时，进入目标类的代理类创建流程。


##### createProxy源码

```java
 protected Object createProxy(Class<?> beanClass, String beanName, Object[] specificInterceptors, TargetSource targetSource) {
        if(this.beanFactory instanceof ConfigurableListableBeanFactory) {
            AutoProxyUtils.exposeTargetClass((ConfigurableListableBeanFactory)this.beanFactory, beanName, beanClass);
        }

        ProxyFactory proxyFactory = new ProxyFactory();
        proxyFactory.copyFrom(this);
        
        //判断<aop:config>这个节点中proxy-target-class的配置，为true，即使用CGLIB生成代理
        if(!proxyFactory.isProxyTargetClass()) {
            if(this.shouldProxyTargetClass(beanClass, beanName)) {
                proxyFactory.setProxyTargetClass(true);
            } else {
                 //获取当前Bean实现的所有接口，讲这些接口Class对象都添加到ProxyFactory中。
                 this.evaluateProxyInterfaces(beanClass, proxyFactory);
            }
        }

        //配置proxyFactory
        Advisor[] advisors = this.buildAdvisors(beanName, specificInterceptors);
        Advisor[] var7 = advisors;
        int var8 = advisors.length;

        for(int var9 = 0; var9 < var8; ++var9) {
            Advisor advisor = var7[var9];
            proxyFactory.addAdvisor(advisor);
        }

        proxyFactory.setTargetSource(targetSource);
        this.customizeProxyFactory(proxyFactory);
        proxyFactory.setFrozen(this.freezeProxy);
        if(this.advisorsPreFiltered()) {
            proxyFactory.setPreFiltered(true);
        }
    
        //创建代理类
        return proxyFactory.getProxy(this.getProxyClassLoader());
    }
```

##### getProxy源码
```java
public Object getProxy(ClassLoader classLoader) {
        return this.createAopProxy().getProxy(classLoader);
    }
```
由源码可知，Spring框架会去获取AopProxy对象，即使用JDK代理还是使用Cglib代理。跟踪createAopProxy方法，最终定位到`DefaultAopProxyFactory`类中的createAopProxy方法。

##### DefaultAopProxyFactory类createAopProxy源码

```java
public AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException {
        if(!config.isOptimize() && !config.isProxyTargetClass() && !this.hasNoUserSuppliedProxyInterfaces(config)) {
            return new JdkDynamicAopProxy(config);
        } else {
            Class targetClass = config.getTargetClass();
            if(targetClass == null) {
                throw new AopConfigException("TargetSource cannot determine target class: Either an interface or a target is required for proxy creation.");
            } else {
                return (AopProxy)(!targetClass.isInterface() && !Proxy.isProxyClass(targetClass)?new ObjenesisCglibAopProxy(config):new JdkDynamicAopProxy(config));
            }
        }
    }
```
功能分析：
1. 使用JDK代理的情况
    + 当ProxyConfig的isOptimize方法为false，即` 不让Spring自己去优化`
    + ProxyConfig的isProxyTargetClass方法为false，即`配置了proxy-target-class="false"`
    + ProxyConfig满足hasNoUserSuppliedProxyInterfaces方法执行结果为false，即`<bean>对象实现除SpringProxy接口之外的其他接口`

2. 使用CGLIB代理的情况
   + `isOptimize = true` || `配置proxy-target-class="true"` || `没有实现任何接口或者实现的接口是SpringProxy接口`
   + 目标类不是接口
   + 不是通过`getProxyClass` 或 `newProxyInstance`获取的类


#### AOP代理之JDK动态代理
由`createAopProxy`可知，AOP实现代理的方式有两种：JDK动态代理和CGLIB代理。下面讲解一下JDK动态代理。

##### getProxy源码
```java
public Object getProxy(ClassLoader classLoader) {
        if(logger.isDebugEnabled()) {
            logger.debug("Creating JDK dynamic proxy: target source is " + this.advised.getTargetSource());
        }
        
        //获取所有要代理的接口
        Class[] proxiedInterfaces = AopProxyUtils.completeProxiedInterfaces(this.advised, true);
        
        //寻找要代理接口方法中有没有equals和hashCode方法，有，做标记。同时都有时，寻找结束。
        this.findDefinedEqualsAndHashCodeMethods(proxiedInterfaces);
        
        //获取代理对象。由于JdkDynamicAopProxy类继承了InvocationHandler接口，因此在调用newProxyInstance方法传递第三个参数时，直接传递JdkDynamicAopProxy类本身即可
        return Proxy.newProxyInstance(classLoader, proxiedInterfaces, this);
    }
```

##### invoke源码
在JDK动态代理中， `InvocationHandler接口`和 `Proxy类`是实现我们动态代理所必须用到的。当我们通过代理对象调用一个方法的时候，这个方法的调用就会被转发为由InvocationHandler这个接口的 invoke 方法来进行调用。
```java
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        Object oldProxy = null;
        boolean setProxyContext = false;
        TargetSource targetSource = this.advised.targetSource;
        Class targetClass = null;
        Object target = null;

        Object retVal;
        try {
            //以下两个if语句表明：equals方法与hashCode方法即使满足expression规则，也不会为之产生代理内容，调用的是JdkDynamicAopProxy的equals方法与hashCode方法
            if(!this.equalsDefined && AopUtils.isEqualsMethod(method)) {
                // The target does not implement the equals(Object) method itself.
                Boolean retVal3 = Boolean.valueOf(this.equals(args[0]));
                return retVal3;
            }

            if(!this.hashCodeDefined && AopUtils.isHashCodeMethod(method)) {
                // The target does not implement the hashCode() method itself.
                Integer retVal2 = Integer.valueOf(this.hashCode());
                return retVal2;
            }

            if(method.getDeclaringClass() == DecoratingProxy.class) {
                Class retVal1 = AopProxyUtils.ultimateTargetClass(this.advised);
                return retVal1;
            }

            if(this.advised.opaque || !method.getDeclaringClass().isInterface() || !method.getDeclaringClass().isAssignableFrom(Advised.class)) {
                if(this.advised.exposeProxy) { 
                    // Make invocation available if necessary.
                    oldProxy = AopContext.setCurrentProxy(proxy);
                    setProxyContext = true;
                }

                target = targetSource.getTarget();
                if(target != null) {
                    targetClass = target.getClass();
                }
                
                //获取AdvisedSupport中的所有拦截器和动态拦截器列表，用于拦截方法
                List chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);
                if(chain.isEmpty()) {
                    //如果拦截器列表为空，即某个类/接口下的某个方法可能不满足expression的匹配规则，因此此时通过反射直接调用该方法。
                    Object[] returnType = AopProxyUtils.adaptArgumentsIfNecessary(method, args);
                    retVal = AopUtils.invokeJoinpointUsingReflection(target, method, returnType);
                } else {
                    //如果拦截器列表不为空，需要一个ReflectiveMethodInvocation，并通过proceed方法对原方法进行拦截。
                    ReflectiveMethodInvocation invocation = new ReflectiveMethodInvocation(proxy, target, method, args, targetClass, chain);
                    retVal = invocation.proceed();
                }

                Class returnType1 = method.getReturnType();
                if(retVal != null && retVal == target && returnType1 != Object.class && returnType1.isInstance(proxy) && !RawTargetAccess.class.isAssignableFrom(method.getDeclaringClass())) {
                    retVal = proxy;
                } else if(retVal == null && returnType1 != Void.TYPE && returnType1.isPrimitive()) {
                    throw new AopInvocationException("Null return value from advice does not match primitive return type for: " + method);
                }

                Object var13 = retVal;
                return var13;
            }

            //方法所属的Class是一个接口并且方法所属的Class是AdvisedSupport的父类或者父接口，直接通过反射调用该方法。
            retVal = AopUtils.invokeJoinpointUsingReflection(this.advised, method, args);
        } finally {
            if(target != null && !targetSource.isStatic()) {
                targetSource.releaseTarget(target);
            }

            if(setProxyContext) {
                AopContext.setCurrentProxy(oldProxy);
            }

        }

        return retVal;
    }
```

#### AOP代理之CGLIB代理

##### getProxy源码
```java
public Object getProxy(ClassLoader classLoader) {
        if(logger.isDebugEnabled()) {
            logger.debug("Creating CGLIB proxy: target source is " + this.advised.getTargetSource());
        }

        try {
            Class ex = this.advised.getTargetClass();
            Assert.state(ex != null, "Target class must be available for creating a CGLIB proxy");
            Class proxySuperClass = ex;
            int x;
            if(ClassUtils.isCglibProxyClass(ex)) {
                proxySuperClass = ex.getSuperclass();
                Class[] enhancer = ex.getInterfaces();
                Class[] callbacks = enhancer;
                int types = enhancer.length;

                for(x = 0; x < types; ++x) {
                    Class additionalInterface = callbacks[x];
                    this.advised.addInterface(additionalInterface);
                }
            }

            this.validateClassIfNecessary(proxySuperClass, classLoader);
            
            //获取Enhancer对象，这个CGLIB代理实现中重要的类
            Enhancer var12 = this.createEnhancer();
            if(classLoader != null) {
                var12.setClassLoader(classLoader);
                if(classLoader instanceof SmartClassLoader && ((SmartClassLoader)classLoader).isClassReloadable(proxySuperClass)) {
                    var12.setUseCache(false);
                }
            }

            var12.setSuperclass(proxySuperClass);
            var12.setInterfaces(AopProxyUtils.completeProxiedInterfaces(this.advised));
            var12.setNamingPolicy(SpringNamingPolicy.INSTANCE);
            var12.setStrategy(new CglibAopProxy.ClassLoaderAwareUndeclaredThrowableStrategy(classLoader));
            Callback[] var13 = this.getCallbacks(ex);
            Class[] var14 = new Class[var13.length];

            for(x = 0; x < var14.length; ++x) {
                var14[x] = var13[x].getClass();
            }

            var12.setCallbackFilter(new CglibAopProxy.ProxyCallbackFilter(this.advised.getConfigurationOnlyCopy(), this.fixedInterceptorMap, this.fixedInterceptorOffset));
            var12.setCallbackTypes(var14);
            
            //生成代理类
            return this.createProxyClassAndInstance(var12, var13);
        } catch (CodeGenerationException var9) {
            throw new AopConfigException("Could not generate CGLIB subclass of class [" + this.advised.getTargetClass() + "]: Common causes of this problem include using a final class or a non-visible class", var9);
        } catch (IllegalArgumentException var10) {
            throw new AopConfigException("Could not generate CGLIB subclass of class [" + this.advised.getTargetClass() + "]: Common causes of this problem include using a final class or a non-visible class", var10);
        } catch (Exception var11) {
            throw new AopConfigException("Unexpected AOP exception", var11);
        }
    }
```

##### createProxyClassAndInstance源码

```java
protected Object createProxyClassAndInstance(Enhancer enhancer, Callback[] callbacks) {
        enhancer.setInterceptDuringConstruction(false);
        enhancer.setCallbacks(callbacks);
        return this.constructorArgs != null?enhancer.create(this.constructorArgTypes, this.constructorArgs):enhancer.create();
    }
```
关于Cglib，可以参考 [Cglib及其基本使用
](http://www.cnblogs.com/xrq730/p/6661692.html) 一文。


