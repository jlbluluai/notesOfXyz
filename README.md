# Table of Contents

* [xyz-notes](#xyz-notes)
  * [study](#study)
    * [Java](#java)
      * [base](#base)
      * [jvm](#jvm)
      * [concurrent](#concurrent)
      * [io](#io)
    * [Spring](#spring)
      * [Spring](#spring-1)
      * [Spring MVC](#spring-mvc)
      * [expand](#expand)
    * [MyBatis](#mybatis)
    * [algorithm](#algorithm)
    * [MySQL](#mysql)
    * [Redis](#redis)
    * [Queue](#queue)


# xyz-notes

## study

### Java

#### base

- [包装类型赋值解读](https://github.com/jlbluluai/xyz-notes/blob/master/study/Java/base/%E5%8C%85%E8%A3%85%E7%B1%BB%E5%9E%8B%E8%B5%8B%E5%80%BC%E8%A7%A3%E8%AF%BB.md)
- [参数传递-值传递？引用传递？](https://github.com/jlbluluai/xyz-notes/blob/master/study/Java/base/%E5%8F%82%E6%95%B0%E4%BC%A0%E9%80%92-%E5%80%BC%E4%BC%A0%E9%80%92%EF%BC%9F%E5%BC%95%E7%94%A8%E4%BC%A0%E9%80%92%EF%BC%9F.md)
- [枚举解读](https://github.com/jlbluluai/xyz-notes/blob/master/study/Java/base/%E6%9E%9A%E4%B8%BE%E8%A7%A3%E8%AF%BB.md)
- [泛型解读](https://github.com/jlbluluai/xyz-notes/blob/master/study/Java/base/%E6%B3%9B%E5%9E%8B%E8%A7%A3%E8%AF%BB.md)
- [String解读](https://github.com/jlbluluai/xyz-notes/blob/master/study/Java/base/String解读.md)

#### jvm

- [字节码解读](https://github.com/jlbluluai/xyz-notes/blob/master/study/Java/JVM/%E5%AD%97%E8%8A%82%E7%A0%81%E8%A7%A3%E8%AF%BB.md)
- [Java内存模型解读](https://github.com/jlbluluai/xyz-notes/blob/master/study/Java/JVM/Java%E5%86%85%E5%AD%98%E6%A8%A1%E5%9E%8B%E8%A7%A3%E8%AF%BB.md)
- GC解读
  - [GC解读-宽泛概念](https://github.com/jlbluluai/xyz-notes/blob/master/study/Java/JVM/GC%E8%A7%A3%E8%AF%BB-%E5%AE%BD%E6%B3%9B%E6%A6%82%E5%BF%B5.md)
  - [GC解读-具体实现](https://github.com/jlbluluai/xyz-notes/blob/master/study/Java/JVM/GC%E8%A7%A3%E8%AF%BB-%E5%85%B7%E4%BD%93%E5%AE%9E%E7%8E%B0.md)

#### concurrent

- [并发之 volatile 解读](https://github.com/jlbluluai/xyz-notes/blob/master/study/Java/concurrent/%E5%B9%B6%E5%8F%91%E4%B9%8B%20volatile%20%E8%A7%A3%E8%AF%BB.md)
- [并发之 synchronized 解读](https://github.com/jlbluluai/xyz-notes/blob/master/study/Java/concurrent/%E5%B9%B6%E5%8F%91%E4%B9%8B%20synchronized%20%E8%A7%A3%E8%AF%BB.md)
- [并发之 CAS 解读](https://github.com/jlbluluai/xyz-notes/blob/master/study/Java/concurrent/%E5%B9%B6%E5%8F%91%E4%B9%8B%20CAS%20%E8%A7%A3%E8%AF%BB.md)
- [显示锁之Lock解读](https://github.com/jlbluluai/xyz-notes/blob/master/study/Java/concurrent/%E6%98%BE%E7%A4%BA%E9%94%81%E4%B9%8BLock%E8%A7%A3%E8%AF%BB.md)

#### io

- [装饰器模式在Java IO的运用](https://github.com/jlbluluai/xyz-notes/blob/master/study/Java/io/%E8%A3%85%E9%A5%B0%E5%99%A8%E6%A8%A1%E5%BC%8F%E5%9C%A8Java%20IO%E7%9A%84%E8%BF%90%E7%94%A8.md)

### Spring

#### Spring

- [解读Spring的启动流程-IoC容器的创建](https://github.com/jlbluluai/xyz-notes/blob/master/study/Spring/Spring/解读Spring的启动流程-IoC容器的创建.md)
- [解读Spring的bean创建](https://github.com/jlbluluai/xyz-notes/blob/master/study/Spring/Spring/解读Spring的bean创建.md)
  - [解读Spring的bean创建时初始化函数的执行顺序](https://github.com/jlbluluai/xyz-notes/blob/master/study/Spring/Spring/解读Spring的bean创建时初始化函数的执行顺序.md)
- [解读Spring中AOP如何生效](https://github.com/jlbluluai/xyz-notes/blob/master/study/Spring/Spring/解读Spring中AOP如何生效.md)

#### Spring MVC

- [解读Spring MVC请求流程](https://github.com/jlbluluai/xyz-notes/blob/master/study/Spring/Spring%20MVC/%E8%A7%A3%E8%AF%BBSpring%20MVC%E8%AF%B7%E6%B1%82%E6%B5%81%E7%A8%8B.md)
  - [源码分析DispatcherServlet](https://github.com/jlbluluai/xyz-notes/blob/master/study/Spring/Spring%20MVC/%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90DispatcherServlet.md)
  - [源码分析HandlerMapping](https://github.com/jlbluluai/xyz-notes/blob/master/study/Spring/Spring%20MVC/%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90HandlerMapping.md)
    - [源码分析handler的执行](https://github.com/jlbluluai/xyz-notes/blob/master/study/Spring/Spring%20MVC/%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90handler%E7%9A%84%E6%89%A7%E8%A1%8C.md)
  - [源码分析ViewResolver](https://github.com/jlbluluai/xyz-notes/blob/master/study/Spring/Spring%20MVC/%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90ViewResolver.md)

#### expand

- [动态注入Bean并可正常注入使用](https://github.com/jlbluluai/xyz-notes/blob/master/study/Spring/expand/动态注入Bean并可正常注入使用.md)


### MyBatis

- [解读MyBatis的实现原理](https://github.com/jlbluluai/xyz-notes/blob/master/study/MyBatis/解读MyBatis的实现原理.md)
- [解读MyBatis中Mapper接口和xml文件如何对应](https://github.com/jlbluluai/xyz-notes/blob/master/study/MyBatis/解读MyBatis中Mapper接口和xml文件如何对应.md)

### algorithm

- [动态规划入门](https://github.com/jlbluluai/notesOfXyz/blob/master/study/algorithm/routine/%E5%8A%A8%E6%80%81%E8%A7%84%E5%88%92%E5%85%A5%E9%97%A8.md)
- [排序统合](https://github.com/jlbluluai/xyz-notes/blob/master/study/algorithm/routine/%E6%8E%92%E5%BA%8F%E7%BB%9F%E5%90%88.md)
- [滑动窗口入门（解决子串问题的制胜法宝）](https://github.com/jlbluluai/xyz-notes/blob/master/study/algorithm/routine/滑动窗口入门（解决子串问题的制胜法宝）.md)
- [BFS算法入门](https://github.com/jlbluluai/xyz-notes/blob/master/study/algorithm/routine/BFS算法入门.md)
- [回溯入门](https://github.com/jlbluluai/xyz-notes/blob/master/study/algorithm/routine/回溯入门.md)

### MySQL

- [MySQL之 字符集和比较规则](https://github.com/jlbluluai/xyz-notes/blob/master/study/MySQL/MySQL%E4%B9%8B%20%E5%AD%97%E7%AC%A6%E9%9B%86%E5%92%8C%E6%AF%94%E8%BE%83%E8%A7%84%E5%88%99.md)
- [MySQL的执行流程分析](https://github.com/jlbluluai/xyz-notes/blob/master/study/MySQL/MySQL%E7%9A%84%E6%89%A7%E8%A1%8C%E6%B5%81%E7%A8%8B%E5%88%86%E6%9E%90.md)
- [MySQL索引原理解读](https://github.com/jlbluluai/xyz-notes/blob/master/study/MySQL/MySQL索引原理解读.md)
- [MySQL锁解读](https://github.com/jlbluluai/xyz-notes/blob/master/study/MySQL/MySQL锁解读.md)


### Redis

- [Redis之 基础介绍](https://github.com/jlbluluai/xyz-notes/blob/master/study/Redis/Redis%E4%B9%8B%20%E5%9F%BA%E7%A1%80%E4%BB%8B%E7%BB%8D.md)
- [Redis三大经典问题分析](https://github.com/jlbluluai/xyz-notes/blob/master/study/Redis/Redis三大经典问题分析.md)
- [Redis搭建主从、sentinel、cluster](https://github.com/jlbluluai/xyz-notes/blob/master/study/Redis/Redis搭建主从、sentinel、cluster.md)


### Queue

...