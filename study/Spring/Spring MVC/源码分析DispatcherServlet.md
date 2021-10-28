# Table of Contents

* [源码分析DispatcherServlet](#源码分析dispatcherservlet)
    * [分析点](#分析点)
    * [1. DispatcherServlet初始化机制](#1-dispatcherservlet初始化机制)
    * [2. DispatcherServlet的处理机制](#2-dispatcherservlet的处理机制)
        * [明确流程](#明确流程)
        * [源码分析](#源码分析)

# 源码分析DispatcherServlet


## 分析点

1. `DispatcherServlet`初始化机制
2. `DispatcherServlet`的处理机制


## 1. DispatcherServlet初始化机制

`DispatcherServlet`会在收到第一个请求时触发初始化方法`onRefresh`，最终调用的是`initStrategies`方法：

```java
protected void initStrategies(ApplicationContext context) {
	initMultipartResolver(context);
	initLocaleResolver(context);
	initThemeResolver(context);
	initHandlerMappings(context);
	initHandlerAdapters(context);
	initHandlerExceptionResolvers(context);
	initRequestToViewNameTranslator(context);
	initViewResolvers(context);
	initFlashMapManager(context);
}
```

可以看到条理清晰的初始化了很多解析器啥的，包括像`HandlerMapping`、`ViewResolver`之类也有初始化。




## 2. DispatcherServlet的处理机制

### 明确流程

`DispatcherServlet`可以说是Spring MVC的核心了，可以说整个请求执行流程都是围绕着它的，其实[开篇](#开篇)中简述的过程就是`DispatcherServlet`工作的简单描述，下面我们通过一张时序图更详细的描述这个过程。

![](http://img.yelizi.top/e9300626-66ea-4058-8675-72a18c9e8bfe.jpg$xyz)

1. 用户发送`request`请求，`DispatcherServlet` 接收；
2. `DispatcherServlet` 遍历所有配置的`HandlerMapping`请求查询`Handler`；
3. `HandlerMapping`根据请求的`URL`信息查到对应的`Handler`，以及相关的拦截器`interceptor`并构造`HandlerExecutionChain`；
4. 将构造的`HandlerExecutionChain`对象（里面包含了Handler和所有相关拦截器）等返回给`DispatcherServlet` ；
5. `DispatcherServlet`遍历所有的`HandlerAdapter`类并传递`Handler`请求执行；
6. `HandlerAdapter`执行`Handler`并获取`ModelAndView`对象；
7. `HandlerAdapter`将`ModelAndView`对象返回给`DispatcherServlet`；
8. `DispatcherServlet`遍历所有的`ViewResolver`请求进行视图解析；
9. `ViewResolver`进行视图解析并获取`View`对象；
10. `ViewResolver`向`DispatcherServlet`返回一个`View`对象；
11. `DispatcherServlet`进行视图的渲染填充`model`；
12. `DispatcherServlet`向用户返回响应；


### 源码分析


![](http://img.yelizi.top/de8294f2-6bd6-4dbe-bea5-fa68532ba20b.jpg$xyz)

`DispatcherServlet`本质上还是一个`Servlet`，所以请求响应最终调用的是`Servlet`的`service方法`。

我们在`DispatcherServlet`的`doService`方法开始处打个断点，随便请求个接口。跟`Servlet`有关的链路如下图所示。

![](http://img.yelizi.top/678f76bc-9198-4e8a-a5aa-35767e7e4ef7.jpg$xyz)


在`doService函数`中，除了给`request`加了一堆参数外，最重要的就是调用`doDispatch函数`（这个函数直切`DispatcherServlet`主题，一看就是个重要的东西）。我们直接分析源码：

```java
protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
	HttpServletRequest processedRequest = request;
	HandlerExecutionChain mappedHandler = null;
	boolean multipartRequestParsed = false;

	WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);

	try {
		ModelAndView mv = null;
		Exception dispatchException = null;

		try {
			processedRequest = checkMultipart(request);
			multipartRequestParsed = (processedRequest != request);

			// 针对当前request查找handler并得到一个HandlerExecutionChain对象
			mappedHandler = getHandler(processedRequest);
			if (mappedHandler == null) {
				noHandlerFound(processedRequest, response);
				return;
			}

			// 根据handler获取对应的HandlerAdapter
			HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());

			// Process last-modified header, if supported by the handler.
			String method = request.getMethod();
			boolean isGet = "GET".equals(method);
			if (isGet || "HEAD".equals(method)) {
				long lastModified = ha.getLastModified(request, mappedHandler.getHandler());
				if (new ServletWebRequest(request, response).checkNotModified(lastModified) && isGet) {
					return;
				}
			}

            // 执行拦截器的preHandle 如果有的话
			if (!mappedHandler.applyPreHandle(processedRequest, response)) {
				return;
			}

			// 由HandlerAdapter执行handler，并获取ModelAndView
			mv = ha.handle(processedRequest, response, mappedHandler.getHandler());

			if (asyncManager.isConcurrentHandlingStarted()) {
				return;
			}

			applyDefaultViewName(processedRequest, mv);
			// 执行拦截器的postHandle 如果有的话
			mappedHandler.applyPostHandle(processedRequest, response, mv);
		}
		catch (Exception ex) {
			dispatchException = ex;
		}
		catch (Throwable err) {
			// As of 4.3, we're processing Errors thrown from handler methods as well,
			// making them available for @ExceptionHandler methods and other scenarios.
			dispatchException = new NestedServletException("Handler dispatch failed", err);
		}
		processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);
	}
	catch (Exception ex) {
		triggerAfterCompletion(processedRequest, response, mappedHandler, ex);
	}
	catch (Throwable err) {
		triggerAfterCompletion(processedRequest, response, mappedHandler,
				new NestedServletException("Handler processing failed", err));
	}
	finally {
		if (asyncManager.isConcurrentHandlingStarted()) {
			// Instead of postHandle and afterCompletion
			if (mappedHandler != null) {
				mappedHandler.applyAfterConcurrentHandlingStarted(processedRequest, response);
			}
		}
		else {
			// Clean up any resources used by a multipart request.
			if (multipartRequestParsed) {
				cleanupMultipart(processedRequest);
			}
		}
	}
}
```

分析完里面几个重点如下：

1. 如何根据`request`得到`HandlerExecutionChain`
2. 如何根据`handler`得到`HandlerAdapter`
3. 如何如何执行`handler`
4. 拦截器执行的时机
    - 源码分析中已经提及
5. 视图解析器如何工作


[1、2、3详见](https://github.com/jlbluluai/xyz-notes/blob/master/study/Spring/Spring%20MVC/%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90HandlerMapping.md)

[5详见](https://github.com/jlbluluai/xyz-notes/blob/master/study/Spring/Spring%20MVC/%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90ViewResolver.md)
