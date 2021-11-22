# Table of Contents

* [解读Spring中AOP如何生效](#解读spring中aop如何生效)
    * [准备](#准备)
    * [1. Demo实现Spring的AOP](#1-demo实现spring的aop)
    * [2. 分析Spring的AOP的生效原理](#2-分析spring的aop的生效原理)


# 解读Spring中AOP如何生效

## 准备

首先，基于对AOP的逻辑已经有了解继续往下看，本篇不对AOP本身含义做什么解释。

那么，既然是面向切面编程，目的就是为了不对主逻辑造成侵入但实现业务代码的织入，那么Spring中AOP是如何生效的呢。

本篇基于几个点展开叙述：

1. Demo实现Spring的AOP
2. 分析Spring的AOP的生效原理


## 1. Demo实现Spring的AOP

既然只是为了分析，就也没必要那么全面，本篇只以前置增强举例：

```java
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE, ElementType.METHOD})
@Documented
public @interface TestAopBefore {
}

@Aspect
@Component
public class TestAopHandler {

    @Before("@annotation(TestAopBefore)")
    public void doBefore(JoinPoint joinPoint) {
        System.out.println("执行了切面的before");
    }
}

    @TestAopBefore
    public void sayHello(){
        System.out.println("hello");
    }
```

套壳一个Controller后调用执行下，结果如下：

![](http://img.yelizi.top/e973eed9-8feb-4c36-93c0-73e149d80fc4.jpg$xyz)



## 2. 分析Spring的AOP的生效原理


那么Demo中的`TestAopHandler`是如何生效的呢？

可以先猜想一下，首先进行代理创建代理bean那必然IoC容器管理的就是代理后的bean而不是原始bean，那么生效的时机必然在bean创建时。回顾下bean创建流程，其实到这已经不难猜了，bean创建流程中，对bean进行包装处理的就是`BeanPostProcessor`，实际上Spring AOP生效就是借助了`BeanPostProcessor`。

Debug后我们定位到`AbstractAutoProxyCreator#postProcessAfterInitialization`，熟悉的`BeanPostProcessor`的after处理，至于为啥是after，不难想象，要生成最终代理bean，我们肯定希望这时候的原始bean该初始化的都准备好了然后只需要织入切面动态生成代理bean已经具备了原始bean的全部属性。


![](http://img.yelizi.top/754e9fc8-ff44-4678-8d3b-877e2403597d.jpg$xyz)

`wrapIfNecessary`方法指向`createProxy`，这个函数已经具备了bean的信息，原始bean以及要切入的增强（Advisor）。


```java
protected Object createProxy(Class<?> beanClass, @Nullable String beanName,
@Nullable Object[] specificInterceptors, TargetSource targetSource) {

        if (this.beanFactory instanceof ConfigurableListableBeanFactory) {
        AutoProxyUtils.exposeTargetClass((ConfigurableListableBeanFactory) this.beanFactory, beanName, beanClass);
        }

        // Spring AOP用来处理代理的工厂类
        ProxyFactory proxyFactory = new ProxyFactory();
        proxyFactory.copyFrom(this);

        // 设置了ProxyFactory一些属性
        if (!proxyFactory.isProxyTargetClass()) {
        if (shouldProxyTargetClass(beanClass, beanName)) {
        proxyFactory.setProxyTargetClass(true);
        }
        else {
        evaluateProxyInterfaces(beanClass, proxyFactory);
        }
        }

        Advisor[] advisors = buildAdvisors(beanName, specificInterceptors);
        proxyFactory.addAdvisors(advisors);
        proxyFactory.setTargetSource(targetSource);
        customizeProxyFactory(proxyFactory);

        proxyFactory.setFrozen(this.freezeProxy);
        if (advisorsPreFiltered()) {
        proxyFactory.setPreFiltered(true);
        }

        // 获取代理后的bean
        return proxyFactory.getProxy(getProxyClassLoader());
        }
```

继续往下就走到`DefaultAopProxyFactory#createAopProxy`

```java
@Override
public AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException {
        if (config.isOptimize() || config.isProxyTargetClass() || hasNoUserSuppliedProxyInterfaces(config)) {
        Class<?> targetClass = config.getTargetClass();
        if (targetClass == null) {
        throw new AopConfigException("TargetSource cannot determine target class: " +
        "Either an interface or a target is required for proxy creation.");
        }
        // 若目标类是接口 则用JDK动态代理 否则 Cglib动态代理
        if (targetClass.isInterface() || Proxy.isProxyClass(targetClass)) {
        return new JdkDynamicAopProxy(config);
        }
        return new ObjenesisCglibAopProxy(config);
        }
        else {
        return new JdkDynamicAopProxy(config);
        }
        }
```

然后就是执行`JdkDynamicAopProxy#getProxy`或者`CglibAopProxy#getProxy`，熟悉这两种动态代理的接下来就很好明白了，要熟悉就肯定得知道两种动态代理的原理，这里暂时不展开叙述。