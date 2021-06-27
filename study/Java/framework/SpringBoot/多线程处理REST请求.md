参考：



### 前言

​	有没有想过一个问题，Spring MVC接收请求然后分配处理肯定是要分配线程的，我们姑且称作主线程组，这个数量肯定是有限的，要是有个服务一时间请求人数过多，而且这个服务处理时间还长，那大量的主线程就会被占用用来处理这个，性能大大下降，所以我们给服务个体再开启一个线程去处理服务，主线程只负责接收请求再分发，不是就能提高服务处理性能了吗。



### Callable

​	如果只有一台机子，简单的考虑下，我们可以在接收到请求并分配ok后立马将任务指派给子线程去做，主线程指派完就结束了，又可以空出来去接收下一个请求。

```java
@RestController
public class AsyncController {
    private Logger logger = LoggerFactory.getLogger(getClass());

    @GetMapping("/order")
    public Callable<String> order() throws Exception {
        logger.info("主线程开始");
        Callable<String> result = () -> {
            logger.info("副线程开始");
            Thread.sleep(1000);
            logger.info("副线程结束");
            return "success";
        };
        logger.info("主线程返回");
        return result;
    }
}
```

​	访问后的日志结果是这样的。

```java
2019-03-31 10:36:45.101  INFO 14248 --- [nio-8080-exec-2] com.xyz.web.async.AsyncController        : 主线程开始
2019-03-31 10:36:45.102  INFO 14248 --- [nio-8080-exec-2] com.xyz.web.async.AsyncController        : 主线程返回
2019-03-31 10:36:45.111  INFO 14248 --- [      MvcAsync1] com.xyz.web.async.AsyncController        : 副线程开始
2019-03-31 10:36:46.112  INFO 14248 --- [      MvcAsync1] com.xyz.web.async.AsyncController        : 副线程结束
2019-03-31 10:37:42.868  INFO 14248 --- [nio-8080-exec-9] com.xyz.web.async.AsyncController        : 主线程开始
2019-03-31 10:37:42.868  INFO 14248 --- [nio-8080-exec-9] com.xyz.web.async.AsyncController        : 主线程返回
2019-03-31 10:37:42.871  INFO 14248 --- [      MvcAsync2] com.xyz.web.async.AsyncController        : 副线程开始
2019-03-31 10:37:43.872  INFO 14248 --- [      MvcAsync2] com.xyz.web.async.AsyncController        : 副线程结束
```

​	我一共发送了两次请求，我们可以看到每次主线程都是可以提前下班的，然后随便副线程处理多久，这样就能大大提高性能了。



### DeferredResult

​	如果是以前的单系统集群时代，那肯定Callable的解决方案就行了（也没办法玩花里胡哨的不是），但是分布式系统的当代肯定是不行的，服务的真正处理往往已经在另外一台系统上了，并且现在还引入了消息队列的技术，系统与系统间的单线交互变成了，不断塞任务进消息队列和不断从消息队列读任务处理这样的状态。

​	这里引出了DeferredResult的技术运用，异步请求处理。A系统1线程接收请求，塞入消息队列，B系统1线程拿到处理任务，处理完塞入消息队列，A系统2线程从消息队列取出来反馈给客户端，看似没毛病，但是A系统2线程又如何去找到客户端，DeferredResult就是为了这个而存在的，A系统1线程知道客户端，DeferredResult记住，任务处理完A系统2线程直接用DeferredResult的setResult即可将相应的反馈信息给客户端了。

​	这样一个异步请求处理就完美的在几个系统几个线程的协助下完成了，当然我们这里不可能还搞出几给系统和消息队列了，这里简单写个类模拟消息队列，直接通知即可。

​	模拟消息队列

```java
package com.xyz.web.async;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Component;

@Component
public class MockQueue {

    private String placeOrder;

    private String completeOrder;

    private Logger logger = LoggerFactory.getLogger(getClass());

    public String getPlaceOrder() {
        return placeOrder;
    }

    public void setPlaceOrder(String placeOrder) throws InterruptedException {
       new Thread(()->{
           logger.info("接到下单请求：" + placeOrder);
           try {
               Thread.sleep(1000);
           } catch (InterruptedException e) {
               e.printStackTrace();
           }
           this.completeOrder = placeOrder;
           logger.info("下单请求处理完毕");
       }).start();

    }

    public String getCompleteOrder() {
        return completeOrder;
    }

    public void setCompleteOrder(String completeOrder) {
        this.completeOrder = completeOrder;
    }
}
```

​	DeferredResult持有类

```java
package com.xyz.web.async;

import org.springframework.stereotype.Component;
import org.springframework.web.context.request.async.DeferredResult;

import java.util.HashMap;
import java.util.Map;

@Component
public class DeferredResultHolder {

    private Map<String, DeferredResult<String>> map = new HashMap<>();

    public Map<String, DeferredResult<String>> getMap() {
        return map;
    }

    public void setMap(Map<String, DeferredResult<String>> map) {
        this.map = map;
    }
}
```

​	监听类（模拟A系统线程2）

```java
package com.xyz.web.async;

import org.apache.commons.lang.StringUtils;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.ApplicationListener;
import org.springframework.context.event.ContextRefreshedEvent;
import org.springframework.stereotype.Component;

@Component
public class QueueListener implements ApplicationListener<ContextRefreshedEvent> {

    @Autowired
    private MockQueue mockQueue;

    @Autowired
    private DeferredResultHolder deferredResultHolder;

    private Logger logger = LoggerFactory.getLogger(getClass());

    @Override
    public void onApplicationEvent(ContextRefreshedEvent event) {
        new Thread(() -> {
            while (true) {
                if (StringUtils.isNotBlank(mockQueue.getCompleteOrder())) {
                    String orderNum = mockQueue.getCompleteOrder();
                    logger.info("处理结果：" + orderNum);
                    deferredResultHolder.getMap().get(orderNum).setResult("place order success");
                    mockQueue.setCompleteOrder(null);
                } else {
                    try {
                        Thread.sleep(1000);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            }
        }).start();
    }
}
```

​	控制器类

```java
package com.xyz.web.async;

import org.apache.commons.lang.RandomStringUtils;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;
import org.springframework.web.context.request.async.DeferredResult;

import java.util.concurrent.Callable;

@RestController
public class AsyncController {
    private Logger logger = LoggerFactory.getLogger(getClass());

    @Autowired
    private MockQueue mockQueue;

    @Autowired
    private DeferredResultHolder deferredResultHolder;

    @GetMapping("/order2")
    public DeferredResult<String> order2() throws Exception {
        logger.info("主线程开始");

        String orderNumber = RandomStringUtils.randomNumeric(8);
        mockQueue.setPlaceOrder(orderNumber);

        DeferredResult<String> result = new DeferredResult<>();
        deferredResultHolder.getMap().put(orderNumber,result);

        logger.info("主线程返回");
        return result;
    }


}
```

​	我们访问后日志是这样的。

```java
2019-03-31 10:52:25.835  INFO 14248 --- [nio-8080-exec-7] com.xyz.web.async.AsyncController        : 主线程开始
2019-03-31 10:52:25.841  INFO 14248 --- [      Thread-33] com.xyz.web.async.MockQueue              : 接到下单请求：82445150
2019-03-31 10:52:25.841  INFO 14248 --- [nio-8080-exec-7] com.xyz.web.async.AsyncController        : 主线程返回
2019-03-31 10:52:26.843  INFO 14248 --- [      Thread-33] com.xyz.web.async.MockQueue              : 下单请求处理完毕
2019-03-31 10:52:26.993  INFO 14248 --- [      Thread-21] com.xyz.web.async.QueueListener          : 处理结果：82445150

```

​	很完美的模拟了叙述的那个异步的过程，哈哈。