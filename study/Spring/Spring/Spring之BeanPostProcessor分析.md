# Table of Contents

* [Spring之BeanPostProcessor分析](#spring之beanpostprocessor分析)
    * [概述](#概述)
    * [实践](#实践)
    * [生效机制](#生效机制)
    * [总结](#总结)


# Spring之BeanPostProcessor分析


## 概述

`BeanPostProcessor`是Spring管理bean的一个很重要的机制，其两个方法分别代表bean初始化前执行和初始化后执行。

## 实践

```
@Component
@Slf4j
public class DemoBeanProcessor implements BeanPostProcessor {

    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
        log.info("初始化bean before beanName={}", beanName);
        return bean;
    }

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
        log.info("初始化bean after beanName={}", beanName);
        return bean;
    }

}
```

只要确定这个类的路径在启动时能被扫描到注册为bean，它就生效了。

我的处理对bean未加限制，启动时就会看到满屏打印这两行日志（Spring本身就有很多bean）。


## 生效机制

回顾IoC容器启动的关键方法`refresh`，启动涉及这里机制的两步：

```
...
invokeBeanFactoryPostProcessors(beanFactory);

registerBeanPostProcessors(beanFactory);
...
```

上一步保证了我们的`BeanPostProcessor`已经注册了BeanDefinition，这样我们才能找到。

下一步则就是注册了一堆默认`BeanPostProcessor`以及自定义的`BeanPostProcessor`（各类框架、纯粹我的自定义的都算）。

如下图，就是扫描到自定义`BeanPostProcessor`的关键处（注意红框）：

![](http://img.yelizi.top/99f56a11-c02f-44e6-9c5f-0f63e05d444d.jpg$xyz)

不要去纠结细节，是不是已经很好理解了，至此`BeanPostProcessor`已经注册完成，接下来看怎么使用。

看如下两个方法（`AbstractAutowireCapableBeanFactory`类中）：

![](http://img.yelizi.top/cd38ac1c-1c27-424d-97b5-d1d0d6db4845.jpg$xyz)

很清晰的循环处理逻辑，至于哪边调用的这两个的方法，各位可以自行去看下，就是在创建bean的流程中。


## 总结

`BeanPostProcessor`也没啥奥秘，作用也就是在bean初始化前后做一些处理，比如`Aware`、`@PostConstruct`等等你能想到的非侵入扩展bean的操作几乎都是借由`BeanPostProcessor`实现的。