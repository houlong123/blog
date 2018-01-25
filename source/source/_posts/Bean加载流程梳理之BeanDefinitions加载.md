---
title: Bean加载流程梳理之BeanDefinitions加载
date: 2017-08-10 13:59:35
tags: Spring
---

<i><font color="red">BeanDefinitions加载是发生在容器获取BeanFactory时，具体是在刷新BeanFactory后进行BeanDefinitions的加载。BeanDefinitions成功加载后，可供后续获取bean对象做准备。</font></i>

#### Bean加载源码入口
Spring的重要特征之一是IOC(Inversion of Control)，即：控制反转。IOC技术促进了松耦合。Spring提供了两种IOC容器类型：BeanFactory和ApplicationContext，在类结构上，ApplicationContext是继承自BeanFactory的。下面讲解ApplicationContext容器。
下面有很简单的一段代码可以作为Spring中Bean加载的入口：

```java
ApplicationContext context = new ClassPathXmlApplicationContext("spring.xml");
context.getBean("beanName");
```
ClassPathXmlApplicationContext用于加载CLASSPATH下的Spring配置文件。由示例可知，`context.getBean("beanName");`即可获取到Bean的实例，那么必然`ApplicationContext context = new ClassPathXmlApplicationContext("spring.xml");`就已经完成了对所有Bean实例的加载，因此可以通过ClassPathXmlApplicationContext作为Bean加载源码入口。

类图：

<!-- more -->

![ClassPathXmlApplicationContext类继承图](http://thumbsnap.com/i/6lhuqDDO.jpg)


#### 源码解析


```Java
public ClassPathXmlApplicationContext(String configLocation) throws BeansException {
        this(new String[]{configLocation}, true, (ApplicationContext)null);
}
```


```java
public ClassPathXmlApplicationContext(String[] configLocations, boolean refresh, ApplicationContext parent) throws BeansException {
        super(parent);
        this.setConfigLocations(configLocations);
        if(refresh) {
            this.refresh();
        }
}
```
第二个构造函数完成的功能：
+ super(parent)
    
    设置一下父级ApplicationContext，这里是null

+ setConfigLocations(configLocations)
    
    里面做了两件事情：
    + 将指定的Spring配置文件的路径存储到本地
    + 解析Spring配置文件路径中的${PlaceHolder}占位符，替换为系统变量中PlaceHolder对应的Value值，System本身就自带一些系统变量比如class.path、os.name、user.dir等，也可以通过System.setProperty()方法设置自己需要的系统变量

+ refresh()
    
    这个就是整个Spring Bean加载的核心，它是ClassPathXmlApplicationContext的父类AbstractApplicationContext的一个方法，用于刷新整个Spring上下文信息，定义了整个Spring上下文加载的流程。

#### refresh()源码


```java
public void refresh() throws BeansException, IllegalStateException {
        Object var1 = this.startupShutdownMonitor;
        synchronized(this.startupShutdownMonitor) {
            //Prepare this context for refreshing
            this.prepareRefresh();
            
            //Tell the subclass to refresh the internal bean factory
            ConfigurableListableBeanFactory beanFactory = this.obtainFreshBeanFactory();
            
            //Prepare the bean factory for use in this context.
            this.prepareBeanFactory(beanFactory);

            try {
                //Allows post-processing of the bean factory in context subclasses.
                this.postProcessBeanFactory(beanFactory);
               
                //Invoke factory processors registered as beans in the context
                this.invokeBeanFactoryPostProcessors(beanFactory);
               
                //Register bean processors that intercept bean creatio
                this.registerBeanPostProcessors(beanFactory);
                
                //Initialize message source for this context
                this.initMessageSource();
               
                //Initialize event multicaster for this context
                this.initApplicationEventMulticaster();
                
                //Initialize other special beans in specific context subclasses
                this.onRefresh();
                
                //check for listener beans and register them
                this.registerListeners();
               
                //Instantiate all remaining (non-lazy-init) singletons
                this.finishBeanFactoryInitialization(beanFactory);
                
                //Last step: publish corresponding event
                this.finishRefresh();
            } catch (BeansException var9) {
                if(this.logger.isWarnEnabled()) {
                    this.logger.warn("Exception encountered during context initialization - cancelling refresh attempt: " + var9);
                }
                //Destroy already created singletons to avoid dangling resources
                this.destroyBeans();
                //Reset 'active' flag
                this.cancelRefresh(var9);
                throw var9;
            } finally {
                this.resetCommonCaches();
            }

    }
}
```
此方法使用了设计模式中的[模板设计模式](http://blog.csdn.net/zhengzhb/article/details/7405608)和[策略设计模式](http://blog.csdn.net/zhengzhb/article/details/7609670)。模板方法中的个别基本方法由相应的子类去具体实现，易于扩展，达到了代码的复用效果。

<i><b><font color=red size=2 face="黑体">注：refresh()方法和close()方法都使用了startUpShutdownMonitor对象锁加锁，这就保证了在调用refresh()方法的时候无法调用close()方法，反之亦然，避免了冲突</font></b></i>

#### prepareRefresh源码

```java
//Prepare this context for refreshing, setting its startup date and active flag
protected void prepareRefresh() {
        this.startupDate = System.currentTimeMillis();
        this.closed.set(false);
        this.active.set(true);
        if(this.logger.isInfoEnabled()) {
            this.logger.info("Refreshing " + this);
        }

        this.initPropertySources();
        this.getEnvironment().validateRequiredProperties();
        this.earlyApplicationEvents = new LinkedHashSet();
}
```
功能：
+ 设置一下刷新Spring上下文的开始时间
+ 将active标识位设置为true，closed设置为false
+ 初始化资源配置并对必要资源进行校验

#### obtainFreshBeanFactory源码

```java
protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
        this.refreshBeanFactory();
        ConfigurableListableBeanFactory beanFactory = this.getBeanFactory();
        if(this.logger.isDebugEnabled()) {
            this.logger.debug("Bean factory for " + this.getDisplayName() + ": " + beanFactory);
        }
        return beanFactory;
}
```
功能：
+ 刷新BeanFactory
+ 获取类型为ConfigurableListableBeanFactory的beanFactory实例并返回

#### refreshBeanFactory源码
```java
protected final void refreshBeanFactory() throws BeansException {
        //beanFactory不为空时，销毁所有bean，并将beanFactory置空
        if(this.hasBeanFactory()) {
            this.destroyBeans();
            this.closeBeanFactory();
        }

        try {
            //创建DefaultListableBeanFactory类型的bean
            DefaultListableBeanFactory ex = this.createBeanFactory();
            ex.setSerializationId(this.getId());
            //设置BeanFactory
            this.customizeBeanFactory(ex);
            //加载bean的定义
            this.loadBeanDefinitions(ex);
            Object var2 = this.beanFactoryMonitor;
            synchronized(this.beanFactoryMonitor) {
                this.beanFactory = ex;
            }
        } catch (IOException var5) {
            throw new ApplicationContextException("I/O error parsing bean definition source for " + this.getDisplayName(), var5);
        }
    }
```
在上述源码中`DefaultListableBeanFactory ex = this.createBeanFactory();`是代码核心，`DefaultListableBeanFactory`类是构造bean的核心类，关系继承图如下：

类图：
![DefaultListableBeanFactory类继承图](http://thumbsnap.com/i/rg8ryXwX.jpg)

#### loadBeanDefinitions源码

```java
//由类继承关系可知，相应源码在AbstractXmlApplicationContext类
protected void loadBeanDefinitions(DefaultListableBeanFactory beanFactory) throws BeansException, IOException {
        //获取XmlBeanDefinitionReader对象
        XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);
        
        //XmlBeanDefinitionReader对象相关设置
        beanDefinitionReader.setEnvironment(this.getEnvironment());
        
        //由类图可知，AbstractXmlApplicationContext类间接继承了ResourceLoader接口，因此类本身也是一种resourceLoader
        beanDefinitionReader.setResourceLoader(this);
        beanDefinitionReader.setEntityResolver(new ResourceEntityResolver(this));
        this.initBeanDefinitionReader(beanDefinitionReader);
        
        //bean定义加载，此处为bean加载核心
        this.loadBeanDefinitions(beanDefinitionReader);
    }
```

#### loadBeanDefinitions源码
```java
protected void loadBeanDefinitions(XmlBeanDefinitionReader reader) throws BeansException, IOException {
        Resource[] configResources = this.getConfigResources();
        if(configResources != null) {
            reader.loadBeanDefinitions(configResources);
        }

        String[] configLocations = this.getConfigLocations();
        if(configLocations != null) {
            reader.loadBeanDefinitions(configLocations);
        }

    }
```
此方法的功能主要就是获取配置资源文件，如果资源不为空，则使`XmlBeanDefinitionReader`用对象去加载资源文件。


#### loadBeanDefinitions 源码
```java
//经过一系列的处理，程序会执行到loadBeanDefinitions()相应的重载方法。
public int loadBeanDefinitions(EncodedResource encodedResource) throws BeanDefinitionStoreException {
        Assert.notNull(encodedResource, "EncodedResource must not be null");
        if(this.logger.isInfoEnabled()) {
            this.logger.info("Loading XML bean definitions from " + encodedResource.getResource());
        }

        Object currentResources = (Set)this.resourcesCurrentlyBeingLoaded.get();
        if(currentResources == null) {
            currentResources = new HashSet(4);
            this.resourcesCurrentlyBeingLoaded.set(currentResources);
        }

        if(!((Set)currentResources).add(encodedResource)) {
            throw new BeanDefinitionStoreException("Detected cyclic loading of " + encodedResource + " - check your import definitions!");
        } else {
            int var5;
            try {
                //将bean配置资源文件转换为流
                InputStream ex = encodedResource.getResource().getInputStream();

                try {
                    InputSource inputSource = new InputSource(ex);
                    if(encodedResource.getEncoding() != null) {
                        inputSource.setEncoding(encodedResource.getEncoding());
                    }
            
                    //加载bean配置文件
                    var5 = this.doLoadBeanDefinitions(inputSource, encodedResource.getResource());
                } finally {
                    ex.close();
                }
            } catch (IOException var15) {
                throw new BeanDefinitionStoreException("IOException parsing XML document from " + encodedResource.getResource(), var15);
            } finally {
                ((Set)currentResources).remove(encodedResource);
                if(((Set)currentResources).isEmpty()) {
                    this.resourcesCurrentlyBeingLoaded.remove();
                }

            }

            return var5;
        }
    }
```

#### doRegisterBeanDefinitions源码
```java
//经过一系列处理，程序执行到doRegisterBeanDefinitions()方法
protected void doRegisterBeanDefinitions(Element root) {
        BeanDefinitionParserDelegate parent = this.delegate;
        this.delegate = this.createDelegate(this.getReaderContext(), root, parent);
        if(this.delegate.isDefaultNamespace(root)) {
            String profileSpec = root.getAttribute("profile");
            if(StringUtils.hasText(profileSpec)) {
                String[] specifiedProfiles = StringUtils.tokenizeToStringArray(profileSpec, ",; ");
                if(!this.getReaderContext().getEnvironment().acceptsProfiles(specifiedProfiles)) {
                    if(this.logger.isInfoEnabled()) {
                        this.logger.info("Skipped XML bean definition file due to specified profiles [" + profileSpec + "] not matching: " + this.getReaderContext().getResource());
                    }

                    return;
                }
            }
        }

        this.preProcessXml(root);
        //解析xml配置文件，加载bean定义的核心代码
        this.parseBeanDefinitions(root, this.delegate);
        this.postProcessXml(root);
        this.delegate = parent;
    }
```

#### parseBeanDefinitions源码
```java
protected void parseBeanDefinitions(Element root, BeanDefinitionParserDelegate delegate) {
        if(delegate.isDefaultNamespace(root)) {
            NodeList nl = root.getChildNodes();

            for(int i = 0; i < nl.getLength(); ++i) {
                Node node = nl.item(i);
                if(node instanceof Element) {
                    Element ele = (Element)node;
                    if(delegate.isDefaultNamespace(ele)) {  //如果是默认的命名空间，则执行，查看源码可知，默认命名空间为"http://www.springframework.org/schema/beans"
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


```java
//如果为非默认命名空间时，进行相应解析
public BeanDefinition parseCustomElement(Element ele, BeanDefinition containingBd) {
        String namespaceUri = this.getNamespaceURI(ele);
        NamespaceHandler handler = this.readerContext.getNamespaceHandlerResolver().resolve(namespaceUri);
        if(handler == null) {
            this.error("Unable to locate Spring NamespaceHandler for XML schema namespace [" + namespaceUri + "]", ele);
            return null;
        } else {
            return handler.parse(ele, new ParserContext(this.readerContext, this, containingBd));
        }
    }
```


```java
//如果为默认命名空间时，则进行相应的文件解析
private void parseDefaultElement(Element ele, BeanDefinitionParserDelegate delegate) {
        if(delegate.nodeNameEquals(ele, "import")) {
            this.importBeanDefinitionResource(ele);
        } else if(delegate.nodeNameEquals(ele, "alias")) {
            this.processAliasRegistration(ele);
        } else if(delegate.nodeNameEquals(ele, "bean")) {
            this.processBeanDefinition(ele, delegate);
        } else if(delegate.nodeNameEquals(ele, "beans")) {
            this.doRegisterBeanDefinitions(ele);
        }

    }
```
由源码可知，当为默认命名空间时，将解析xml配置文件中的`<import>, <alias>, <bean>, <beans>`元素。



xml配置文件范例:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:aop="http://www.springframework.org/schema/aop"
       xmlns:mvc="http://www.springframework.org/schema/mvc" xmlns:context="http://www.springframework.org/schema/context"
       xmlns:task="http://www.springframework.org/schema/task"
       xsi:schemaLocation="
     http://www.springframework.org/schema/task http://www.springframework.org/schema/task/spring-task.xsd
     http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context.xsd
     http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
     http://www.springframework.org/schema/aop http://www.springframework.org/schema/aop/spring-aop.xsd
     http://www.springframework.org/schema/mvc http://www.springframework.org/schema/mvc/spring-mvc.xsd
     ">

    <mvc:annotation-driven/>
    <aop:aspectj-autoproxy/>

    <task:executor id="taskExecutor" pool-size="5-500" queue-capacity="1000"/>
    <task:scheduler id="taskScheduler" pool-size="1000"/>
    <task:annotation-driven proxy-target-class="true" executor="taskExecutor" scheduler="taskScheduler"/>

    <context:component-scan base-package="com.xxx.xxx">
        <context:exclude-filter type="regex" expression="com.xxx.xxx.controller"/>
    </context:component-scan>
    
    <import resource="classpath:spring-configs/applicationContext-xxx.xml"/>
    <bean id="mongoTemplate" class="xxx" />
</beans>
```