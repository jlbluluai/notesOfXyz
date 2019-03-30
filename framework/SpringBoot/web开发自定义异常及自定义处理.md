参考：



### 前言

​	我们都知道web开发中若是服务器端抛出了异常，会报500错误码，但默认返回给请求端的异常信息有时候不是太能让人好懂的，所以我们需要自定义异常，并自定义处理反馈回前端的信息。



### Demo

​	废话不多说，先看我们自定义的异常。

```java
package com.xyz.exception;

public class UserNotExistException extends RuntimeException {

    private static final long serialVersionUID = 1662351321311659029L;

    private String id;

    public UserNotExistException(String id) {
        super("user not exist");
        this.id = id;
    }

    public String getId() {
        return id;
    }

    public void setId(String id) {
        this.id = id;
    }
}
```

​	该异常除了继承普通运行时异常，还自定义了id属性。使用上，我们模拟抛出下该异常。

```java
    @GetMapping(value = "/{id:\\d+}")
    @JsonView(User.UserDetailView.class)
    public User getInfo(@PathVariable String id) {
        throw new UserNotExistException(id);
    }
```

​	这时候我们发起请求，前端收到响应信息是这样的。

​	浏览器直接访问。

```
Whitelabel Error Page
This application has no explicit mapping for /error, so you are seeing this as a fallback.

Sat Mar 30 13:33:16 CST 2019
There was an unexpected error (type=Internal Server Error, status=500).
user not exist
```

​	通过Restlet Client插件模拟app访问。

```
{
"timestamp": 1553923938167,
"status": 500,
"error": "Internal Server Error",
"exception": "com.xyz.exception.UserNotExistException",
"message": "user not exist",
"path": "/user/1"
}
```

​	综合来说就是又臭又长，不容易找到异常原因还不能放入我们想要的额外信息，就感觉自定义了异常有啥用啊，这是因为我们毕竟从头到尾也确实没干涉过它到底反馈什么给前端吧。

​	要想自定义反馈，我们还得写一个异常类处理类。

```java
package com.xyz.web.controller;

import com.xyz.exception.UserNotExistException;
import org.springframework.http.HttpStatus;
import org.springframework.web.bind.annotation.ControllerAdvice;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.ResponseBody;
import org.springframework.web.bind.annotation.ResponseStatus;

import java.util.HashMap;
import java.util.Map;

@ControllerAdvice
public class ControllerExceptionHandler {

    @ExceptionHandler(UserNotExistException.class)
    @ResponseBody
    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    public Map<String,Object> handleUserNotExistException(UserNotExistException ex){
        Map<String,Object> result = new HashMap<>();
        result.put("id",ex.getId());
        result.put("message",ex.getMessage());
        return result;
    }
}
```

​	像这样我们看代码含义结果只包含id和message，实际上我们再访问下，不论是浏览器端还是app端都会收到这样的回馈。

```
{
"id": "1",
"message": "user not exist"
}
```

​	因为默认的时候Spring MVC会判断浏览器端和app端，但这边我们自定义后并未判断，所以若是抛出的异常是执行这边我们自定义的处理方法时自然不论请求端是啥都给一样的结果了。