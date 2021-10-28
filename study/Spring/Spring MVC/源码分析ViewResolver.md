# Table of Contents

* [源码分析ViewResolver](#源码分析viewresolver)
    * [分析点](#分析点)
    * [ModelAndView是啥](#modelandview是啥)
    * [ViewResolver初始化机制](#viewresolver初始化机制)
    * [ViewResolver处理机制](#viewresolver处理机制)

# 源码分析ViewResolver

## 分析点

1. ModelAndView是啥
2. ViewResolver初始化机制
3. ViewResolver处理机制

## ModelAndView是啥

`ModelAndView`持有模型和视图（视图的名字）。


## ViewResolver初始化机制

和`HandlerMapping`的初始化类似，找所有实现`ViewResolver`接口的bean。


## ViewResolver处理机制

解析`ModelAndView`的调用链路如下：

`DispatcherServlet#doDispatch`
->
`DispatcherServlet#doDispatch#processDispatchResult`
->
`DispatcherServlet#render`

我们贴上源码分析`render`函数：

```java
protected void render(ModelAndView mv, HttpServletRequest request, HttpServletResponse response) throws Exception {
	// Determine locale for request and apply it to the response.
	Locale locale =
			(this.localeResolver != null ? this.localeResolver.resolveLocale(request) : request.getLocale());
	response.setLocale(locale);

	View view;
	String viewName = mv.getViewName();
	if (viewName != null) {
		// 解析得到视图，像我的环境里会用FreeMarker的解析器
		view = resolveViewName(viewName, mv.getModelInternal(), locale, request);
		if (view == null) {
			throw new ServletException("Could not resolve view with name '" + mv.getViewName() +
					"' in servlet with name '" + getServletName() + "'");
		}
	}
	else {
		// No need to lookup: the ModelAndView object contains the actual View object.
		view = mv.getView();
		if (view == null) {
			throw new ServletException("ModelAndView [" + mv + "] neither contains a view name nor a " +
					"View object in servlet with name '" + getServletName() + "'");
		}
	}

	// Delegate to the View object for rendering.
	if (logger.isTraceEnabled()) {
		logger.trace("Rendering view [" + view + "] ");
	}
	try {
		if (mv.getStatus() != null) {
			response.setStatus(mv.getStatus().value());
		}
		// 将视图反馈给浏览器 最终会到FreeMarkerView 进行相关操作然后传输给浏览器
		view.render(mv.getModelInternal(), request, response);
	}
	catch (Exception ex) {
		if (logger.isDebugEnabled()) {
			logger.debug("Error rendering view [" + view + "]", ex);
		}
		throw ex;
	}
}
```