# Table of Contents

* [Spring之Transaction分析](#spring之transaction分析)
    * [概述](#概述)
    * [Spring事务机制分析](#spring事务机制分析)
        * [分析事务代理类](#分析事务代理类)
    * [基于xml的声明式事务](#基于xml的声明式事务)
    * [基于注解的声明式事务](#基于注解的声明式事务)
    * [开启Spring声明式事务的方式](#开启spring声明式事务的方式)
    * [Spring事务的传播行为](#spring事务的传播行为)
    * [Spring事务的隔离级别](#spring事务的隔离级别)
    * [Spring事务的rollbackFor](#spring事务的rollbackfor)


# Spring之Transaction分析

## 概述

Spring的一个亮点就在于利用代理实现了非嵌入的事务支持，我们一般称之为声明式事务。区别于编程式事务（或者你可以称之为显式事务），声明式事务对我们的业务几乎是领侵入。

本篇旨在分析Spring的事务机制以及声明式事务的原理。

## Spring事务机制分析

万变不离其宗，要了解Spring的事务机制，我们首先要明白数据库的事务是个啥玩意，在Java里面，JDBC客户端又是如何进行事务的。我假设各位都明白，继续向下看。

那么，其实不难想象，要非侵入的织入事务的开启、提交以及回滚那必然会用到代理，鉴于事务的方法不一定是接口方法，我猜测其代理必然使用的是Cglib动态代理（实际就是如此，下文慢慢分析）。同样的我假设各位对代理以及Cglib动态代理都是了解的，继续向下看。

**源码提要**：定位Spring事务的源码处肯定会有整体的印象，`spring-tx`包下`org.springframework.transaction`目录就是Spring事务相关所有核心支持类了。


我们把目光聚焦到`org.springframework.transaction.TransactionInterceptor`，熟悉Spring AOP的同学一下子就看出其是一个代理处理类（不要先去纠结什么时机、怎么调用的啥啥，直接从原理出发找到这里是不是让自己条理更清晰呢？）。

其核心方法`invoke`，我们通过调用一个案例来看一下，如图：

![](http://img.yelizi.top/847a0341-41a0-4a52-a942-51ae4cc2638b.jpg$xyz)

`invoke`方法的参数`MethodInvocation`可以看到包含我们要执行的方法的所有信息（拿到类、方法信息，能干啥不用我明说了吧）。

显然具体事务相关的操作在方法`invokeWithinTransaction`中，具体怎么操作的，可以先想象下，好了，继续看。

这个方法比较长还有分支，先别被绕进去，我们不去看细节，就先看if逻辑找事务操作，如图（关注红框）：

![](http://img.yelizi.top/f6b829e9-bc2e-4b96-ab19-0d9b997a2292.jpg$xyz)


很好懂的事务开启、异常回滚、提交，好了，至此，粗略的至少我们不会不明不白Spring的事务是如何操作的。


### 分析事务代理类

既然明白了声明式事务是通过动态代理实现的，那能看一下代理出类显然有助于我们去分析。

不卖关子，启动参数加如下配置：

```
-Dcglib.debugLocation=自己指定的路径（可以指定本项目全路径方便查看）
```

然后启动项目即可将动态生成的代理类文件生成，看下图：

![](http://img.yelizi.top/bc683d57-2ce9-4efb-bff2-4846052ffd6a.jpg$xyz)

可以看出其是一个标准的Cglib动态代理类。


## 基于xml的声明式事务

...


## 基于注解的声明式事务

相较于xml声明式事务，注解方式显的更常用，并且当下主流的SpringBoot项目中再引入xml配置倒显得怪异了。

基于注解的声明式事务的关键注解就是`Transactional`,其位于`spring-tx`包下。

表明一个方法开启事务，很简单，比如：

```
@Transactional(rollbackFor = Exception.class)
public void doAdd(){
    TestEntity testEntity = new TestEntity();
    testEntity.setName("小mi");
    testEntity.setAge(22);
    testDao.insert(testEntity);
    int i = 1 / 0;
}
```

## 开启Spring声明式事务的方式

无论是基于xml还是基于注解，单纯的使用是不会有作用，简单的说就是没有事务管理器，它们就是无根之萍（事务管理器我将再开篇分析）。事务管理器默认是不开启的，我们需要开启：

1. 若还引有Spring xml配置，可以配置`<tx:annotation-driven />`
2. 找个配置类或者bean类加上`@EnableTransactionManagement`注解
3. Springboot自动就有配置类加上了`@EnableTransactionManagement`注解（笔者2.1.3.RELEASE版本已经可以，至于之前能不能需要读者自己分析尝试）


## Spring事务的传播行为

Spring的声明式事务要是只是简单的剥离侵入，那简直太过小瞧它了，如果你也仅仅这么用过那就太可惜了。

其实Spring声明式事务最大的特点就是其传播行为的设置，在`Transactional`注解中有一项配置：

```
Propagation propagation() default Propagation.REQUIRED;
```

这就是设置传播行为的，其枚举几个值我罗列并介绍一下：

- PROPAGATION_REQUIRED：如果当前没有事务创建事务，有就加入
- PROPAGATION_SUPPORTS：如果当前有事务就加入，没有就没有事务
- PROPAGATION_MANDATORY：如果当前有事务就加入，没有就抛出异常
- PROPAGATION_REQUIRES_NEW：新建事务，如果当前有事务，则挂起当前事务
- PROPAGATION_NOT_SUPPORTED：非事务执行，如果当前有事务，则挂起当前事务
- PROPAGATION_NEVER：以非事务执行，如果当前有事务，则抛出异常
- PROPAGATION_NESTED：如果当前有事务就创建一个(子)事务嵌套在当前事务中，没有就创建事务


正是有了这些传播行为的支撑，在处理复杂的业务逻辑时，只要组合得当，就能让我们的代码健壮性得到很强的保证（当然如果理不清还是别组合了，组合不得当自己给自己埋Bug）。


## Spring事务的隔离级别

在`Transactional`注解中还有一项有意思的配置：

```
Isolation isolation() default Isolation.DEFAULT;
```

其是设置隔离级别的，隔离级别的不懂的去看数据库知识。

有同学就好奇了，隔离级别不是数据库的吗，Spring设置这玩意是啥意思？如果你明白数据库的隔离级别是客户端级别的，就不会有疑问了。

## Spring事务的rollbackFor


同样的作为`Transactional`注解中一项配置：

```
Class<? extends Throwable>[] rollbackFor() default {};
```

该配置决定了什么异常决定了事务会回滚，不要觉得不配置无所谓，默认只对`Error`和`RuntimeException`响应回滚。

实际开发，大部分项目都会基于`Exception`封装自己的业务异常，这显然不属于默认回滚范畴。

所以实际开发中若是实在不想动脑子因地制宜，就索性直接指定`Exception`，也就是所有异常都回滚：

```
@Transactional(rollbackFor = Exception.class)
```