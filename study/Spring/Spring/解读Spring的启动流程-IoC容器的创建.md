# Table of Contents

* [解读Spring的启动流程-IoC容器的创建](#解读spring的启动流程-ioc容器的创建)
    * [分析环境准备](#分析环境准备)
    * [分析点](#分析点)
    * [Spring启动是启动了啥](#spring启动是启动了啥)
    * [寻找启动容器的入口](#寻找启动容器的入口)
    * [容器初始化的工作](#容器初始化的工作)
        * [附1](#附1)
        * [附2](#附2)
        * [附3](#附3)


# 解读Spring的启动流程-IoC容器的创建


## 分析环境准备

Spring版本：5.1.5.RELEASE。

分析用项目：linzone。

基于SpringBoot启动方式追踪分析，至于xml等其他方式其实最终都是走到一个核心逻辑，最后做个补充讲讲其他方式在走到核心逻辑前有什么不一样。


## 分析点

以下分析点不代表后面分析的步骤，只是预告式的有哪些可以关注的点

1. Spring启动是启动了个啥
2. Bean是如何初始化的
3. Bean的生命周期是啥样的


## Spring启动是启动了啥

`IoC`容器，所以说是Spring的启动，其实就是`IoC`容器的初始化过程。

`IoC`在Spring中的工作本质上就是管理bean，在`spring-beans`包中提供了一个标准的bean管理接口`BeanFactory`，下面的讲解就会涉及到它。


## 寻找启动容器的入口

项目的启动方式是标准的SpringBoot启动方式：

```java
public static void main(String[] args) {
        SpringApplication.run(LinzoneApplication.class, args);
        }
```

上面提到了我们启动其实就是启动一个`IoC`容器，那么我们随着`run`函数一路向下来到创建容器的地方：

```java
protected ConfigurableApplicationContext createApplicationContext() {
        Class<?> contextClass = this.applicationContextClass;
        if (contextClass == null) {
        try {
        switch (this.webApplicationType) {
        case SERVLET:
        contextClass = Class.forName(DEFAULT_SERVLET_WEB_CONTEXT_CLASS);
        break;
        case REACTIVE:
        contextClass = Class.forName(DEFAULT_REACTIVE_WEB_CONTEXT_CLASS);
        break;
default:
        contextClass = Class.forName(DEFAULT_CONTEXT_CLASS);
        }
        }
        catch (ClassNotFoundException ex) {
        throw new IllegalStateException(
        "Unable create a default ApplicationContext, "
        + "please specify an ApplicationContextClass",
        ex);
        }
        }
        return (ConfigurableApplicationContext) BeanUtils.instantiateClass(contextClass);
        }
```

看源码可以看到，创建还有分支判断，这边就不是重点了，一般我们的Web走的就是`SEVLET`，也就是创建了一个`AnnotationConfigServletWebServerApplicationContext`对象：

```
public static final String DEFAULT_SERVLET_WEB_CONTEXT_CLASS = "org.springframework.boot."
		+ "web.servlet.context.AnnotationConfigServletWebServerApplicationContext";
```

创建完后顺着逻辑看，看到标志的refresh调用，最终就定位到`AbstractApplicationContext#refresh`，可以上一张图看看这个容器的关系：

![](http://img.yelizi.top/4140676b-918f-474a-9682-5389d0253eb2.jpg$xyz)


## 容器初始化的工作


重点就是这个`refresh`方法，我们继续分析：


```java
public void refresh() throws BeansException, IllegalStateException {
synchronized (this.startupShutdownMonitor) {
        // 一些准备工作，不涉及主逻辑
        prepareRefresh();

        // 这一步是准备好BeanFactory，但其实我们的链路中GenericApplicationContext构造时就创建了DefaultListableBeanFactory，所以在我们这个分析流程中这一步只是获取了一个已经初始化的BeanFactory
        ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

        // 这一步设置了BeanFactory的类加载器，添加了几个默认的BeanPostProcessor，还有注册了跟环境相关的几个bean
        prepareBeanFactory(beanFactory);

        // 上面所有的逻辑都只是做一些基础准备工作，从这边开始才是正式的逻辑开始
        try {
        // 这一步实际又注册了特殊的BeanPostProcessor
        postProcessBeanFactory(beanFactory);

        // 这一步就很关键了，所有非默认的BeanDefinition都是在在这一步完成注册，具体分析见附1
        invokeBeanFactoryPostProcessors(beanFactory);

        // 这一步就是注册 BeanPostProcessor 涉及到包括框架里定义的 还有自定义的 见附2
        registerBeanPostProcessors(beanFactory);

        // 国际化的设置 跟主流程关联不大
        initMessageSource();

        // 初始化事件广播器 跟主流程关联也不大
        initApplicationEventMulticaster();

        // 模板方法 拓展给子类可以初始化一些特殊的bean 注意是初始化 不是注册bean定义了
        onRefresh();

        // 注册监听器 实现ApplicationListener接口
        registerListeners();

        // 这一步很重要，实例化所有非懒加载的单例的bean，这一步后基本上的所有的bean就就位了，不再是空壳的bean定义了 见附3
        finishBeanFactoryInitialization(beanFactory);

        // 初始化完成的一些通知 标志初始化结束
        finishRefresh();
        }

        catch (BeansException ex) {
        if (logger.isWarnEnabled()) {
        logger.warn("Exception encountered during context initialization - " +
        "cancelling refresh attempt: " + ex);
        }

        // Destroy already created singletons to avoid dangling resources.
        destroyBeans();

        // Reset 'active' flag.
        cancelRefresh(ex);

        // Propagate exception to caller.
        throw ex;
        }

        finally {
        // Reset common introspection caches in Spring's core, since we
        // might not ever need metadata for singleton beans anymore...
        resetCommonCaches();
        }
        }
        }
```


### 附1

`invokeBeanFactoryPostProcessors`这一步最后主逻辑导向就是`PostProcessorRegistrationDelegate#invokeBeanFactoryPostProcessors`：

![](http://img.yelizi.top/4555b2b2-eb55-4d08-ba13-b83c1771d360.jpg$xyz)

可以看到这个方法参数是刚刚创建的`BeanFactory`以及三个应该是默认加入的`BeanFactoryPostProcessor`（请注意区分前面提到的默认注册的`BeanPostProcessor`）。


至于说我们的业务bean的`BeanDefinition`什么时候注册的，下面给出流程（不完整的分析每一步的函数全部源码了，太冗长了）。

`PostProcessorRegistrationDelegate#invokeBeanFactoryPostProcessors`

->

// 调用给定的 BeanDefinitionRegistryPostProcessor 进行注册，事实上这里只有一个就是下一步
`PostProcessorRegistrationDelegate#invokeBeanDefinitionRegistryPostProcessors`

->

`ConfigurationClassPostProcessor#postProcessBeanDefinitionRegistry`

->

// 这里主要就是以我们的启动类（Spring Boot房事）LinzoneApplication 为起点去找所有的bean定义
`ConfigurationClassPostProcessor#processConfigBeanDefinitions`

... ->

// 这一步主要就是根据启动类去解析出所有的bean类
`ConfigurationClassParser#doProcessConfigurationClass`

主要思路几个点（因为里面无限套娃，不方便整体去讲）：

1. 以一个类作为起点去解析（这里就是SpringBoot的启动类）
2. 主要扫描类似`ComponentScans`和`Component`这样的注解，然后找bean
3. 还有诸如`Import`这样的注解特殊玩法也能自定义注册`BeanDefinition`（这个比较有意思，自己封装服务并要实现完全配置化时可能会派上用场，我就用过，可以自己摸索，算是一个联动联想）

.. 其他不再详述，主要围绕大概如何注册了项目里所有的`BeanDefinition`就是这么回事。



### 附2

执行完后，我们自定义的`BeanPostProcessor`也在列表中了。至于`BeanPostProcessor`是干啥的，我们留个扩展点[解读BeanPostProcessor作用](#)

![](http://img.yelizi.top/117bd859-4a92-44ad-a5fe-2787eb0c6c0c.jpg$xyz)



### 附3


我们依旧按调用流程去分析：

`AbstractApplicationContext#finishBeanFactoryInitialization`

梦的开始

... ->

`DefaultListableBeanFactory#preInstantiateSingletons`

里面一个for循环遍历所有的bean定义

... ->

`AbstractBeanFactory#doGetBean`

精准的获取一个bean的方法（未注册的还会先创建），里面还涉及了一堆套娃（就是里面依赖的bean，也得注册下），总体来说，不是懒加载的bean都会被加载。

->

`AbstractAutowireCapableBeanFactory#createBean`

->

`AbstractAutowireCapableBeanFactory#doCreateBean`

具体的创建方法，具体bean创建[详细见](#https://github.com/jlbluluai/xyz-notes/blob/master/study/Spring/Spring/解读Spring的bean创建.md)