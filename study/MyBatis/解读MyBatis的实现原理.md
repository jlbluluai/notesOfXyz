# Table of Contents

* [解读MyBatis的实现原理](#解读mybatis的实现原理)
    * [准备](#准备)
    * [分析点](#分析点)
    * [1. 实现一个简单的Mapper案例](#1-实现一个简单的mapper案例)
    * [2. Mapper的原理是啥](#2-mapper的原理是啥)
        * [2.1 MyBatis初始化-SqlSessionFactory的生成](#21-mybatis初始化-sqlsessionfactory的生成)
        * [2.2 SqlSession是啥](#22-sqlsession是啥)
        * [2.3 SqlSession如何生成一个Mapper](#23-sqlsession如何生成一个mapper)
        * [2.4 Mapper的代理类如何处理](#24-mapper的代理类如何处理)
    * [3. Spring如何完成对MyBatis的整合](#3-spring如何完成对mybatis的整合)
        * [3.1 Mapper是如何注册进BeanDefinition集合的](#31-mapper是如何注册进beandefinition集合的)
        * [3.2 Mapper是如何生成bean的](#32-mapper是如何生成bean的)


# 解读MyBatis的实现原理

## 准备

本篇将基于MyBatis3以及SpringBoot2（Spring5）的源码进行分析。

[MyBatis3官中文档](https://mybatis.org/mybatis-3/zh/index.html)

## 分析点

众所皆知，MyBatis是一个ORM框架，但并非是Spring的原生生态，所以本篇会大的分为两部分分析，先单纯的分析MyBatis实现原理，再分析Spring和MyBatis是如何结合的。

1. 实现一个简单的Mapper案例
2. Mapper的原理是啥
3. Spring如何完成对MyBatis的整合


## 1. 实现一个简单的Mapper案例

只放出主代码，配置文件以及mapper的文件自行参考官档。

```
public static void main(String[] args) throws IOException {
    String resource = "mybatis-config.xml";
    InputStream inputStream = Resources.getResourceAsStream(resource);
    SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream);

    SqlSession sqlSession = sqlSessionFactory.openSession();
    TestDao mapper = sqlSession.getMapper(TestDao.class);
    TestEntity testEntity = mapper.selectByPrimaryKey(1);
    System.out.println(testEntity);
}
```

![](http://img.yelizi.top/d94a6b00-a8f7-471b-ae9b-9fba1a3c49b0.jpg$xyz)


## 2. Mapper的原理是啥

我们观察上面实现的简单案例，主要切入点发现两个：

1. 通过xml配置文件（当然也可以直接Java配置）生成一个`SqlSessionFactory`
2. `SqlSessionFactory`中打开了一个`SqlSession`，`SqlSession`就可以获取指定的mapper


### 2.1 MyBatis初始化-SqlSessionFactory的生成

我们顺着案例中的build方法定位到`SqlSessionFactoryBuilder#build`：

![](http://img.yelizi.top/7ad9c15a-078b-49d4-88e9-d4c5ddc95ecd.jpg$xyz)

三个参数后两个开源忽略，这里的`SqlSessionFactoryBuilder`是标准的建造者模式的运用，我们案例的build调用后两个参数都是null，所以这里就是根据配置xml的流拿到一个`XMLConfigBuilder`，这个`XMLConfigBuilder`再调用parse方法得到一个`Configuration`，然后构建了一个`DefaultSqlSessionFactory`。

![](http://img.yelizi.top/79a60bae-3884-46c5-a9f3-50fade932a19.jpg$xyz)

`XMLConfigBuilder#parse`代码很好懂，就是解析出如图所示的一堆标签，然后返回一个`Configuration`，这里就不展开了，可以简单理解这个`Configuration`具备了数据库连接信息以及Mapper信息等等。

`DefaultSqlSessionFactory`就是`SqlSessionFactory`的默认实现，其唯一的属性就是`Configuration`，所以单纯就是根据`Configuration`拿到一个`SqlSession`的工厂类。



### 2.2 SqlSession是啥

`SqlSession`是`SqlSessionFactory`通过调用openSession生成，这玩意也是个建造者模式的实现，最终定位到`DefaultSqlSessionFactory#openSessionFromDataSource`

![](http://img.yelizi.top/0c8faad6-cabd-4c82-97a8-1354a156fe0e.jpg$xyz)

这里面取出数据库信息，生成事务，并最终得到一个`Executor`（这玩意不要忽略，名字就不一般，最终执行就是靠它的）并和`Configuration`一起作为参数构建了一个`DefaultSqlSession`。

`SqlSession`作为一个接口，其下定义了几乎所有的sql操作函数，也就是说，只要拿到`SqlSession`，你就能访问数据库，当然一般来说我们还是会借助Mapper类去访问，更直观，这里就要提到其中的getMapper方法：

```
<T> T getMapper(Class<T> type);
```

### 2.3 SqlSession如何生成一个Mapper

看默认实现中，最后还是到了`Configuration`中：

```java
public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
        return mapperRegistry.getMapper(type, sqlSession);
        }
```

mapperRegistry是`Configuration`中维护一个`MapperRegistry`属性，在初始化时会将所有的mapper信息放入其中，我们打个断点到初始化结束：

![](http://img.yelizi.top/80ffcf74-69c2-46b3-a024-8d4137a923fe.jpg$xyz)


至于`MapperRegistry`如何获取mapper的，代码很明确：

![](http://img.yelizi.top/0f3fe3f5-fe5e-42e5-9628-953bc6a05744.jpg$xyz)


这时候我们就可以把目光定位到`MapperProxyFactory`：

![](http://img.yelizi.top/742a8056-6249-45e0-b6ea-19e6c7985647.jpg$xyz)

`MapperProxy`是一个`InvocationHandler`（JDK动态代理的实现方式），而且可以看到通过`Proxy.newProxyInstance`的方式生成了代理对象，真相就大白了。


### 2.4 Mapper的代理类如何处理

既然是用了JDK动态代理的方式，那么关注那个`InvocationHandler`的invoke逻辑即可，也就是在`MapperProxy`中：

![](http://img.yelizi.top/2730bbf9-dff2-4488-9b02-0a8aa77c83eb.jpg$xyz)

上面的try逻辑主要针对Object和接口默认实现方法的处理，我们不管，主要关注下面生成了一个`MapperMethod`并且调用了其execute方法。

`MapperMethod`的构造我们可以看到传入了接口信息、方法信息以及配置信息，并且execute方法传入了`SqlSession`和需要的参数，那我们就把目光聚焦到`MapperMethod#execute`：


![](http://img.yelizi.top/12b3aa9f-110f-43e9-9e6e-7a4a706ef24e.jpg$xyz)

这下就明了了，最终还是调用的`SqlSession`那些已经实现的执行方法。



## 3. Spring如何完成对MyBatis的整合


这里就直接以我的项目`linzone`做现成的分析案例。想一想，区别就在于我们的Mapper全部初始化后要作为bean交给Spring的IoC容器管理，那么猜猜MyBatis在哪一步整合进去。

那么我们回顾下Spring IoC容器初始化流程，首先会生成`BeanDefinition`，然后若是非懒加载的单例bean初始化时就会创建。那么，要整合MyBatis，必然也是先注册每个Mapper进入`BeanDefinition`集合，再去生成bean，也就是分成两部分。

以下分析基于`@MapperScan`的方式，其他方式后面再看。


### 3.1 Mapper是如何注册进BeanDefinition集合的

有过自己写组件动态注册bean经验的人就知道Spring留了`ImportBeanDefinitionRegistrar`这么个口子，只需要实现它并且在相关配置类通过`@Import`引入，Spring就会执行里面的方法去注册自定义的bean，关于这个的原理这里就不展开叙述了。我们直到`@MapperScan`方式就是通过这个实现注册就行了，看MapperScan注解上引入的`MapperScannerRegistrar`就是一个`ImportBeanDefinitionRegistrar`。

![](http://img.yelizi.top/9cf42d30-d06b-43f1-8c3a-4d5bf70a5bd8.jpg$xyz)


比如我们如下的使用（执行要扫描的包路径以及SqlSessionFactory）：

```
@MapperScan(basePackages = {"com.xyz.linzone.dao.function"}, sqlSessionFactoryRef = "sqlSessionFactoryFun")
```

我们就看看到底如何根据这个路径`com.xyz.linzone.dao.function`把这个路径下的Mapper都注册为`BeanDefinition`，最终定位到`ClassPathScanningCandidateComponentProvider#scanCandidateComponents`，可以看看整个请求链路：

![](http://img.yelizi.top/6010d880-31ec-45fb-8774-18cc5bb70447.jpg$xyz)


该步骤会将指定路径下所有的接口注册为`BeanDefinitionHolder`，当然到这里还不算完，这样这个`BeanDefinitionHolder`不会有任何作用，仔细想想它又和我们的MyBatis有啥联系呢，关键在于`ClassPathMapperScanner#doScan`中拿到了这些基础`BeanDefinitionHolder`后又进行了一层封装：

![](http://img.yelizi.top/ee46f8ac-2c9c-413d-925b-140e84962d2f.jpg$xyz)

它里面最重要的逻辑就是取出`BeanDefinitionHolder`中`BeanDefinition`将原本挂在的Mapper接口的class信息替换成`MapperFactoryBean.class`，并且将`sqlSessionFactory`以及`SqlSessionTemplate`（这玩意也是SqlSession实现，专为Spring整合MyBatis做的）作为属性塞入`BeanDefinition`，至于这样搞完后`BeanDefinitionHolder`是啥样的，举个例子（注意图中红圈的属性）：

![](http://img.yelizi.top/21d393ce-7c96-4ca9-ac6c-e299a4f3338f.jpg$xyz)

那么这个`MapperFactoryBean`是干啥的，就看下一小节分析。


### 3.2 Mapper是如何生成bean的

接着上面讲，Mapper都已经注册称为`BeanDefinition`了，那下一步就是生成bean了。

分析Spring时通常bean的创建方式无非取出`BeanDefinition`中的class信息然后通过反射去创建，那显然对Mapper行不通，所以这也就是为啥Mapper的`BeanDefinition`要把class信息替换为`MapperFactoryBean`，因为我们要通过它去创建Mapper的代理对象（和MyBatis原本挂钩）：

![](http://img.yelizi.top/d33cef15-fc4d-43f7-8a45-67e2522bf748.jpg$xyz)

源码可以看到`MapperFactoryBean`本身也是一个`FactoryBean`的实现，并且就是通过`SqlSession#getMapper`去生成Mapper对象的，最后就去作为一个bean实例存在了。

至于Spring中最终是如何去调用生成，我们回到Spring的创建bean去追踪。

在`AbstractBeanFactory#doGetBean`如图所示这一步，会先生成一个`MapperFactoryBean`的实例，然后再根据这个实例往下最后就会调取其中的getObject方法，那就能生成Mapper的代理对象了。

![](http://img.yelizi.top/03f1c331-7ecc-420e-8692-f6bc3e115432.jpg$xyz)