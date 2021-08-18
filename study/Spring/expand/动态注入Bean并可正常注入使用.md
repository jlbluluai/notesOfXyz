# Table of Contents

* [动态注入Bean并可正常注入使用](#动态注入bean并可正常注入使用)
  * [前因](#前因)
  * [过程1（惨烈失败）](#过程1（惨烈失败）)
  * [过程2（八仙过海，各显神通）](#过程2（八仙过海，各显神通）)
  * [过程3（突破）](#过程3（突破）)
  * [总结](#总结)


# 动态注入Bean并可正常注入使用

## 前因

动态注入Bean，知名搜索网站一堆方案，可至少在我遇到的场景都没办起作用。

描述下场景，想要统合一个文件服务并单独出来作为一个starter（Spring Boot的），然后通过application.yml的配置实现动态服务配置。

简单来说，我要通过application.yml去配置动态的bean的属性，这个bean没有任何其他标志，全凭配置。


## 过程1（惨烈失败）

通过实现`BeanFactoryAware`，这个类注册为Bean，并注入绑定了配置属性的bean（自动绑定属性到类这个自行研究）。

然后在`BeanFactoryAware`的实现方法`setBeanFactory`中通过自带的参数beanFactory（需转成`ListableBeanFactory`）根据配置去注册bean，看似很完美。

然后，直接翻车，在业务类中注入配置的服务，启动，Game Over，找不到bean，注入无效。聪明的小脑瓜一转加个`@lazy`，成功。这里其实就看出来了，不是动态注册bean失败，而是好像注册的时机不对，至少在实例化了业务bean后，注入需要的bean时是找不到的。

那就加个`@Lazy`就好了呗，这么没追求？强制使用者进行非平常的操作，显然不是一个优秀设计者该干的事情，继续找寻方案。

## 过程2（八仙过海，各显神通）

过程就不多描述了，基本上能在知名搜索网站搜到的都试过了。

这个过程中对于当前问题产生的原因也明目了，为啥会找不到bean，是因为bean的创建过程中前一步其实还有一个注册BeanDefinition的过程，有了这个保底，后面创建bean时比如注入另外一个bean，但这个bean还没创建，但只要BeanDefinition能找到，就会自动创建。

而上述的做法就是在所有bean创建完后强硬的又注册了几个额外的bean进去，那显然在bean创建过程中确实是找不到的，这也是为什么`@Lazy`就能管用（使用时才注入，那就算后插进去的bean也已经准备好了）。

其实，到这也知道应该怎么处理了，既然bean的创建是依据BeanDefinition，那在bean创建前注册我们动态的BeanDefinition不就行了吗。

## 过程3（突破）

方案已经找到，但面临一个问题，注册BeanDefinition阶段是没有bean的概念的，也就是你自动绑定配置的类是没有用的。那怎么获取配置？别慌，调试Spring源码，发现所有的属性起始一开始在搞bean这玩意之前就已经全部解析到Envirment中了。

然后抱着尝试的心态就发现了`EnvironmentAware`，结果就如下代码所示

```
public class XXXBeanDefinitionRegistrar implements ImportBeanDefinitionRegistrar, EnvironmentAware {

    /**
     * 环境配置（可取得配置文件中的配置）
     */
    private ConfigurableEnvironment environment;


    @Override
    public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
        // 具体配置了啥，按具体情况，完整的key即可获取值（按照applition.properties中的配置理解）
        String xxx = environment.getProperty("a.b.c.d");
    }

    @Override
    public void setEnvironment(Environment environment) {
        this.environment = (ConfigurableEnvironment) environment;
    }
}
```

然后在配置类上加一个注解引入这个类
```
@Import(XXXBeanDefinitionRegistrar.class)
```

若是配置复杂，不一定非得在这把参数都解析出来，可以只解析必要的属性，bean的名字以及bean要用到的类。然后再回到最初的方案`BeanFactoryAware`，这时候再获取bean去补齐里面的属性。


## 总结

只是一个经历的自述总结，用作记录这次探索的的相对收货，至少有一点，对于Spring bean的整个生命周期又有了一个全新的认知。

**可做参考的实现案例**：[实现案例](https://github.com/jlbluluai/xyz-support/blob/master/src/main/java/com/xyz/support/file/FileConfiguration.java)
