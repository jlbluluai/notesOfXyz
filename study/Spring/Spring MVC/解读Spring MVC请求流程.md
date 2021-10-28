# Table of Contents

* [解读Spring MVC请求流程](#解读spring-mvc请求流程)
    * [开篇](#开篇)
    * [解读DispatcherServlet的工作](#解读dispatcherservlet的工作)
    * [解读HandlerMapping的工作](#解读handlermapping的工作)
    * [解读ViewResolver的工作](#解读viewresolver的工作)


# 解读Spring MVC请求流程

## 开篇

![](http://img.yelizi.top/c3d50f34-6852-44b3-95a0-7dfa82f19238.jpg$xyz)


简单叙述下这个过程：

1. 浏览器发送Http请求，被`DispatcherServlet`捕获；
2. `DispatcherServlet`对请求`URL`解析，得到`URI`（[URL和URI是啥?](https://www.zhihu.com/question/21950864)），并根据URI调用`HandlerMapping`获取该`Handler`所有相关的对象（`Handler`对象以及`Handler`对象对应的拦截器）；
3. 调用`Handler（Controller）`中相关的处理方法；
4. 执行`Handler`中，可能还会涉及调用相关的业务逻辑；
5. `Handler`执行完毕后，向`DispatcherServlet`返回一个`ModerlAndView`对象；
6. 根据`ModelAndView`选择合适的视图解析器`ViewResolver`返回给`DispatcherServlet`；
7. `ViewResolver`结合`Model`和`View`渲染视图；
8. 将渲染好的视图返回给浏览器。

总结一下，整个流程最重要的四点：**DispatcherServlet**，**HandlerMapping**，**ViewResolver**。


## 解读DispatcherServlet的工作

关注`DispatcherServlet`的工作主要就关注两方面：

1. 什么时候启动的？
2. 如何处理请求的？

[源码分析DispatcherServlet](https://github.com/jlbluluai/xyz-notes/blob/master/study/Spring/Spring%20MVC/%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90DispatcherServlet.md)


## 解读HandlerMapping的工作


关注`HandlerMapping`的工作主要就关注两方面：

1. 如何初始化的？
2. 如何处理的？

[源码分析HandlerMapping](https://github.com/jlbluluai/xyz-notes/blob/master/study/Spring/Spring%20MVC/%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90HandlerMapping.md)


## 解读ViewResolver的工作

传统的那种前后端未分离时，页面还是由服务端渲染好返回浏览器（比如Jsp），这种情况下就会得到`ModelAndView`并且就需要`ViewResolver`解析渲染了。

[源码分析DispatcherServlet](https://github.com/jlbluluai/xyz-notes/blob/master/study/Spring/Spring%20MVC/%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90ViewResolver.md)