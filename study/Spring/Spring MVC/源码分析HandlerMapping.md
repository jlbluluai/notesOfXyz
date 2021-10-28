# Table of Contents

* [源码分析HandlerMapping](#源码分析handlermapping)
    * [分析点](#分析点)
    * [1. HandlerMapping初始化](#1-handlermapping初始化)
    * [2. Handler是哪来的](#2-handler是哪来的)
    * [3. 如何获取一个Handler工作](#3-如何获取一个handler工作)
        * [3.1 Handler如何执行](#31-handler如何执行)
    * [4. 附加的作业](#4-附加的作业)


# 源码分析HandlerMapping

## 分析点

围绕`HandlerMapping`就是几个重点：

1. `HandlerMapping`初始化
2. `Handler`是哪来的
3. 如何获取一个`Handler`工作
    1. `Handler`如何执行
4. 附加的作业


## 1. HandlerMapping初始化

在`DispatcherServlet`中有如下属性（所以初始化`HandlerMapping`也是发生在`DispatcherServlet`时）：

```java
private List<HandlerMapping> handlerMappings;
```

该属性就保存着所有的处理器映射，纵观全局，对其赋值的方法就是`DispatcherServlet`中`initHandlerMappings`。


我们启动项目并发送一个请求看看初始化时`handlerMappings`都放了些什么。


![](http://img.yelizi.top/37fffed3-6462-4751-b456-f1606b322180.jpg$xyz)


其实这几个都实现了`HandlerMapping`接口。

![](http://img.yelizi.top/44285577-c6e8-4454-a12b-e81bb9dcf927.jpg$xyz)


`BeanFactoryUtils.beansOfTypeIncludingAncestors`主要就是根据给定的接口找匹配的bean，这里的接口就是`HandlerMapping`。

```java
Map<String, HandlerMapping> matchingBeans =
		BeanFactoryUtils.beansOfTypeIncludingAncestors(context, HandlerMapping.class, true, false);
```


## 2. Handler是哪来的

既然在`DispatcherServlet`流程中，`Handler`是`HandlerMapping`找出来的，我们就关注下上面提及的初始化的`handlerMappings`，上面截图可以看到初始化了4个`HandlerMapping`，其中有一个`RequestMappingHandlerMapping`其实就是处理我们业务web请求的，我们关注它是怎么初始化的，也就能知道`Handler`是哪来的。


在`RequestMappingHandlerMapping`中有一个很重要的属性，其实是定义在其父类`AbstractHandlerMethodMapping`中的：

```java
private final MappingRegistry mappingRegistry = new MappingRegistry();
```

`MappingRegistry`中保存了所有的请求连接对应关系，它有一个方法`register`，`RequestMappingHandlerMapping`其实就是通过`InitializingBean`的方式循环获取bean去注册的。



## 3. 如何获取一个Handler工作

定位到`DispatcherServlet`中获取`handler`的方法，可以看到`HandlerMapping`的工作模式很简单，就是遍历然后根据`request`去尝试获取`HandlerExecutionChain`，一旦获取就立刻返回（所以如果有自定义`HandlerMapping`的一定要注意自定义放入的顺序，自定义的顺序问题引起的不生效在spring中还是挺常见的，就不展开说了）。

```java
protected HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
	if (this.handlerMappings != null) {
		for (HandlerMapping mapping : this.handlerMappings) {
			HandlerExecutionChain handler = mapping.getHandler(request);
			if (handler != null) {
				return handler;
			}
		}
	}
	return null;
}
```

那么这时候我们发起的是一个普通的请求，我们看看链路是怎么走的。

![](http://img.yelizi.top/9b54268c-d642-42d7-b24b-55e37c45ec37.jpg$xyz)

关注两个信息，生效的`HandlerMapping`是初始化里截图的`RequestMappingHandlerMapping`，看名字也能猜出来正常请求应该就是走他的，还有就是我们获取到了一个`handler`，这个`handler`是一个`HandlerMethod`方法，重点是它其中就能找到控制器的`bean`，并且还有方法的信息，那么显然有这些信息就能执行对应的方法了。

那么现在好奇的就是这个`handler`是如何获取的，顺着链路发现是取自`AbstractHandlerMethodMapping.mappingRegistry属性`，这里就和上面讲的串起来了，`mappingRegistry`看如下截图就能一目了然了（包含了项目所有的请求的映射信息）。

![](http://img.yelizi.top/4ec6ac36-d711-48d6-8b5f-82df0c409ee6.jpg$xyz)


### 3.1 Handler如何执行

[源码分析handler的执行](https://github.com/jlbluluai/xyz-notes/blob/master/study/Spring/Spring%20MVC/%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90handler%E7%9A%84%E6%89%A7%E8%A1%8C.md)

## 4. 附加的作业

上面提到了`DispatcherServlet`最终获取到的是一个`HandlerExecutionChain`，这玩意不仅包含了`Handler`还有拦截器，这拦截器什么时候加入的呢。


`RequestMappingHandlerMapping`获取`handler`的完整方法如下（关注`getHandlerExecutionChain`方法的调用）:

```java
public final HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
		Object handler = getHandlerInternal(request);
		if (handler == null) {
			handler = getDefaultHandler();
		}
		if (handler == null) {
			return null;
		}
		// Bean name or resolved handler?
		if (handler instanceof String) {
			String handlerName = (String) handler;
			handler = obtainApplicationContext().getBean(handlerName);
		}

        // 该方法就去取得匹配的拦截器和handler一起组合成HandlerExecutionChain
		HandlerExecutionChain executionChain = getHandlerExecutionChain(handler, request);

		if (logger.isTraceEnabled()) {
			logger.trace("Mapped to " + handler);
		}
		else if (logger.isDebugEnabled() && !request.getDispatcherType().equals(DispatcherType.ASYNC)) {
			logger.debug("Mapped to " + executionChain.getHandler());
		}

		if (CorsUtils.isCorsRequest(request)) {
			CorsConfiguration globalConfig = this.corsConfigurationSource.getCorsConfiguration(request);
			CorsConfiguration handlerConfig = getCorsConfiguration(handler, request);
			CorsConfiguration config = (globalConfig != null ? globalConfig.combine(handlerConfig) : handlerConfig);
			executionChain = getCorsHandlerExecutionChain(request, executionChain, config);
		}

		return executionChain;
	}
```