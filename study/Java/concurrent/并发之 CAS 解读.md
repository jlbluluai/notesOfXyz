# Table of Contents

* [并发之 CAS 解读](#并发之-cas-解读)
    * [介绍](#介绍)
    * [原子类介绍](#原子类介绍)
    * [原子类的问题](#原子类的问题)
        * [高并发下的性能浪费](#高并发下的性能浪费)
        * [ABA问题](#aba问题)
    * [总结](#总结)
    * [参考](#参考)


# 并发之 CAS 解读


## 介绍

> 比较并交换(compare and swap, CAS)，是原子操作的一种，可用于在多线程编程中实现不被打断的数据交换操作，从而避免多线程同时改写某一数据时由于执行顺序不确定性以及中断的不可预知性产生的数据不一致问题。 该操作通过将内存中的值与指定数据进行比较，当数值一样时将内存中的数据替换为新的值。 -- 维基百科

用代码来描述这个CAS过程如下：

```c
int cas(long *addr, long old, long new)
{
    if(*addr != old)
        return 0;
    *addr = new;
    return 1;
}
```

但千万不要认为CAS是这样实现的，CAS既然是原子指令，那就是单独的一条指令才称为原子指令，这是现代CPU提供的内置指令（所以也不要纠结底层原理了）。无论哪个上层程序语言最多通过互斥锁模拟这个原子过程，绝不能称之为原子指令。

因其特性，上层程序语言经常拿它配合自旋来减轻甚至避免互斥带来的线程阻塞、唤醒开销，比如Java中的 `synchronized`（偏向锁、轻量级锁）、`AQS`、原子类。

`synchronized` 和 `AQS` 这里不细说（单独开篇讲，而且他俩本质就是互斥锁，只是引入CAS做了一定的优化），而原子类可以说是真的无锁的锁方案实现（有点绕），本篇就围绕它展开讲解。


## 原子类介绍

Java提供了一系列以 `Atomic` 为前缀的类，他们底层就是运用CAS指令保证了共享变量的并发安全性。

我们以 `AtomicInteger` 为例继续讲解，先放出案例代码：


```java
public class AtomicTest {

    public static void incr(AtomicInteger shared) {
        shared.incrementAndGet();
    }

    public static void main(String[] args) throws Exception {
        AtomicInteger shared = new AtomicInteger();
        ExecutorService taskExecutor = Executors.newFixedThreadPool(3);

        CyclicBarrier cyclicBarrier = new CyclicBarrier(4);

        taskExecutor.execute(() -> {
            try {
                cyclicBarrier.await();
            } catch (Exception e) {
                e.printStackTrace();
            }
            System.out.println("线程1开始");
            for (int i = 0; i < 1000; i++) {
                incr(shared);
            }
        });
        taskExecutor.execute(() -> {
            try {
                cyclicBarrier.await();
            } catch (Exception e) {
                e.printStackTrace();
            }
            System.out.println("线程2开始");
            for (int i = 0; i < 1000; i++) {
                incr(shared);
            }
        });
        taskExecutor.execute(() -> {
            try {
                cyclicBarrier.await();
            } catch (Exception e) {
                e.printStackTrace();
            }
            System.out.println("线程3开始");
            for (int i = 0; i < 1000; i++) {
                incr(shared);
            }
        });

        Thread.sleep(1000);
        System.out.println("证明shared值还未被计算：" + shared.get());
        cyclicBarrier.await();

        Thread.sleep(5000);
        System.out.println("最后shared值：" + shared.get());
    }

}
```

描述下案例：

1. 定义一个 `AtomicInteger` 对象，默认初始值就是0；
2. 启动三个线程，逻辑都是循环100次对 `AtomicInteger` 对象执行 +1；
3. 为了排除线程先后启动的干扰，我这里特地用了 `CyclicBarrier`，不了解的可以去搜下
    - 效果就是使得异步的代码到达同步点后会先暂停等待大家都就位，然后再继续向下执行
    - 这里就简单的认为这样就保证了三个线程中的for循环是并发执行的
4. 结果不出意外的话就是3000

这就是CAS的奥妙，这段案例代码我们并没有加锁，区别普通的代码可能就是 `int shared = 0;` 换成了 `AtomicInteger shared = new AtomicInteger();`

我这里用一张图来简单描述下赋值过程：

![](http://img.yelizi.top/3afb4cb0-0aa8-4682-8d05-2d4b37a0fc21.jpg$xyz)

也就是在不断的自旋做CAS操作，其实看 `AtomicInteger#incrementAndGet` 就知道了：

```
public final int incrementAndGet() {
    return unsafe.getAndAddInt(this, valueOffset, 1) + 1;
}

public final int getAndAddInt(Object var1, long var2, int var4) {
    int var5;
    do {
        var5 = this.getIntVolatile(var1, var2);
    } while(!this.compareAndSwapInt(var1, var2, var5, var5 + var4));

    return var5;
}
```

其实就是一个无限while循环，直到CAS成功为止。

## 原子类的问题

原子类看似美好的无锁化，有没有问题呢，当然有，典型的两个问题：

1. 高并发下的性能浪费
2. ABA问题


### 高并发下的性能浪费

有锁一般都面临着线程的阻塞、唤醒，而无锁化就是为了减少这个开销。但一旦并发量很大，想象下场景，假设10个线程并发，频繁的CAS操作，但只有极个别有机会CAS成功，极大部分循环的自旋，这直接造成CPU资源严重浪费。

聪明的Java工程师也考虑到了这一点，针对数字类原子类做了优化，例如 `LongAdder` ，具体怎么优化的呢，原本针对1个共享值的加减竞争肯定非常激烈，那么每个线程算自己的呢？算完后再合并不就是了。具体怎么实现的，这里就不展开叙述了。


### ABA问题

CAS确实是原子指令，但前提还得获取旧值，也就是整个过程简化下就是下面2步骤：

```
// 取得shared旧值
old = get(shared);
// cas对shared赋值新值
cas(shared, new, old)
```

看似没有什么问题，但假设线程1获取到 `旧值A` ，线程切换为线程2，线程2获取 `旧值A` 设置为 `新值B` ，线程切换为线程3，线程3获取 `旧值B` 设置为 `新值A`，线程切换回线程1，CAS赋值，匹配旧值还是 `A`，成功赋值新值。

那么这个过程严谨吗，此 `A` 非原 `A`，按道理线程1应该CAS失败，但是成功了，这就是ABA问题。

那么有问题吗，结果不是对的吗，有必要钻牛角尖吗？这是因为上面的例子是 `AtomicInteger`，对1个数字钻牛角尖确实没必要，但换成引用呢，Java中也提供了引用的原子类 `AtomicReference`，我们CAS只会比较引用的地址，也就是说内容若改了其实我们不知道（而恰恰涉及引用时，一般起作用的就是引用里面的值），举个例子：

```java
public class AtomicTest2 {

    public static class User {
        String name;

        public User(String name) {
            this.name = name;
        }
    }

    public static void main(String[] args) throws Exception {
        ExecutorService taskExecutor = Executors.newFixedThreadPool(3);

        CyclicBarrier cyclicBarrier = new CyclicBarrier(2);

        User a = new User("小a");
        User b = new User("小b");

        // 初始化原子类 默认为A
        AtomicReference<User> shared = new AtomicReference<>(a);

        // 线程1
        taskExecutor.execute(() -> {
            User old = shared.get();
            // 只有旧值确定为小a则cas替换为小b
            if ("小a".equals(old.name)) {
                // 屏障模拟线程1被线程切换了
                try {
                    cyclicBarrier.await();
                } catch (Exception e) {
                }
                boolean success = shared.compareAndSet(a, b);
                System.out.println("线程1CAS结果：" + success);
            }

        });

        // 线程2 模拟将A改成B
        taskExecutor.execute(() -> {
            User old = shared.get();
            boolean success = shared.compareAndSet(old, b);
            System.out.println("线程2CAS结果：" + success);
        });
        // 暂停1s确保线程2执行完再执行线程3
        Thread.sleep(1000);

        // 线程3 模拟在线程2后又将B改成A 并且悄悄修改A的名字
        taskExecutor.execute(() -> {
            User old = shared.get();
            a.name = "小a-假";
            boolean success = shared.compareAndSet(old, a);
            System.out.println("线程3CAS结果：" + success);
        });

        // 暂停1s确保线程3也执行完，释放对线程1的屏障，再休息1s确保线程1执行完
        Thread.sleep(1000);
        cyclicBarrier.await();
        Thread.sleep(1000);

        // 输出结果
        System.out.println(shared.get().name);
        System.out.println(a.name);
    }

}
```

我们关注下结果：

```
线程2CAS结果：true
线程3CAS结果：true
线程1CAS结果：true
小b
小a-假
```

那么案例中涉及的3个CAS操作合法吗，合法，但对我们业务来说，结果合理吗，显然不合理。符合业务期望的结果应该是因为线程3改动了a，所以线程1条件应不满足，所以最后shared名称应为`小a-假`。

所以，问题很清晰，涉及引用的CAS操作，如果单纯的比对目标对象的引用地址显然是不严谨的，所以理应引入一个新的具备唯一性的参数做比较参考，Java工程师很聪明，直接用 `Pair` 包装了目标对象和具备唯一性的参数，那么这个 `Pair` 的引用地址每一次都必然是唯一的。

这个全新的原子类就是 `AtomicStampedReference`，它和 `AtomicReference` 的使用区别就是操作时还需要一个值（类似版本号）做比对，一般来说用时间戳完全可以应对了，具体用法我这里就不展示了，留给在座的实战机会。


## 总结

CAS一听高大上，但其实本质并不难理解（你要硬抠U上的底层原理，当我们不认识），Java中原子类家族就是基于CAS做的无锁的锁方案实现，你可以理解为就是乐观锁。


值得注意的是 `synchronized` 和 `AQS` 都有引入CAS操作，但并不是说它们也是乐观锁，它们只是用CAS来优化了下自己。

所以最后请明确记住，CAS不是锁！！！人家只是根正苗红的一条CPU指令！！！。

