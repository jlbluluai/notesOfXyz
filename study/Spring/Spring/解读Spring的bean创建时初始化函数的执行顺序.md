# Table of Contents

* [解读Spring的bean创建时初始化函数的执行顺序](#解读spring的bean创建时初始化函数的执行顺序)
    * [分析点](#分析点)
    * [分析](#分析)
        * [InitializingBean#afterPropertiesSet以及自定义函数的执行时机](#initializingbeanafterpropertiesset以及自定义函数的执行时机)
        * [@PostConstruct的函数执行时机](#postconstruct的函数执行时机)
    * [结论](#结论)


# 解读Spring的bean创建时初始化函数的执行顺序

## 分析点

本篇的嘉宾主要涉及三个：

1. 实现`InitializingBean#afterPropertiesSet`
2. 指定自定义函数
3. 通过`@PostConstruct`注解的函数


## 分析

### InitializingBean#afterPropertiesSet以及自定义函数的执行时机

直接看`AbstractAutowireCapableBeanFactory#invokeInitMethods`的源码（bean创建步骤中初始化里一步骤）：

```java
protected void invokeInitMethods(String beanName, final Object bean, @Nullable RootBeanDefinition mbd)
        throws Throwable {

        // 执行InitializingBean#afterPropertiesSet
        boolean isInitializingBean = (bean instanceof InitializingBean);
        if (isInitializingBean && (mbd == null || !mbd.isExternallyManagedInitMethod("afterPropertiesSet"))) {
        if (logger.isTraceEnabled()) {
        logger.trace("Invoking afterPropertiesSet() on bean with name '" + beanName + "'");
        }
        if (System.getSecurityManager() != null) {
        try {
        AccessController.doPrivileged((PrivilegedExceptionAction<Object>) () -> {
        ((InitializingBean) bean).afterPropertiesSet();
        return null;
        }, getAccessControlContext());
        }
        catch (PrivilegedActionException pae) {
        throw pae.getException();
        }
        }
        else {
        ((InitializingBean) bean).afterPropertiesSet();
        }
        }

        // 执行自定义的初始化函数
        if (mbd != null && bean.getClass() != NullBean.class) {
        String initMethodName = mbd.getInitMethodName();
        if (StringUtils.hasLength(initMethodName) &&
        !(isInitializingBean && "afterPropertiesSet".equals(initMethodName)) &&
        !mbd.isExternallyManagedInitMethod(initMethodName)) {
        invokeCustomInitMethod(beanName, bean, mbd);
        }
        }
        }
```

### @PostConstruct的函数执行时机

全局搜索哪边有对`@PostConstruct`的处理，定位到`InitDestroyAnnotationBeanPostProcessor`这个类，其实一下子就明了了，针对`@PostConstruct`的函数处理其实就是一个`BeanPostProcessor`的before执行。

具体咱就不分析了。


## 结论

[分析完bean的生命周期](#https://github.com/jlbluluai/xyz-notes/blob/master/study/Spring/Spring/解读Spring的bean创建.md)就知道`BeanPostProcessor`的before是先于初始化函数执行的，所以：

`@PostConstruct` 先于 `InitializingBean#afterPropertiesSet` 先于 `指定自定义初始化函数`


具体使用上建议优先使用`InitializingBean#afterPropertiesSet`
`InitializingBean#afterPropertiesSet`，其次`@PostConstruct`，这两者简单明了一眼就知道是初始化函数，自定义的方式看具体情况（SpringBoot里只能@Bean的方式或者xml的方式才能指定，不通用）。