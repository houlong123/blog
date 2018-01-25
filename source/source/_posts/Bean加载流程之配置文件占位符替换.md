---
title: Bean加载流程之配置文件占位符替换
date: 2017-08-18 14:09:59
tags: Spring
---

在讲解properties文件中${...}替换之前，首先介绍一下`BeanFactoryPostProcessor`类。

#### BeanFactoryPostProcessor讲解
`BeanFactoryPostProcessor`和`BeanPostProcessor`，这两个接口，都是spring初始化bean时`对外暴露的扩展点`。两个接口名称看起来很相似，但作用及使用场景却不同。

+ BeanFactoryPostProcessor 接口
    
    源码
    
    ```java
    public interface BeanFactoryPostProcessor {
        void postProcessBeanFactory(ConfigurableListableBeanFactory var1) throws BeansException;
    }
    ```

    此接口只有一个方法，接受`ConfigurableListableBeanFactory`参数。实现该接口，可以在spring的`bean创建之前，修改bean的定义属性`。即读取bean的配置元数据，并可以根据需要进行修改(直白点，就是修改bean的BeanDefinition中相应信息)。由[Bean加载流程梳理之BeanDefinitions加载](https://houlong123.github.io/2017/08/10/Bean%E5%8A%A0%E8%BD%BD%E6%B5%81%E7%A8%8B%E6%A2%B3%E7%90%861/) 可知，在获取BeanFactory时，即接口中唯一方法的参数，Spring框架会将加载的BeanDefinitions放进BeanFactory，因此该接口中的方法可以获取所有bean的BeanDefinition。然后进行相应修改。
    *`BeanFactoryPostProcessor是在spring容器加载了bean的定义文件之后，在bean实例化之前执行的`*。

<!-- more -->

+ BeanPostProcessor接口
    
    源码
    
    ```java
    public interface BeanPostProcessor {
        //spring容器实例化bean之后，在执行bean的初始化之前进行的处理
        Object postProcessBeforeInitialization(Object var1, String var2) throws BeansException;

       //spring容器实例化bean之后，在执行bean的初始化之后进行的处理
        Object postProcessAfterInitialization(Object var1, String var2) throws BeansException;
    }
    ```
    该接口中有两个方法，分别在bean初始化前后添加一些自己的处理逻辑。由`initializeBean`源码可知，这里说的初始化方法，指的是下面两种：
    1. bean实现了InitializingBean接口，对应的方法为afterPropertiesSet
    2. 在bean定义的时候，通过init-method设置的方法。
    
    initializeBean源码
    
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
            this.invokeAwareMethods(beanName, bean);
        }

        Object wrappedBean = bean;
        if(mbd == null || !mbd.isSynthetic()) {
            ////spring容器实例化bean之后，在执行bean的初始化之前进行的处理
            wrappedBean = this.applyBeanPostProcessorsBeforeInitialization(bean, beanName);
        }

        try {
            //bean初始化方法执行。该方法具体实现是判断bean是否继承InitializingBean，是，执行afterPropertiesSet方法；是否配置了init-method，设置了，执行相应方法。
            this.invokeInitMethods(beanName, wrappedBean, mbd);
        } catch (Throwable var6) {
            throw new BeanCreationException(mbd != null?mbd.getResourceDescription():null, beanName, "Invocation of init method failed", var6);
        }

        if(mbd == null || !mbd.isSynthetic()) {
            ////spring容器实例化bean之后，在执行bean的初始化之后进行的处理
            wrappedBean = this.applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
        }

        return wrappedBean;
    }
    ```
以上内容重点讲解了`BeanFactoryPostProcessor`接口，主要作用就是：在bean实例创建之前，可以根据需要修改bean的BeanDefinition，从而达到bean的修改。有了上面的基础，下面继续学习配置文件中占位符替换的知识。


#### 项目配置

+ applicationContext-database.xml配置
```xml
<?xml version="1.0" encoding="UTF-8"?>

<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:tx="http://www.springframework.org/schema/tx"
       xsi:schemaLocation="
     http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
     http://www.springframework.org/schema/tx http://www.springframework.org/schema/tx/spring-tx.xsd">

    <!-- 此类是重点，该类也是本次重点讲解对象 -->
    <bean id="propertyConfigurer" class="org.springframework.beans.factory.config.PropertyPlaceholderConfigurer">
        <property name="locations">
            <list>
                <value>classpath:properties/database.properties</value>
            </list>
        </property>
    </bean>
    
    <bean id="dataSource" class="com.alibaba.druid.pool.DruidDataSource" init-method="init" destroy-method="close">
        <property name="url" value="${mysql.url}" />
        <property name="username" value="${mysql.username}" />
        <property name="password" value="${mysql.password}" />
    </bean>
</beans>
```

+ database.properties配置文件

```properties
mysql.url=jdbc:mysql://db.frogshealth.com:3306/mcenter?characterEncoding=utf8&useUnicode=true
mysql.username=test
mysql.password=test
```
现在重点讲解如何将properties文件中的属性读入并替换"${}"占位符。

#### PropertyPlaceholderConfigurer源码
在applicationContext-database.xml文件中我们看到了一个类PropertyPlaceholderConfigurer，顾名思义它就是一个属性占位符配置器，看一下这个类的继承关系图。

类图：

![PropertyPlaceholderConfigurer类图](http://thumbsnap.com/i/E0nRN7TD.jpg)
由类图可知，PropertyPlaceholderConfigurer是BeanFactoryPostProcessor接口的实现类。由此可知，Spring上下文必然是在Bean定义全部加载完毕后且Bean实例化之前通过postProcessBeanFactory方法一次性地替换了占位符"${}"。

回顾一下，在加载beanDefinitions后(AbstractApplicationContext类中的refresh()方法)，会触发`BeanFactoryPostProcessors`的子类执行。

#### refresh源码

```java
public void refresh() throws BeansException, IllegalStateException {
        Object var1 = this.startupShutdownMonitor;
        synchronized(this.startupShutdownMonitor) {
            this.prepareRefresh();
           //获取BeanFactory对象的同时，会加载BeanDefinition
           ConfigurableListableBeanFactory beanFactory = this.obtainFreshBeanFactory();
            this.prepareBeanFactory(beanFactory);

            try {
                this.postProcessBeanFactory(beanFactory);
               
               //触发BeanFactoryPostProcessors的子类执行postProcessBeanFactory方法。
               this.invokeBeanFactoryPostProcessors(beanFactory);
                this.registerBeanPostProcessors(beanFactory);
                this.initMessageSource();
                this.initApplicationEventMulticaster();
                this.onRefresh();
                this.registerListeners();
                this.finishBeanFactoryInitialization(beanFactory);
                this.finishRefresh();
            } catch (BeansException var9) {
                if(this.logger.isWarnEnabled()) {
                    this.logger.warn("Exception encountered during context initialization - cancelling refresh attempt: " + var9);
                }

                this.destroyBeans();
                this.cancelRefresh(var9);
                throw var9;
            } finally {
                this.resetCommonCaches();
            }

        }
    }
```


#### invokeBeanFactoryPostProcessors源码

```java
//为AbstractApplicationContext类中相应方法
protected void invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory beanFactory) {
        //通过代理对象，转移职责
        PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors(beanFactory, this.getBeanFactoryPostProcessors());
        if(beanFactory.getTempClassLoader() == null && beanFactory.containsBean("loadTimeWeaver")) {
            beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
            beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
        }

    }
```


```java
//PostProcessorRegistrationDelegate类中方法
public static void invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory beanFactory, List<BeanFactoryPostProcessor> beanFactoryPostProcessors) {
        HashSet processedBeans = new HashSet();
        int postProcessorName;
        ArrayList var20;
        ArrayList var23;
        //从获取BeanFactory的源码中可知我们使用的是DefaultListableBeanFactory，它是BeanDefinitionRegistry的子类
        if(beanFactory instanceof BeanDefinitionRegistry) {
            BeanDefinitionRegistry postProcessorNames = (BeanDefinitionRegistry)beanFactory;
            LinkedList priorityOrderedPostProcessors = new LinkedList();
            LinkedList orderedPostProcessorNames = new LinkedList();
            
            //beanFactoryPostProcessors是通过AbstractApplicationContext.getBeanFactoryPostProcessors()获取的，即这些BeanFactoryPostProcessor是通过AbstractApplicationContext的addBeanFactoryPostProcessor方法添加的而不是配置文件里面配置的BeanFactoryPostProcessor的实现Bean，因此这个判断没有任何可执行的BeanFactoryPostProcessor。
            Iterator nonOrderedPostProcessorNames = beanFactoryPostProcessors.iterator();

            while(nonOrderedPostProcessorNames.hasNext()) {
                BeanFactoryPostProcessor orderedPostProcessors = (BeanFactoryPostProcessor)nonOrderedPostProcessorNames.next();
                if(orderedPostProcessors instanceof BeanDefinitionRegistryPostProcessor) {
                    BeanDefinitionRegistryPostProcessor nonOrderedPostProcessors = (BeanDefinitionRegistryPostProcessor)orderedPostProcessors;
                    nonOrderedPostProcessors.postProcessBeanDefinitionRegistry(postProcessorNames);
                    orderedPostProcessorNames.add(nonOrderedPostProcessors);
                } else {
                    priorityOrderedPostProcessors.add(orderedPostProcessors);
                }
            }

            String[] var18 = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
            var20 = new ArrayList();
            String[] var22 = var18;
            postProcessorName = var18.length;

            int postProcessorName1;
            for(postProcessorName1 = 0; postProcessorName1 < postProcessorName; ++postProcessorName1) {
                String ppName = var22[postProcessorName1];
                if(beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
                    var20.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
                    processedBeans.add(ppName);
                }
            }

            sortPostProcessors(beanFactory, var20);
            orderedPostProcessorNames.addAll(var20);
            invokeBeanDefinitionRegistryPostProcessors(var20, postProcessorNames);
            var18 = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
            var23 = new ArrayList();
            String[] var26 = var18;
            postProcessorName1 = var18.length;

            int var30;
            for(var30 = 0; var30 < postProcessorName1; ++var30) {
                String ppName1 = var26[var30];
                if(!processedBeans.contains(ppName1) && beanFactory.isTypeMatch(ppName1, Ordered.class)) {
                    var23.add(beanFactory.getBean(ppName1, BeanDefinitionRegistryPostProcessor.class));
                    processedBeans.add(ppName1);
                }
            }

            sortPostProcessors(beanFactory, var23);
            orderedPostProcessorNames.addAll(var23);
            invokeBeanDefinitionRegistryPostProcessors(var23, postProcessorNames);
            boolean var27 = true;

            while(var27) {
                var27 = false;
                var18 = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
                String[] var28 = var18;
                var30 = var18.length;

                for(int var33 = 0; var33 < var30; ++var33) {
                    String ppName2 = var28[var33];
                    if(!processedBeans.contains(ppName2)) {
                        BeanDefinitionRegistryPostProcessor pp = (BeanDefinitionRegistryPostProcessor)beanFactory.getBean(ppName2, BeanDefinitionRegistryPostProcessor.class);
                        orderedPostProcessorNames.add(pp);
                        processedBeans.add(ppName2);
                        pp.postProcessBeanDefinitionRegistry(postProcessorNames);
                        var27 = true;
                    }
                }
            }

            invokeBeanFactoryPostProcessors((Collection)orderedPostProcessorNames, (ConfigurableListableBeanFactory)beanFactory);
            invokeBeanFactoryPostProcessors((Collection)priorityOrderedPostProcessors, (ConfigurableListableBeanFactory)beanFactory);
        } else {
            invokeBeanFactoryPostProcessors((Collection)beanFactoryPostProcessors, (ConfigurableListableBeanFactory)beanFactory);
        }

        //获取spring配置文件中定义的所有实现BeanFactoryPostProcessor接口的bean，然后根据优先级进行排序，之后对于每个BeanFactoryPostProcessor，调用postProcessBeanFactory方法。
        String[] var15 = beanFactory.getBeanNamesForType(BeanFactoryPostProcessor.class, true, false);
        ArrayList var16 = new ArrayList();
        ArrayList var17 = new ArrayList();
        ArrayList var19 = new ArrayList();
        String[] var21 = var15;
        int var24 = var15.length;

        String var29;
        for(postProcessorName = 0; postProcessorName < var24; ++postProcessorName) {
            var29 = var21[postProcessorName];
            if(!processedBeans.contains(var29)) {
                if(beanFactory.isTypeMatch(var29, PriorityOrdered.class)) {
                    var16.add(beanFactory.getBean(var29, BeanFactoryPostProcessor.class));
                } else if(beanFactory.isTypeMatch(var29, Ordered.class)) {
                    var17.add(var29);
                } else {
                    var19.add(var29);
                }
            }
        }

        sortPostProcessors(beanFactory, var16);
        invokeBeanFactoryPostProcessors((Collection)var16, (ConfigurableListableBeanFactory)beanFactory);
        var20 = new ArrayList();
        Iterator var25 = var17.iterator();

        while(var25.hasNext()) {
            String var31 = (String)var25.next();
            var20.add(beanFactory.getBean(var31, BeanFactoryPostProcessor.class));
        }

        sortPostProcessors(beanFactory, var20);
        invokeBeanFactoryPostProcessors((Collection)var20, (ConfigurableListableBeanFactory)beanFactory);
        var23 = new ArrayList();
        Iterator var32 = var19.iterator();

        while(var32.hasNext()) {
            var29 = (String)var32.next();
            var23.add(beanFactory.getBean(var29, BeanFactoryPostProcessor.class));
        }

        invokeBeanFactoryPostProcessors((Collection)var23, (ConfigurableListableBeanFactory)beanFactory);
        beanFactory.clearMetadataCache();
    }
```
功能分析：
1. 如果beanFactory为BeanDefinitionRegistry的子类，则先执行通过AbstractApplicationContext.addBeanFactoryPostProcessor()加进去的BeanFactoryPostProcessor接口子类的postProcessBeanFactory()方法
2. 如果有自定义的BeanFactoryPostProcessor子类，则执行Spring配置文件中自定义的BeanFactoryPostProcessor接口子类的postProcessBeanFactory()方法。

#### PropertyResourceConfigurer类中postProcessBeanFactory源码

在applicationContext-database.xml配置文件中，我们定义了名为propertyConfigurer的bean，该bean为BeanFactoryPostProcessor的子类。通过上述源码分析可知，在触发BeanFactoryPostProcessor子类执行时，PropertyPlaceholderConfigurer中的postProcessBeanFactory会执行，进行${}替换操作。
```java
public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
        try {
            //合并了.properties文件（之所以叫做合并是因为多个.properties文件中可能有相同的Key）
            Properties ex = this.mergeProperties();
           
            //必要的情况下对合并的P roperties进行转换
            this.convertProperties(ex);
           
            //开始替换占位符"${...}"
            this.processProperties(beanFactory, ex);
        } catch (IOException var3) {
            throw new BeanInitializationException("Could not load properties", var3);
        }
    }
```

#### doProcessProperties源码

在processProperties()方法中,通过PlaceholderResolvingStringValueResolver对象，即一个持有.properties文件配置的字符串值解析器，然后去执行PlaceholderConfigurerSupport类中的doProcessProperties()方法。
```java
protected void doProcessProperties(ConfigurableListableBeanFactory beanFactoryToProcess, StringValueResolver valueResolver) {
        //获取BeanDefinitionVistor对象，即一个Bean定义访问工具，持有字符串值解析器。然后可以通过BeanDefinitionVistor访问Bean定义，在遇到需要解析的字符串的时候使用构造函数传入的StringValueResolver解析字符串。
        BeanDefinitionVisitor visitor = new BeanDefinitionVisitor(valueResolver);
        
        //获取所有Bean定义的名称
        String[] beanNames = beanFactoryToProcess.getBeanDefinitionNames();
        String[] var5 = beanNames;
        int var6 = beanNames.length;

        //遍历所有Bean定义的名称，注意判断"!(curName.equals(this.beanName)"中，this.beanName指的是PropertyPlaceholderConfigurer，意为PropertyPlaceholderConfigurer本身不会去解析占位符"${...}"。
        for(int var7 = 0; var7 < var6; ++var7) {
            String curName = var5[var7];
            if(!curName.equals(this.beanName) || !beanFactoryToProcess.equals(this.beanFactory)) {
                //获取bean的BeanDefinition对象。
                BeanDefinition bd = beanFactoryToProcess.getBeanDefinition(curName);

                try {
                   
                 //调用BeanDefinitionVisitor对象的visitBeanDefinition()方法 visitor.visitBeanDefinition(bd);
                } catch (Exception var11) {
                    throw new BeanDefinitionStoreException(bd.getResourceDescription(), curName, var11.getMessage(), var11);
                }
            }
        }

        beanFactoryToProcess.resolveAliases(valueResolver);
        beanFactoryToProcess.addEmbeddedValueResolver(valueResolver);
    }
```
功能分析：
上述代码的功能主要是获取BeanDefinitionVisitor对象，然后遍历已加载的BeanDefinitions，调用BeanDefinitionVisitor对象的visitBeanDefinition()方法。

#### visitBeanDefinition源码
```java
public void visitBeanDefinition(BeanDefinition beanDefinition) {
        this.visitParentName(beanDefinition);
        this.visitBeanClassName(beanDefinition);
        this.visitFactoryBeanName(beanDefinition);
        this.visitFactoryMethodName(beanDefinition);
        this.visitScope(beanDefinition);
        this.visitPropertyValues(beanDefinition.getPropertyValues());
        ConstructorArgumentValues cas = beanDefinition.getConstructorArgumentValues();
        this.visitIndexedArgumentValues(cas.getIndexedArgumentValues());
        this.visitGenericArgumentValues(cas.getGenericArgumentValues());
    }
```
功能分析：
该方法轮番访问<bean>定义中的parent、class、factory-bean、factory-method、scope、property、constructor-arg属性，但凡遇到需要"${...}"就进行解析。这里解析的是property标签。


#### visitPropertyValues源码
```java
 protected void visitPropertyValues(MutablePropertyValues pvs) {
        PropertyValue[] pvArray = pvs.getPropertyValues();
        PropertyValue[] var3 = pvArray;
        int var4 = pvArray.length;
        
        //获取属性数组进行遍历
        for(int var5 = 0; var5 < var4; ++var5) {
            PropertyValue pv = var3[var5];
            //对属性值进行解析获取新属性值
            Object newVal = this.resolveValue(pv.getValue());
           
            //判断新属性值与原属性值是否相等，不等时，重新设置属性值
            if(!ObjectUtils.nullSafeEquals(newVal, pv.getValue())) {
                pvs.add(pv.getName(), newVal);
            }
        }

    }
```

#### resolveValue源码

```java
protected Object resolveValue(Object value) {
        if(value instanceof BeanDefinition) {
            this.visitBeanDefinition((BeanDefinition)value);
        } else if(value instanceof BeanDefinitionHolder) {
            this.visitBeanDefinition(((BeanDefinitionHolder)value).getBeanDefinition());
        } else {
            String stringValue;
            if(value instanceof RuntimeBeanReference) {
                RuntimeBeanReference typedStringValue = (RuntimeBeanReference)value;
                stringValue = this.resolveStringValue(typedStringValue.getBeanName());
                if(!stringValue.equals(typedStringValue.getBeanName())) {
                    return new RuntimeBeanReference(stringValue);
                }
            } else if(value instanceof RuntimeBeanNameReference) {
                RuntimeBeanNameReference typedStringValue1 = (RuntimeBeanNameReference)value;
                stringValue = this.resolveStringValue(typedStringValue1.getBeanName());
                if(!stringValue.equals(typedStringValue1.getBeanName())) {
                    return new RuntimeBeanNameReference(stringValue);
                }
            } else if(value instanceof Object[]) {
                this.visitArray((Object[])((Object[])value));
            } else if(value instanceof List) {
                this.visitList((List)value);
            } else if(value instanceof Set) {
                this.visitSet((Set)value);
            } else if(value instanceof Map) {
                this.visitMap((Map)value);
            } else if(value instanceof TypedStringValue) {
                TypedStringValue typedStringValue2 = (TypedStringValue)value;
                stringValue = typedStringValue2.getValue();
                if(stringValue != null) {
                    String visitedString = this.resolveStringValue(stringValue);
                    typedStringValue2.setValue(visitedString);
                }
            } else if(value instanceof String) {
                return this.resolveStringValue((String)value);
            }
        }

        return value;
    }
```
功能分析：

主要对value类型做一个判断，然后执行相应方法处理。已resolveStringValue()方法为例分析。

#### resolveStringValue源码

```java
protected String resolveStringValue(String strVal) {
        if(this.valueResolver == null) {
            throw new IllegalStateException("No StringValueResolver specified - pass a resolver object into the constructor or override the \'resolveStringValue\' method");
        } else {
            //调用PlaceholderResolvingStringValueResolver类中的resolveStringValue()方法
            String resolvedValue = this.valueResolver.resolveStringValue(strVal);
            return strVal.equals(resolvedValue)?strVal:resolvedValue;
        }
    }
```

#### resolveStringValue源码

```java
public String resolveStringValue(String strVal) throws BeansException {
            String resolved = this.helper.replacePlaceholders(strVal, this.resolver);
            if(PropertyPlaceholderConfigurer.this.trimValues) {
                resolved = resolved.trim();
            }

            return resolved.equals(PropertyPlaceholderConfigurer.this.nullValue)?null:resolved;
        }
```

经过一系列处理，会执行到PropertyPlaceholderHelper类中的parseStringValue()方法
```java
protected String parseStringValue(String strVal, PropertyPlaceholderHelper.PlaceholderResolver placeholderResolver, Set<String> visitedPlaceholders) {
        StringBuilder result = new StringBuilder(strVal);
        
        //获取占位符前缀"${"的位置索引startIndex
        int startIndex = strVal.indexOf(this.placeholderPrefix);

        while(startIndex != -1) {
            //占位符前缀"{"存在，从"{"后面开始获取占位符后缀"}"的位置索引endIndex
            int endIndex = this.findPlaceholderEndIndex(result, startIndex);
            if(endIndex != -1) {
                //如果占位符前缀位置索引startIndex与占位符后缀的位置索引endIndex都存在，截取中间的部分placeHolder
                String placeholder = result.substring(startIndex + this.placeholderPrefix.length(), endIndex);
                String originalPlaceholder = placeholder;
                if(!visitedPlaceholders.add(placeholder)) {
                    throw new IllegalArgumentException("Circular placeholder reference \'" + placeholder + "\' in property definitions");
                }

                placeholder = this.parseStringValue(placeholder, placeholderResolver, visitedPlaceholders);
                //从Properties中获取placeHolder对应的值propVal
                String propVal = placeholderResolver.resolvePlaceholder(placeholder);
                
                //如果propVal不存在，尝试对placeHolder使用":"进行一次分割，如果分割出来有结果，那么前面一部分命名为actualPlaceholder，后面一部分命名为defaultValue，尝试从Properties中获取actualPlaceholder对应的value，如果存在则取此value，如果不存在则取defaultValue，最终赋值给propVal

                if(propVal == null && this.valueSeparator != null) {
                    int separatorIndex = placeholder.indexOf(this.valueSeparator);
                    if(separatorIndex != -1) {
                        String actualPlaceholder = placeholder.substring(0, separatorIndex);
                        String defaultValue = placeholder.substring(separatorIndex + this.valueSeparator.length());
                        propVal = placeholderResolver.resolvePlaceholder(actualPlaceholder);
                        if(propVal == null) {
                            propVal = defaultValue;
                        }
                    }
                }

                if(propVal != null) {
                    propVal = this.parseStringValue(propVal, placeholderResolver, visitedPlaceholders);
                   
                    //替换占位符"${...}"的值
                    result.replace(startIndex, endIndex + this.placeholderSuffix.length(), propVal);
                    if(logger.isTraceEnabled()) {
                        logger.trace("Resolved placeholder \'" + placeholder + "\'");
                    }

                    startIndex = result.indexOf(this.placeholderPrefix, startIndex + propVal.length());
                } else {
                    if(!this.ignoreUnresolvablePlaceholders) {
                        throw new IllegalArgumentException("Could not resolve placeholder \'" + placeholder + "\' in string value \"" + strVal + "\"");
                    }

                    startIndex = result.indexOf(this.placeholderPrefix, endIndex + this.placeholderSuffix.length());
                }

                visitedPlaceholders.remove(originalPlaceholder);
            } else {
                startIndex = -1;
            }
        }

        return result.toString();
    }
```
功能分析：

1. 获取占位符前缀"${"的位置索引startIndex
2. 占位符前缀"{"存在，从"{"存在，从"{"后面开始获取占位符后缀"}"的位置索引endIndex
3. 如果占位符前缀位置索引startIndex与占位符后缀的位置索引endIndex都存在，截取中间的部分placeHolder
4. 从Properties中获取placeHolder对应的值propVal
5. 如果propVal不存在，尝试对placeHolder使用":"进行一次分割，如果分割出来有结果，那么前面一部分命名为actualPlaceholder，后面一部分命名为defaultValue，尝试从Properties中获取actualPlaceholder对应的value，如果存在则取此value，如果不存在则取defaultValue，最终赋值给propVal
6. 返回propVal，就是替换之后的值

通过这样一整个的流程，将占位符"${...}"中的内容替换为了我们需要的值。

