参考：



### 前言

​	以前初学Spring Boot的时候肯定是接触过本篇将要讲的的内容的，拦截无非就是过滤器和拦截器，当然本篇还会多涉及一个新的拦截方式——利用切片。本篇就将围绕这三者详细整理出在Spring Boot中，我们如何去拦截REST服务。



### 过滤器（Fiiter）

​	如其名，它就像一个滤网似的介于客户端和服务端之间，任何的请求（根据配置需要拦截的请求地址）进来都会现在过滤器兜一圈，然后再进服务器（可能还由过滤器加了附加的东西）或者就被pass掉了。

​	那么，言归正题，在Spring Boot中我们怎么实现一个过滤器呢，以前在普通的web工程中，我们是有一个web.xml文件的，还有印象的小伙伴应该记得我们是在web.xml中配置过滤器的，设置过滤url，还可以统一设置编码。那么在Spring Boot中，我们显然是没有web.xml的，这里就需要Java代码来大显身手了。

​	首先javax.servlet提供了一个统一Filter接口，其中就三个方法，大家应该都猜得到，过滤器生命周期三步骤：初始化、处理、销毁，这三个方法就对应这个生命周期，那么自己写的过滤器也就是这样了。

```java
package com.xyz.web.filter;

import javax.servlet.*;
import java.io.IOException;

@Component//过滤器生效的一种方式，不过这样是默认所有请求都要过滤的
//@WebFilter(urlPatterns = "/*")//过滤器生效的第二种方式，阔以指定过滤路径
public class TimeFilter implements Filter {
    //初始化，项目启动时就会执行
    @Override
    public void init(FilterConfig filterConfig) throws ServletException {
        System.out.println("time filter init...");
    }

    //执行，做相关拦截的操作
    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) throws IOException, ServletException {
        System.out.println("time filter start...");
        long start = System.currentTimeMillis();
        chain.doFilter(request, response);
        System.out.println("time filter:" + (System.currentTimeMillis() - start));
        System.out.println("time filter finish...");
    }

    //销毁
    @Override
    public void destroy() {
        System.out.println("time filter destory...");
    }
}

```

​	当然，代码注释处我也提到了，若是通过@Component注解注册过滤器，只能过滤所有请求，没法控制。针对这个对类注解处我还注释了一个@WebFilter，用它就可以指定过滤请求。这样的话，还得给启动类加一个注解注册过滤器。

```java
@ServletComponentScan(basePackages= {"com.xyz.web.filter"})
```

​	当然还有一个问题，我们项目集成了Spring Boot，所以直接用spring的注解标识类，我们引入的很多第三方jar包也提供了过滤器，我们想让他们生效，显然不能通过这种方式，所以还可以通过另外一种方式注册，通过配置类，不过这边@Component注解就得注释了。

```java
package com.xyz.web.config;

import com.xyz.web.filter.TimeFilter;
import org.springframework.boot.web.servlet.FilterRegistrationBean;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurerAdapter;

import java.util.ArrayList;
import java.util.List;

@Configuration
public class WebConfig {

    @Bean
    public FilterRegistrationBean timeFilter(){
        FilterRegistrationBean filterRegistrationBean = new FilterRegistrationBean();
        TimeFilter timeFilter = new TimeFilter();
        filterRegistrationBean.setFilter(timeFilter);//注入自己的过滤器，第三方过滤器
        List<String> urls = new ArrayList<>();
        urls.add("/*");//定义要过滤的请求路径，/*就是所有，有特殊需求自己看情况
        filterRegistrationBean.setUrlPatterns(urls);
        return filterRegistrationBean;
    }
}

```

​	这样的话，我们就技能控制请求路径，也能够注册第三方jar包中的过滤器了。



### 拦截器（Interceptor）

​	这个就是见名知意的拦截了，不同于过滤器，过滤器是在请求进来的时候就要拦截的，是javax.servlet提供的，和spring没有关系，所以并不会知道请求最后具体有哪个控制器哪个方法执行。而拦截器是spring提供的，它是真正在请求分配到哪个方法要开始执行才执行的，也分为三个步骤：方法执行前，方法提交时，方法完成后。

​	我们要自己写拦截器的话，得实现一个spring提供的接口HandlerInterceptor。

```java
package com.xyz.web.interceptor;

import org.springframework.stereotype.Component;
import org.springframework.web.method.HandlerMethod;
import org.springframework.web.servlet.HandlerInterceptor;
import org.springframework.web.servlet.ModelAndView;

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;

@Component
public class TimeInterceptor implements HandlerInterceptor {
    //方法执行前
    @Override
    public boolean preHandle(HttpServletRequest httpServletRequest, HttpServletResponse httpServletResponse, Object o) throws Exception {
        System.out.println("prehandle");
        System.out.println(((HandlerMethod) o).getBean().getClass().getName());
        System.out.println(((HandlerMethod) o).getMethod().getName());
        httpServletRequest.setAttribute("startTime", System.currentTimeMillis());
        return true;
    }

    //方法提交时，若方法执行异常这步不会执行
    @Override
    public void postHandle(HttpServletRequest httpServletRequest, HttpServletResponse httpServletResponse, Object o, ModelAndView modelAndView) throws Exception {
        System.out.println("posthandle");
        Long start = (Long) httpServletRequest.getAttribute("startTime");
        System.out.println("time interceptor 耗时:" + (System.currentTimeMillis() - start));
    }

    //方法完成后，这步是方法无论异常与否都必然执行
    @Override
    public void afterCompletion(HttpServletRequest httpServletRequest, HttpServletResponse httpServletResponse, Object o, Exception e) throws Exception {
        System.out.println("afterCompletion");
        Long start = (Long) httpServletRequest.getAttribute("startTime");
        System.out.println("time interceptor 耗时:" + (System.currentTimeMillis() - start));
        System.out.println(e);
    }
}

```

​	当然，拦截器和过滤器不一样，这里用@Component注册一下只是简单的让拦截器类成为了Spring管理的bean，它还不能称作为一个起作用的拦截器，我们还需要在配置类中注册拦截器（还记得小伙伴应该记得以前普通web项目中我们会写一个spring mvc的配置xml，那时是在那里面注册的）。

```java
package com.xyz.web.config;

import com.xyz.web.interceptor.TimeInterceptor;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.config.annotation.InterceptorRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurerAdapter;

@Configuration
public class WebConfig extends WebMvcConfigurerAdapter {

    @Autowired
    private TimeInterceptor timeInterceptor;

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(timeInterceptor);
    }
    
}

```

​	这样的话也是默认所有请求都拦截的，若想指定可以类似这样的用法。

```java
registry.addInterceptor(timeInterceptor).addPathPatterns(参数：存放拦截路径的字符串数组).excludePathPatterns(参数：存放不拦截路径的字符串数组);
```

​	像这样我们就能实现一些权限控制，当然这样的权限控制就有点原始了，Spring Security是Spring官方提供一款安全性框架，其中便有完备的权限控制功能，有兴趣的小伙伴可以去学一学。



### 切片（AOP）

