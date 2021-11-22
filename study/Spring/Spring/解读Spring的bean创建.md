# Table of Contents

* [解读Spring的bean创建](#解读spring的bean创建)
    * [分析点](#分析点)
    * [1. bean的创建时机](#1-bean的创建时机)
    * [2. 针对原型和单例bean创建的区分](#2-针对原型和单例bean创建的区分)
    * [3. bean的生命周期分析](#3-bean的生命周期分析)
    * [4. bean的实际创建分析](#4-bean的实际创建分析)
        * [4.1. 分析-bean对象实例化](#41-分析-bean对象实例化)
        * [4.2. 分析-bean对象初始化](#42-分析-bean对象初始化)
    * [5. bean的销毁分析](#5-bean的销毁分析)


# 解读Spring的bean创建


## 分析点

1. bean的创建时机
2. 针对原型和单例bean创建的区分
3. bean的生命周期分析
4. bean的实际创建分析
5. bean的销毁分析

## 1. bean的创建时机

1. IoC容器启动时，会针对所有非懒加载的单例bean进行一次创建
2. 懒加载的单例bean将在使用时进行创建
3. 原型bean则是每个使用的地方都进行创建


## 2. 针对原型和单例bean创建的区分

区分的源码在 `AbstractBeanFactory#doGetBean` :

```java
if (mbd.isSingleton()) {
        sharedInstance = getSingleton(beanName, () -> {
        try {
        return createBean(beanName, mbd, args);
        }
        catch (BeansException ex) {
        // Explicitly remove instance from singleton cache: It might have been put there
        // eagerly by the creation process, to allow for circular reference resolution.
        // Also remove any beans that received a temporary reference to the bean.
        destroySingleton(beanName);
        throw ex;
        }
        });
        bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
        }

        else if (mbd.isPrototype()) {
        // It's a prototype -> create a new instance.
        Object prototypeInstance = null;
        try {
        beforePrototypeCreation(beanName);
        prototypeInstance = createBean(beanName, mbd, args);
        }
        finally {
        afterPrototypeCreation(beanName);
        }
        bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
        }
```

提供有两个方法 `isSingleton` 和 `isPrototype` 去区分单例还是原型。

原型很好理解，调用`createBean`拿到对象。单例的则又过了一个方法`getSingleton`，该方法中会先判断单例bean缓存中有没有该bean，有就执行创建bean的逻辑，创建完后再将创建的bean塞入单例bean缓存中。


## 3. bean的生命周期分析

在通过源码分析前，我们概念性的看看bean的生命周期是啥样的，再带着这个概念去分析源码bean到底怎么创建。

![](http://img.yelizi.top/c5a4f8be-2029-4677-987c-25b21b74e64c.jpg$xyz)

1. 实例化bean对象并包装在`BeanWrapper`中；
2. 设置对象属性（依赖注入）；
3. 若bean实现了xxAware接口，将相关xxAware实例注入bean；
4. 所有注册的`BeanPostProcessor`的before方法会挨个执行；
5. 若bean实现了`InitializingBean`接口，执行`afterPropertiesSet`方法；
6. 若配置了自定义的init-method，执行；
7. 所有注册的`BeanPostProcessor`的after方法会挨个执行；
8. 注册相关的销毁回调（为下两步做准备的）
9. 若bean实现了`DisposableBean`接口，执行`destroy`方法；
10. 若配置了自定义的destroy-method，执行；

这里我定义另一个bean，实现了 `InitializingBean`, `BeanNameAware`, `DisposableBean`，并且设置了自定义的初始化方法和自定义的销毁方法，顺带还自定义了一个`BeanPostProcessor`，我们启动程序看看日志打印：

![](http://img.yelizi.top/c73bd7a1-1774-4bd6-b791-0dbb59ffbe43.jpg$xyz)

![](http://img.yelizi.top/21d8f3ea-2259-47c3-893d-90cf2d9dee3a.jpg$xyz)


## 4. bean的实际创建分析

这里我们便扒开源码好好分析下上面的生命周期，首先定位到`AbstractAutowireCapableBeanFactory#doCreateBean`：


```java
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final @Nullable Object[] args)
        throws BeanCreationException {

        // 该步骤就根据beanName解析出是哪个bean类并实例化包装进BeanWrapper createBeanInstance见 4.1.  分析-bean对象实例化
        BeanWrapper instanceWrapper = null;
        if (mbd.isSingleton()) {
        instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
        }
        if (instanceWrapper == null) {
        instanceWrapper = createBeanInstance(beanName, mbd, args);
        }
final Object bean = instanceWrapper.getWrappedInstance();
        Class<?> beanType = instanceWrapper.getWrappedClass();
        if (beanType != NullBean.class) {
        mbd.resolvedTargetType = beanType;
        }

// Allow post-processors to modify the merged bean definition.
synchronized (mbd.postProcessingLock) {
        if (!mbd.postProcessed) {
        try {
        applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
        }
        catch (Throwable ex) {
        throw new BeanCreationException(mbd.getResourceDescription(), beanName,
        "Post-processing of merged bean definition failed", ex);
        }
        mbd.postProcessed = true;
        }
        }

        // Eagerly cache singletons to be able to resolve circular references
        // even when triggered by lifecycle interfaces like BeanFactoryAware.
        boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
        isSingletonCurrentlyInCreation(beanName));
        if (earlySingletonExposure) {
        if (logger.isTraceEnabled()) {
        logger.trace("Eagerly caching bean '" + beanName +
        "' to allow for resolving potential circular references");
        }
        addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
        }

        // 第二个重点处 这里开始对刚刚生成的bean进行初始化工作
        Object exposedObject = bean;
        try {
        // 该步骤进行了一些准备的bean填充工作，包括实现了InstantiationAwareBeanPostProcessor（BeanPostProcessor的子类）的BeanPostProcessor执行
        populateBean(beanName, mbd, instanceWrapper);
        // 真正初始化bean 详细见 4.2.  分析-bean对象初始化
        exposedObject = initializeBean(beanName, exposedObject, mbd);
        }
        catch (Throwable ex) {
        if (ex instanceof BeanCreationException && beanName.equals(((BeanCreationException) ex).getBeanName())) {
        throw (BeanCreationException) ex;
        }
        else {
        throw new BeanCreationException(
        mbd.getResourceDescription(), beanName, "Initialization of bean failed", ex);
        }
        }

        if (earlySingletonExposure) {
        Object earlySingletonReference = getSingleton(beanName, false);
        if (earlySingletonReference != null) {
        if (exposedObject == bean) {
        exposedObject = earlySingletonReference;
        }
        else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {
        String[] dependentBeans = getDependentBeans(beanName);
        Set<String> actualDependentBeans = new LinkedHashSet<>(dependentBeans.length);
        for (String dependentBean : dependentBeans) {
        if (!removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
        actualDependentBeans.add(dependentBean);
        }
        }
        if (!actualDependentBeans.isEmpty()) {
        throw new BeanCurrentlyInCreationException(beanName,
        "Bean with name '" + beanName + "' has been injected into other beans [" +
        StringUtils.collectionToCommaDelimitedString(actualDependentBeans) +
        "] in its raw version as part of a circular reference, but has eventually been " +
        "wrapped. This means that said other beans do not use the final version of the " +
        "bean. This is often the result of over-eager type matching - consider using " +
        "'getBeanNamesOfType' with the 'allowEagerInit' flag turned off, for example.");
        }
        }
        }
        }

        // 该步骤就是注册销毁的回调
        try {
        registerDisposableBeanIfNecessary(beanName, bean, mbd);
        }
        catch (BeanDefinitionValidationException ex) {
        throw new BeanCreationException(
        mbd.getResourceDescription(), beanName, "Invalid destruction signature", ex);
        }

        return exposedObject;
        }
```


### 4.1. 分析-bean对象实例化

以自定义的一个`TestBean`为例（通过`@Bean`方式注册），一步步单步到`SimpleInstantiationStrategy#instantiate`:


![](http://img.yelizi.top/068ddca3-6001-4f78-808c-c3c249466557.jpg$xyz)

很显然就是通过反射技术，然后就把这个实例化的bean塞入`BeanWrapper`，由`BeanWrapper`带着继续继续更多的封装。

该步骤执行完，我们看日志：

![](http://img.yelizi.top/8825cacd-4492-4000-bd5d-5040204cb817.jpg$xyz)


### 4.2. 分析-bean对象初始化

主要跟踪`AbstractAutowireCapableBeanFactory#initializeBean`:

```java
protected Object initializeBean(final String beanName, final Object bean, @Nullable RootBeanDefinition mbd) {
        // 执行Aware的方法 BeanNameAware、BeanClassLoaderAware、BeanFactoryAware
        if (System.getSecurityManager() != null) {
        AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
        invokeAwareMethods(beanName, bean);
        return null;
        }, getAccessControlContext());
        }
        else {
        invokeAwareMethods(beanName, bean);
        }

        // 执行BeanPostProcessor的前置处理
        Object wrappedBean = bean;
        if (mbd == null || !mbd.isSynthetic()) {
        wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
        }

        // 执行初始化方法
        try {
        invokeInitMethods(beanName, wrappedBean, mbd);
        }
        catch (Throwable ex) {
        throw new BeanCreationException(
        (mbd != null ? mbd.getResourceDescription() : null),
        beanName, "Invocation of init method failed", ex);
        }
        // 执行BeanPostProcessor的后置处理
        if (mbd == null || !mbd.isSynthetic()) {
        wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
        }

        return wrappedBean;
        }
```

具体每个步骤怎么实现就不细讲了，逻辑都很顺，直接看源码就行。有一个补充点（主要是@PostConstruct也是常用的一种bean初始化执行方法的方式，但是这边初始化逻辑中是没有体现的）[@PostConstruct和初始化方法的执行顺序](#https://github.com/jlbluluai/xyz-notes/blob/master/study/Spring/Spring/解读Spring的bean创建时初始化函数的执行顺序.md)

该步骤执行完后我们看下日志：

![](http://img.yelizi.top/5381f250-1e13-451d-921f-558c984ca78c.jpg$xyz)



## 5. bean的销毁分析

那么bean销毁时干了什么呢，刚刚提到了创建bean时会注册销毁的回调，由于stop时没搞明白断点怎么生效，我只能从代码上去分析。

最终定位到`AbstractApplicationContext#close`然后顺着这个线路就来到了`DisposableBeanAdapter#destroy`，看这个玩意就知道是装饰器模式的运用，干了啥？执行了装饰的`DisposableBean`的destroy方法，并且再判断是否有自定义销毁函数再执行。

stop时也会打印如下日志：

![](http://img.yelizi.top/149634ac-a6fe-4925-aa28-8accbd1160be.jpg$xyz)