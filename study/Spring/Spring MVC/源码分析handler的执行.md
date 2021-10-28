# Table of Contents

* [源码分析handler的执行](#源码分析handler的执行)
    * [关注点](#关注点)
    * [分析](#分析)


# 源码分析handler的执行


## 关注点

1. 参数解析器和返回值解析器在里面如何起作用
2. 最终是如何执行的


## 分析

说道`Handler`的执行，就要说到另一个重要接口 `HandlerAdapter`，这玩意其实就是 [适配器模式](https://baike.baidu.com/item/%E9%80%82%E9%85%8D%E5%99%A8%E6%A8%A1%E5%BC%8F/10218946?fr=aladdin) 的运用，灵活应对各种`handler`，例如我们上面讲到当前请求方式的`handler`是一个`HandlerMethod`对象，最后其实就会找到`RequestMappingHandlerAdapter`这个适配器，为什么呢，看以下源码：

```java
public final boolean supports(Object handler) {
        return (handler instanceof HandlerMethod && supportsInternal((HandlerMethod) handler));
        }
```

那么我们就看关键的执行`handler`的逻辑（定位到`RequestMappingHandlerAdapter#handleInternal`）:

![](http://img.yelizi.top/cb123e4c-2a9f-46f5-9fa3-1741edb924d4.jpg$xyz)

别管里面的判断逻辑（这不是重点），关键点在于`invokeHandlerMethod`函数的调用，这里在流程里就拿到了`ModelAndView`（当然如果是request请求会返回null，因为结果已经反馈），直接上源码分析该函数：

```java
protected ModelAndView invokeHandlerMethod(HttpServletRequest request,
		HttpServletResponse response, HandlerMethod handlerMethod) throws Exception {

	ServletWebRequest webRequest = new ServletWebRequest(request, response);
	try {
		WebDataBinderFactory binderFactory = getDataBinderFactory(handlerMethod);
		ModelFactory modelFactory = getModelFactory(handlerMethod, binderFactory);

		ServletInvocableHandlerMethod invocableMethod = createInvocableHandlerMethod(handlerMethod);
		// 载入参数解析器
		if (this.argumentResolvers != null) {
			invocableMethod.setHandlerMethodArgumentResolvers(this.argumentResolvers);
		}
		// 载入返回值处理器
		if (this.returnValueHandlers != null) {
			invocableMethod.setHandlerMethodReturnValueHandlers(this.returnValueHandlers);
		}
		invocableMethod.setDataBinderFactory(binderFactory);
		invocableMethod.setParameterNameDiscoverer(this.parameterNameDiscoverer);

		ModelAndViewContainer mavContainer = new ModelAndViewContainer();
		mavContainer.addAllAttributes(RequestContextUtils.getInputFlashMap(request));
		modelFactory.initModel(webRequest, mavContainer, invocableMethod);
		mavContainer.setIgnoreDefaultModelOnRedirect(this.ignoreDefaultModelOnRedirect);

		AsyncWebRequest asyncWebRequest = WebAsyncUtils.createAsyncWebRequest(request, response);
		asyncWebRequest.setTimeout(this.asyncRequestTimeout);

		WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);
		asyncManager.setTaskExecutor(this.taskExecutor);
		asyncManager.setAsyncWebRequest(asyncWebRequest);
		asyncManager.registerCallableInterceptors(this.callableInterceptors);
		asyncManager.registerDeferredResultInterceptors(this.deferredResultInterceptors);

		if (asyncManager.hasConcurrentResult()) {
			Object result = asyncManager.getConcurrentResult();
			mavContainer = (ModelAndViewContainer) asyncManager.getConcurrentResultContext()[0];
			asyncManager.clearConcurrentResult();
			LogFormatUtils.traceDebug(logger, traceOn -> {
				String formatted = LogFormatUtils.formatValue(result, !traceOn);
				return "Resume with async result [" + formatted + "]";
			});
			invocableMethod = invocableMethod.wrapConcurrentResult(result);
		}

        // 真正执行
		invocableMethod.invokeAndHandle(webRequest, mavContainer);
		if (asyncManager.isConcurrentHandlingStarted()) {
			return null;
		}

        // 若有ModelAndView返回就返回 
		return getModelAndView(mavContainer, modelFactory, webRequest);
	}
	finally {
		webRequest.requestCompleted();
	}
}
```

里面有两个很有意思的点，这里会设置参数解析器和返回值处理器，这两个玩意自己扩展spring时肯定会遇到，挺有意思的，其他地方会讲。回归正题，该函数主要就是准备好了一大堆参数，真正执行的地方还得继续往下：

```java
invocableMethod.invokeAndHandle(webRequest, mavContainer);
```

这里就追踪定位到`InvocableHandlerMethod#invokeAndHandle`：

```java
public Object invokeForRequest(NativeWebRequest request, @Nullable ModelAndViewContainer mavContainer,
		Object... providedArgs) throws Exception {

    // 获取参数 里面还涉及到参数解析器的运作，可以看看
	Object[] args = getMethodArgumentValues(request, mavContainer, providedArgs);
	if (logger.isTraceEnabled()) {
		logger.trace("Arguments: " + Arrays.toString(args));
	}
	
	// 具体执行 有对象 有方法 有参数 猜一猜
	// 没错 反射执行
	return doInvoke(args);
}
```

再回到上一级`ServletInvocableHandlerMethod#invokeAndHandle`:

```java
public void invokeAndHandle(ServletWebRequest webRequest, ModelAndViewContainer mavContainer,
		Object... providedArgs) throws Exception {

    // 执行拿到返回值 上面有分析
	Object returnValue = invokeForRequest(webRequest, mavContainer, providedArgs);
	setResponseStatus(webRequest);

	if (returnValue == null) {
		if (isRequestNotModified(webRequest) || getResponseStatus() != null || mavContainer.isRequestHandled()) {
			mavContainer.setRequestHandled(true);
			return;
		}
	}
	else if (StringUtils.hasText(getResponseStatusReason())) {
		mavContainer.setRequestHandled(true);
		return;
	}

	mavContainer.setRequestHandled(false);
	// 执行返回值处理器 像RequestResponseBodyMethodProcessor就会直接返回了 并且mavContainer.setRequestHandled(true); 这样后面ModelAndView就会直接返回null
	Assert.state(this.returnValueHandlers != null, "No return value handlers");
	try {
		this.returnValueHandlers.handleReturnValue(
				returnValue, getReturnValueType(returnValue), mavContainer, webRequest);
	}
	catch (Exception ex) {
		if (logger.isTraceEnabled()) {
			logger.trace(formatErrorForReturnValue(returnValue), ex);
		}
		throw ex;
	}
}
```