# 本手册专注于Java中校验相关

## javax提供校验注解

| 注解                        | 作用                                                     |
| :-------------------------- | :------------------------------------------------------- |
| @Valid                      | 被注释的元素是一个对象，需要检查此对象的所有字段值       |
| @Null                       | 被注释的元素必须为 null                                  |
| @NotNull                    | 被注释的元素必须不为 null                                |
| @AssertTrue                 | 被注释的元素必须为 true                                  |
| @AssertFalse                | 被注释的元素必须为 false                                 |
| @Min(value)                 | 被注释的元素必须是一个数字，其值必须大于等于指定的最小值 |
| @Max(value)                 | 被注释的元素必须是一个数字，其值必须小于等于指定的最大值 |
| @DecimalMin(value)          | 被注释的元素必须是一个数字，其值必须大于等于指定的最小值 |
| @DecimalMax(value)          | 被注释的元素必须是一个数字，其值必须小于等于指定的最大值 |
| @Size(max, min)             | 被注释的元素的大小必须在指定的范围内                     |
| @Digits (integer, fraction) | 被注释的元素必须是一个数字，其值必须在可接受的范围内     |
| @Past                       | 被注释的元素必须是一个过去的日期                         |
| @Future                     | 被注释的元素必须是一个将来的日期                         |
| @Pattern(value)             | 被注释的元素必须符合指定的正则表达式                     |



## Hibernate 提供校验注解

 

| 注解                                                 | 作用                                                         |
| ---------------------------------------------------- | ------------------------------------------------------------ |
| @Email                                               | 被注释的元素必须是电子邮箱地址                               |
| @Length(min=, max=)                                  | 被注释的字符串的大小必须在指定的范围内                       |
| @NotEmpty                                            | 被注释的字符串的必须非空                                     |
| @Range(min=, max=)                                   | 被注释的元素必须在合适的范围内                               |
| @NotBlank                                            | 被注释的字符串的必须非空，字符串不为null，并且字符串trim()以后length要大于0 |
| @URL(protocol=,  host=,    port=,   regexp=, flags=) | 被注释的字符串必须是一个有效的url                            |



## 自定义校验注解

定义一个注解，最基本的写法：

```java
@Target({ElementType.METHOD,ElementType.FIELD})
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = MyConstraintValidator.class)
public @interface MyConstraint {
    String message();

    Class<?>[] groups() default { };

    Class<? extends Payload>[] payload() default { };
}
```

处理逻辑类，该类实现ConstraintValidator，Spring会识别注册为bean，所以可以注入其它bean，isValid方法返回true则通过校验，false则未通过，就会返回注解使用中定义的message。

```java
public class MyConstraintValidator implements ConstraintValidator<MyConstraint,Object> {

    @Autowired
    private HelloService helloService;

    @Override
    public void initialize(MyConstraint constraintAnnotation) {
        System.out.println("my validator init...");
    }

    @Override
    public boolean isValid(Object value, ConstraintValidatorContext context) {
        helloService.greeting("tom");
        System.out.println(value);
        return false;
    }
}
```