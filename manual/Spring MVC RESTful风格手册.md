# 本手册专注于Spring MVC中RESTful风格的使用相关

## 最简单的理解方式

传统的url

```
/user?id=1&username=Jack
```

RESTful风格url

```
/user/1/Jack
```



## 常用注解介绍

### @RestController

该注解对应@Controller也是加在类上，但有一层意思，标志这个Controller是一个RESTful风格的Controller，使用上的表现就是省去了每个方法都加一个@ResponseBody，换句话说，每个方法的返回就会转换为json格式数据。

例子：

```java
@RestController
public class UserController {
```

### @RequestBody

该注解用来方法传入参数上，Spring将自动绑定HTTP请求体到参数上。

例子：

```java
public User create(@Valid @RequestBody User user, BindingResult errors) {
```

### @ResponseBody

该注解用于方法，正如@RestController中所说，类已经用了@RestController方法就不需要再单独标志@ResponseBody，效果一样。

### @PathVariable

该注解用于方法参数，将请求url的模板参数绑定。

例子：

```java
@DeleteMapping(value = "/{id:\\d+}")
public void delete(@PathVariable String id) {
```

### @RequestMapping以及衍生

该注解可用于Controller，直接规定了该类统一处理的url映射。

例子：

```java
@RestController
@RequestMapping("/user")
public class UserController {
```

主要还是用于方法，规定了方法处理的url映射。

例子：

```java
@RequestMapping(value = "/aaa",method = RequestMethod.GET)
```

对于方法注解衍生出四个子注解（替代每次写method的麻烦），四者除了不需要再写method，其它与@RequestMapping用法一致。

#### @GetMapping

#### @PostMapping

#### @PutMapping

#### @DeleteMapping



## REST API示例

- GET 方式请求 /user 返回用户列表
- GET 方式请求 /user/1返回id为1的用户
- POST 方式请求 /user 通过user对象的JSON 参数创建新的user对象
- PUT 方式请求 /user/1 更新id为1的发送json格式的用户对象 
- DELETE 方式请求/user/1删除id为1的user对象
- DELETE 方式请求/user删除所有user（这个应该用不到吧，哈哈）

