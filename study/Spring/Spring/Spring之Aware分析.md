# Table of Contents

* [Spring之Aware分析](#spring之aware分析)
    * [概述](#概述)
    * [由bean生命周期引出](#由bean生命周期引出)
    * [自定义Aware实战](#自定义aware实战)
    * [总结](#总结)


# Spring之Aware分析

## 概述

> A marker superinterface indicating that a bean is eligible to be notified by the Spring container of a particular framework object through a callback-style method. The actual method signature is determined by individual subinterfaces but should typically consist of just one void-returning method that accepts a single argument.    -- 源码注释

上面摘自`org.springframework.beans.factory.Aware`接口注释，简单解释一下就是：

`Aware`接口作为一个标记接口，Bean若是实现了`Aware`接口(的子接口)，有资格通过回调方法感应到Spring容器。（如果你明白bean和Spring容器本身无耦合就能理解了）


## 由bean生命周期引出

我们首先回顾一张bean生命周期的经典图：

![](http://img.yelizi.top/c5a4f8be-2029-4677-987c-25b21b74e64c.jpg$xyz)

其中就有一步是关于`Aware`的，其实图中检查`Aware`之后接`BeanPostProcessor`前置处理是不太严谨的。

`Aware`本身不会生效，其实其也是借助一个`BeanPostProcessor`实现的，跟我们Spring容器密切相关的就是`ApplicationContextAwareProcessor`，看一下源码就明白了。


## 自定义Aware实战

自定义了一个设置`Logger`的`Aware`：

```java
public interface LoggerAware extends Aware {

    void setLogger(Logger logger);
}
```

对应的`BeanPostProcessor`：

```
@Component
public class MyAwareBeanProcessor implements BeanPostProcessor {

    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {

        if (bean instanceof LoggerAware) {
            ((LoggerAware) bean).setLogger(LoggerFactory.getLogger(bean.getClass()));
        }

        return bean;
    }
}
```

具体的bean：

```
@Service
public class AwareDemo implements LoggerAware {

    private Logger logger;

    public void test() {
        logger.info("test aware");
    }

    @Override
    public void setLogger(Logger logger) {
        this.logger = logger;
    }
}
```

这样我们的bean就不需要显示的初始化`Logger`就具备了其属性。


## 总结

不要过分理解，`Aware`就是个标记，Spring通过`BeanPostProcessor`给我们设置了一些`Aware`要求的属性而已。