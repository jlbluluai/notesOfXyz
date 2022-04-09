# Table of Contents

* [Java单元测试的简单实践手册](#java单元测试的简单实践手册)
    * [自测的重要性](#自测的重要性)
    * [junit的简单实践](#junit的简单实践)
    * [junit的mock简单运用](#junit的mock简单运用)


# Java单元测试的简单实践手册

## 自测的重要性

程序员并不是说把代码写完看似编译通过就是干完活了，代码在运行时是否有各种异常或者不合理的逻辑是要靠测试出来的，但如果说写完代码一堆问题就交付给测试这是极度不负责的，所以要有一个自测环节。

而自测中最常见的测试方法就是单元测试，本篇即围绕Java的单元测试做一个简单的实践手册。


## junit的简单实践

junit是Java中最常见的单元测试手段了，这里假设项目是SpringBoot搭建的，在pom中引入如下依赖就基本涵盖了需要用到的jar。

```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
</dependency>
```

实践代码：

```
import org.junit.*;
import org.junit.runner.RunWith;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.junit4.SpringJUnit4ClassRunner;

@RunWith(SpringJUnit4ClassRunner.class)
@SpringBootTest
public class MyTest {


    /**
     * 每个测试方法执行前都会执行一遍
     */
    @Before
    public void before() {
        System.out.println("before");
    }

    @Test
    public void testA() {
        System.out.println("测试a");

        // 断言：注意这里用org.junit下的Assert，spring框架中也有（当然你也可以用那个，注意只是为了不要在测试代码过分依赖其他的类）
        Assert.assertNotNull(null, "测试断言");
    }

    @Test
    public void testB() {
        System.out.println("测试b");
    }

    /**
     * @Ignore 注解的方法执行时将会被忽略
     */
    @Ignore
    @Test
    public void test() {
        System.out.println("测试c");
    }

    /**
     * 每个测试方法执行后都会执行一遍（无论是否异常）
     */
    @After
    public void after() {
        System.out.println("after");
    }

}
```


## junit的mock简单运用

在涉及要模拟调用暴露的api时比如SpringMVC的的接口时，可能我们会通过postman这样的软件去配置调用，当然，在spring test中也提供了mock能够模拟这一套流程。

**简单模拟代码**

```
/**
 * 模拟的controller
 */
@RestController
@RequestMapping("/test")
public class TestController {

    @GetMapping(value = "/aa")
    public void aa() {
        System.out.println(11);
    }

}

import org.junit.Assert;
import org.junit.Before;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.http.MediaType;
import org.springframework.test.context.junit4.SpringJUnit4ClassRunner;
import org.springframework.test.web.servlet.MockMvc;
import org.springframework.test.web.servlet.MvcResult;
import org.springframework.test.web.servlet.request.MockHttpServletRequestBuilder;
import org.springframework.test.web.servlet.request.MockMvcRequestBuilders;
import org.springframework.test.web.servlet.setup.MockMvcBuilders;
import org.springframework.util.LinkedMultiValueMap;
import org.springframework.util.MultiValueMap;
import org.springframework.web.context.WebApplicationContext;

import javax.annotation.Resource;
import javax.servlet.http.Cookie;
import java.util.Objects;

import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;

@RunWith(SpringJUnit4ClassRunner.class)
@SpringBootTest
public class MockTest {

    @Resource
    private WebApplicationContext webApplicationContext;

    private MockMvc mvc;

    private Cookie cookie;

    @Before
    public void setUp() throws Exception {
        // 如有cookie需求设置
        cookie = new Cookie("a", "1");
        mvc = MockMvcBuilders.webAppContextSetup(webApplicationContext)
                .build();
    }

    /**
     * 生成一般json接口调用体并取得结果
     */
    private String genResponse(String url, boolean isGet, MultiValueMap<String, String> params, String body) throws Exception {
        MockHttpServletRequestBuilder builder;
        if (isGet) {
            builder = MockMvcRequestBuilders.get(url);
        } else {
            builder = MockMvcRequestBuilders.post(url);
        }
        builder.cookie(cookie)
                .contentType(MediaType.APPLICATION_JSON_UTF8_VALUE)
                .accept(MediaType.APPLICATION_JSON);
        if (Objects.nonNull(params)) {
            builder.params(params);
        }
        if (Objects.nonNull(body)) {
            builder.content(body);
        }

        MvcResult mvcResult = mvc.perform(builder)
                .andExpect(status().isOk())
                .andReturn();

        return mvcResult.getResponse().getContentAsString();
    }

    @Test
    public void testMock() throws Exception {
        MultiValueMap<String, String> params = new LinkedMultiValueMap<>();
        params.add("token","d03c034d-b209-4d93-a0eb-a5413a123259");

        String data = genResponse("/test/aa", true, params, null);
        Assert.assertNotNull(data);
    }

}
```