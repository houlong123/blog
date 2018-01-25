---
title: Bean加载流程之BeanWrapper构建
date: 2017-08-12 14:06:00
tags: Spring
---

另一篇 [Bean加载流程梳理之BeanDefinitions加载](https://houlong123.github.io/2017/08/10/Bean%E5%8A%A0%E8%BD%BD%E6%B5%81%E7%A8%8B%E6%A2%B3%E7%90%861/)，分析了Spring上下文加载的代码入口及BeanDefinitions加载流程的梳理。在AbstractApplicationContext的refresh方法中，首先获取了`ConfigurableListableBeanFactory`类型的beanFactory，然后对beanFactory进行了一系列的设置及其他资源的设置。一切准备就绪后，使用了`finishBeanFactoryInitialization`方法完成了对于所有非懒加载的Bean的初始化。

#### finishBeanFactoryInitialization源码
```java
protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {
        //设置beanFactory
        if(beanFactory.containsBean("conversionService") && beanFactory.isTypeMatch("conversionService", ConversionService.class)) {
            beanFactory.setConversionService((ConversionService)beanFactory.getBean("conversionService", ConversionService.class));
        }

        if(!beanFactory.hasEmbeddedValueResolver()) {
            beanFactory.addEmbeddedValueResolver(new StringValueResolver() {
                public String resolveStringValue(String strVal) {
                    return AbstractApplicationContext.this.getEnvironment().resolvePlaceholders(strVal);
                }
            });
        }

        String[] weaverAwareNames = beanFactory.getBeanNamesForType(LoadTimeWeaverAware.class, false, false);
        String[] var3 = weaverAwareNames;
        int var4 = weaverAwareNames.length;

        for(int var5 = 0; var5 < var4; ++var5) {
            String weaverAwareName = var3[var5];
            this.getBean(weaverAwareName);
        }

        beanFactory.setTempClassLoader((ClassLoader)null);
        beanFactory.freezeConfiguration();
        //调用了DefaultListableBeanFactory的preInstantiateSingletons方法
        beanFactory.preInstantiateSingletons();
    }
```
功能讲解：
 
 <!-- more -->
 
 finishBeanFactoryInitialization方法中调用了DefaultListableBeanFactory的`preInstantiateSingletons`方法，本文针对preInstantiateSingletons进行分析，解读一下Spring是如何初始化Bean实例对象出来的。
 
 

#### preInstantiateSingletons源码
```java
public void preInstantiateSingletons() throws BeansException {
        if(this.logger.isDebugEnabled()) {
            this.logger.debug("Pre-instantiating singletons in " + this);
        }

        //获取所有beanDefinition，并遍历
        ArrayList beanNames = new ArrayList(this.beanDefinitionNames);
        Iterator var2 = beanNames.iterator();

        while(true) {
            while(true) {
                String beanName;
                RootBeanDefinition singletonInstance;
                do {
                    do {
                        do {
                            if(!var2.hasNext()) {
                                var2 = beanNames.iterator();

                                while(var2.hasNext()) {
                                    beanName = (String)var2.next();
                                    Object singletonInstance1 = this.getSingleton(beanName);
                                    if(singletonInstance1 instanceof SmartInitializingSingleton) {
                                        final SmartInitializingSingleton smartSingleton1 = (SmartInitializingSingleton)singletonInstance1;
                                        if(System.getSecurityManager() != null) {
                                            AccessController.doPrivileged(new PrivilegedAction() {
                                                public Object run() {
                                                    smartSingleton1.afterSingletonsInstantiated();
                                                    return null;
                                                }
                                            }, this.getAccessControlContext());
                                        } else {
                                            smartSingleton1.afterSingletonsInstantiated();
                                        }
                                    }
                                }

                                return;
                            }

                            beanName = (String)var2.next();
                            //将非RootBeanDefinition转换为RootBeanDefinition以供后续操作。
                            //Bean定义公共的抽象类是AbstractBeanDefinition，普通的Bean在Spring加载Bean定义的时候，
                            //实例化出来的是GenericBeanDefinition，而Spring上下文包括实例化所有Bean用的AbstractBeanDefinition是RootBeanDefinition，
                            //这时候就使用getMergedLocalBeanDefinition方法做了一次转化
                            singletonInstance = this.getMergedLocalBeanDefinition(beanName);
                        } while(singletonInstance.isAbstract());
                    } while(!singletonInstance.isSingleton());
                } while(singletonInstance.isLazyInit());
                
                /**由于此方法实例化的是所有非懒加*载的单例Bean，
                *  因此要实例化Bean，必须满足11行的  三个定义：
                *（1）不是抽象的
                *（2）必须是单例的
                *（3）必须是非懒加载的
                */
                if(this.isFactoryBean(beanName)) {
                    final FactoryBean smartSingleton = (FactoryBean)this.getBean("&" + beanName);
                    boolean isEagerInit;
                   
                   //当bean是SmartFactoryBean，并且eagerInit=true时，立即实例化这个Bean
                   if(System.getSecurityManager() != null && smartSingleton instanceof SmartFactoryBean) {
                        isEagerInit = ((Boolean)AccessController.doPrivileged(new PrivilegedAction() {
                            public Boolean run() {
                                return Boolean.valueOf(((SmartFactoryBean)smartSingleton).isEagerInit());
                            }
                        }, this.getAccessControlContext())).booleanValue();
                    } else {
                        isEagerInit = smartSingleton instanceof SmartFactoryBean && ((SmartFactoryBean)smartSingleton).isEagerInit();
                    }

                    if(isEagerInit) {
                        this.getBean(beanName);
                    }
                } else {
                    this.getBean(beanName);
                }
            }
        }
    }
```


#### doGetBean源码
由源码可知，获取Bean对象实例，都是通过getBean方法，getBean方法最终调用的是DefaultListableBeanFactory的父类AbstractBeanFactory类的doGetBean方法，因此这部分重点分析一下doGetBean方法是如何构造出一个单例的Bean的。
```java
protected <T> T doGetBean(String name, Class<T> requiredType, final Object[] args, boolean typeCheckOnly) throws BeansException {
        final String beanName = this.transformedBeanName(name);
        /**
         * 首先检查一下本地的单例缓存是否已经加载过Bean，
         * 没有的话再检查earlySingleton缓存是否已经加载过Bean，
         * 有的话执行后面的逻辑。
         */
        Object sharedInstance = this.getSingleton(beanName);
        Object bean;
        if(sharedInstance != null && args == null) {
            if(this.logger.isDebugEnabled()) {
                if(this.isSingletonCurrentlyInCreation(beanName)) {
                    this.logger.debug("Returning eagerly cached instance of singleton bean \'" + beanName + "\' that is not fully initialized yet - a consequence of a circular reference");
                } else {
                    this.logger.debug("Returning cached instance of singleton bean \'" + beanName + "\'");
                }
            }

            bean = this.getObjectForBeanInstance(sharedInstance, name, beanName, (RootBeanDefinition)null);
        } else {
            if(this.isPrototypeCurrentlyInCreation(beanName)) {
                throw new BeanCurrentlyInCreationException(beanName);
            }

            BeanFactory ex = this.getParentBeanFactory();
            if(ex != null && !this.containsBeanDefinition(beanName)) {
                String var24 = this.originalBeanName(name);
                if(args != null) {
                    return ex.getBean(var24, args);
                }

                return ex.getBean(var24, requiredType);
            }

            if(!typeCheckOnly) {
                this.markBeanAsCreated(beanName);
            }

            try {
                final RootBeanDefinition ex1 = this.getMergedLocalBeanDefinition(beanName);
                this.checkMergedBeanDefinition(ex1, beanName, args);
                //处理bean的依赖类。depends-on依赖的Bean会优先于当前Bean被加载
                String[] dependsOn = ex1.getDependsOn();
                String[] scopeName;
                if(dependsOn != null) {
                    scopeName = dependsOn;
                    int scope = dependsOn.length;

                    for(int ex2 = 0; ex2 < scope; ++ex2) {
                        String dep = scopeName[ex2];
                        if(this.isDependent(beanName, dep)) {
                            throw new BeanCreationException(ex1.getResourceDescription(), beanName, "Circular depends-on relationship between \'" + beanName + "\' and \'" + dep + "\'");
                        }

                        this.registerDependentBean(dep, beanName);
                        this.getBean(dep);
                    }
                }

                if(ex1.isSingleton()) {
                    sharedInstance = this.getSingleton(beanName, new ObjectFactory() {
                        public Object getObject() throws BeansException {
                            try {
                                return AbstractBeanFactory.this.createBean(beanName, ex1, args);
                            } catch (BeansException var2) {
                                AbstractBeanFactory.this.destroySingleton(beanName);
                                throw var2;
                            }
                        }
                    });
                    bean = this.getObjectForBeanInstance(sharedInstance, name, beanName, ex1);
                } else if(ex1.isPrototype()) {
                    scopeName = null;

                    Object var25;
                    try {
                        this.beforePrototypeCreation(beanName);
                        var25 = this.createBean(beanName, ex1, args);
                    } finally {
                        this.afterPrototypeCreation(beanName);
                    }

                    bean = this.getObjectForBeanInstance(var25, name, beanName, ex1);
                } else {
                    String var26 = ex1.getScope();
                    Scope var27 = (Scope)this.scopes.get(var26);
                    if(var27 == null) {
                        throw new IllegalStateException("No Scope registered for scope name \'" + var26 + "\'");
                    }

                    try {
                        Object var28 = var27.get(beanName, new ObjectFactory() {
                            public Object getObject() throws BeansException {
                                AbstractBeanFactory.this.beforePrototypeCreation(beanName);

                                Object var1;
                                try {
                                    //调用AbstractBeanFactory类的createBean方法获取bean
                                    var1 = AbstractBeanFactory.this.createBean(beanName, ex1, args);
                                } finally {
                                    AbstractBeanFactory.this.afterPrototypeCreation(beanName);
                                }

                                return var1;
                            }
                        });
                        bean = this.getObjectForBeanInstance(var28, name, beanName, ex1);
                    } catch (IllegalStateException var21) {
                        throw new BeanCreationException(beanName, "Scope \'" + var26 + "\' is not active for the current thread; consider defining a scoped proxy for this bean if you intend to refer to it from a singleton", var21);
                    }
                }
            } catch (BeansException var23) {
                this.cleanupAfterBeanCreationFailure(beanName);
                throw var23;
            }
        }

        if(requiredType != null && bean != null && !requiredType.isAssignableFrom(bean.getClass())) {
            try {
                return this.getTypeConverter().convertIfNecessary(bean, requiredType);
            } catch (TypeMismatchException var22) {
                if(this.logger.isDebugEnabled()) {
                    this.logger.debug("Failed to convert bean \'" + name + "\' to required type \'" + ClassUtils.getQualifiedName(requiredType) + "\'", var22);
                }

                throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
            }
        } else {
            return bean;
        }
    }
```
功能分析：

上述代码是获取`单例bean`的处理流程，程序会先从本地的单例缓存中获取，如果没有，再去加载单例bean的时候，如果该bean依赖其他bean，则先加载所依赖的其他bean，最后再去调用`AbstractAutowireCapableBeanFactory`类中的createBean方法。


#### doCreateBean源码
从`AbstractAutowireCapableBeanFactory`类中的createBean方法，追溯到`doCreateBean`方法，该方法为bean加载的主流程。
```java
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, Object[] args) throws BeanCreationException {
        //初始化bean
        BeanWrapper instanceWrapper = null;
        if(mbd.isSingleton()) {
            instanceWrapper = (BeanWrapper)this.factoryBeanInstanceCache.remove(beanName);
        }

        if(instanceWrapper == null) {
            //创建出Bean的实例，并包装为BeanWrapper
            instanceWrapper = this.createBeanInstance(beanName, mbd, args);
        }

        final Object bean = instanceWrapper != null?instanceWrapper.getWrappedInstance():null;
        Class beanType = instanceWrapper != null?instanceWrapper.getWrappedClass():null;
        mbd.resolvedTargetType = beanType;
        Object earlySingletonExposure = mbd.postProcessingLock;
        //Allow post-processors to modify the merged bean definition.
        synchronized(mbd.postProcessingLock) {
            if(!mbd.postProcessed) {
                try {
                    this.applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
                } catch (Throwable var17) {
                    throw new BeanCreationException(mbd.getResourceDescription(), beanName, "Post-processing of merged bean definition failed", var17);
                }

                mbd.postProcessed = true;
            }
        }

        boolean var20 = mbd.isSingleton() && this.allowCircularReferences && this.isSingletonCurrentlyInCreation(beanName);
        if(var20) {
            if(this.logger.isDebugEnabled()) {
                this.logger.debug("Eagerly caching bean \'" + beanName + "\' to allow for resolving potential circular references");
            }

            this.addSingletonFactory(beanName, new ObjectFactory() {
                public Object getObject() throws BeansException {
                    return AbstractAutowireCapableBeanFactory.this.getEarlyBeanReference(beanName, mbd, bean);
                }
            });
        }
        //Initialize the bean instance.
        Object exposedObject = bean;

        try {
            this.populateBean(beanName, mbd, instanceWrapper);
            if(exposedObject != null) {
                //此语句执行，在bean的实例化过程中对bean做一些包装处理。（即：BeanPostProcessor接口）
                exposedObject = this.initializeBean(beanName, exposedObject, mbd);
            }
        } catch (Throwable var18) {
            if(var18 instanceof BeanCreationException && beanName.equals(((BeanCreationException)var18).getBeanName())) {
                throw (BeanCreationException)var18;
            }

            throw new BeanCreationException(mbd.getResourceDescription(), beanName, "Initialization of bean failed", var18);
        }

        if(var20) {
            Object ex = this.getSingleton(beanName, false);
            if(ex != null) {
                if(exposedObject == bean) {
                    exposedObject = ex;
                } else if(!this.allowRawInjectionDespiteWrapping && this.hasDependentBean(beanName)) {
                    String[] dependentBeans = this.getDependentBeans(beanName);
                    LinkedHashSet actualDependentBeans = new LinkedHashSet(dependentBeans.length);
                    String[] var12 = dependentBeans;
                    int var13 = dependentBeans.length;

                    for(int var14 = 0; var14 < var13; ++var14) {
                        String dependentBean = var12[var14];
                        if(!this.removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
                            actualDependentBeans.add(dependentBean);
                        }
                    }

                    if(!actualDependentBeans.isEmpty()) {
                        throw new BeanCurrentlyInCreationException(beanName, "Bean with name \'" + beanName + "\' has been injected into other beans [" + StringUtils.collectionToCommaDelimitedString(actualDependentBeans) + "] in its raw version as part of a circular reference, but has eventually been wrapped. This means that said other beans do not use the final version of the bean. This is often the result of over-eager type matching - consider using \'getBeanNamesOfType\' with the \'allowEagerInit\' flag turned off, for example.");
                    }
                }
            }
        }

        try {
            this.registerDisposableBeanIfNecessary(beanName, bean, mbd);
            return exposedObject;
        } catch (BeanDefinitionValidationException var16) {
            throw new BeanCreationException(mbd.getResourceDescription(), beanName, "Invalid destruction signature", var16);
        }
    }
```
功能分析：
该方法先获取BeanWrapper对象，然后在加载bean之前，然后在bean的实例化过程中对bean做一些包装处理，即执行`BeanPostProcessor`接口子类相关逻辑和若bean继承了InitializingBean，则执行afterPropertiesSet()方法。

![Bean实例化过程](http://thumbsnap.com/i/c5DXvHI7.jpg)

#### createBeanInstance部分关键源码

```java
protected BeanWrapper createBeanInstance(String beanName, RootBeanDefinition mbd, Object[] args) {
            if(resolved) {
                return autowireNecessary?this.autowireConstructor(beanName, mbd, (Constructor[])null, (Object[])null):this.instantiateBean(beanName, mbd);
            } else {
                Constructor[] ctors1 = this.determineConstructorsFromBeanPostProcessors(beanClass, beanName);
                //bean标签使用构造函数注入属性的话，执行autowireConstructor方法；否则使用默认构造函数，使用setter注入属性，执行instantiateBean方法
                return ctors1 == null && mbd.getResolvedAutowireMode() != 3 && !mbd.hasConstructorArgumentValues() && ObjectUtils.isEmpty(args)?this.instantiateBean(beanName, mbd):this.autowireConstructor(beanName, mbd, ctors1, args);
            }
        }
    }
```

#### instantiateBean源码
默认构造函数，使用setter注入属性，执行此段代码。
```java
protected BeanWrapper instantiateBean(final String beanName, final RootBeanDefinition mbd) {
        try {
            Object ex;
            if(System.getSecurityManager() != null) {
                ex = AccessController.doPrivileged(new PrivilegedAction() {
                    public Object run() {
                        return AbstractAutowireCapableBeanFactory.this.getInstantiationStrategy().instantiate(mbd, beanName, AbstractAutowireCapableBeanFactory.this);
                    }
                }, this.getAccessControlContext());
            } else {
                ex = this.getInstantiationStrategy().instantiate(mbd, beanName, this);
            }

            BeanWrapperImpl bw = new BeanWrapperImpl(ex);
            this.initBeanWrapper(bw);
            return bw;
        } catch (Throwable var6) {
            throw new BeanCreationException(mbd.getResourceDescription(), beanName, "Instantiation of bean failed", var6);
        }
    }

```


```java
public Object instantiate(RootBeanDefinition bd, String beanName, BeanFactory owner) {
        if(bd.getMethodOverrides().isEmpty()) {
            Object var5 = bd.constructorArgumentLock;
            Constructor constructorToUse;
            synchronized(bd.constructorArgumentLock) {
                constructorToUse = (Constructor)bd.resolvedConstructorOrFactoryMethod;
                if(constructorToUse == null) {
                    final Class clazz = bd.getBeanClass();
                    //接口实例化时，报错
                    if(clazz.isInterface()) {
                        throw new BeanInstantiationException(clazz, "Specified class is an interface");
                    }

                    try {
                        //获取创建bean的构造函数
                        if(System.getSecurityManager() != null) {
                            constructorToUse = (Constructor)AccessController.doPrivileged(new PrivilegedExceptionAction() {
                                public Constructor<?> run() throws Exception {
                                    return clazz.getDeclaredConstructor((Class[])null);
                                }
                            });
                        } else {
                            constructorToUse = clazz.getDeclaredConstructor((Class[])null);
                        }

                        bd.resolvedConstructorOrFactoryMethod = constructorToUse;
                    } catch (Throwable var9) {
                        throw new BeanInstantiationException(clazz, "No default constructor found", var9);
                    }
                }
            }
            //获取bean实例
            return BeanUtils.instantiateClass(constructorToUse, new Object[0]);
        } else {
            return this.instantiateWithMethodInjection(bd, beanName, owner);
        }
    }

```


```java
public static <T> T instantiateClass(Constructor<T> ctor, Object... args) throws BeanInstantiationException {
        Assert.notNull(ctor, "Constructor must not be null");

        try {
            //通过反射技术获取bean实例
            ReflectionUtils.makeAccessible(ctor);
            return ctor.newInstance(args);
        } catch (InstantiationException var3) {
            throw new BeanInstantiationException(ctor, "Is it an abstract class?", var3);
        } catch (IllegalAccessException var4) {
            throw new BeanInstantiationException(ctor, "Is the constructor accessible?", var4);
        } catch (IllegalArgumentException var5) {
            throw new BeanInstantiationException(ctor, "Illegal arguments for constructor", var5);
        } catch (InvocationTargetException var6) {
            throw new BeanInstantiationException(ctor, "Constructor threw exception", var6.getTargetException());
        }
    }
```
功能分析：

通过反射生成Bean的实例。`看到前面有一步makeAccessible，这意味着即使Bean的构造函数是private、protected的，依然不影响Bean的构造`。

最后注意一下，这里被实例化出来的Bean并不会直接返回，而是会被包装为BeanWrapper继续在后面使用。




### 参考文献
[【Spring源码分析】非懒加载的单例Bean初始化过程（上篇](http://www.cnblogs.com/xrq730/p/6361578.html)

[ BeanPostProcessor的使用](http://www.cnblogs.com/jyyzzjl/p/5417418.html)
