# Table of Contents

* [并发之 synchronized 解读](#并发之-synchronized-解读)
    * [与锁的关联](#与锁的关联)
    * [与并发安全性问题的关联](#与并发安全性问题的关联)
    * [使用方式](#使用方式)
    * [效果解读](#效果解读)
    * [可重入性分析](#可重入性分析)
    * [字节码分析](#字节码分析)
    * [引出monitor](#引出monitor)
    * [扩展分析](#扩展分析)
        * [对象结构的补充](#对象结构的补充)
        * [锁升级详解](#锁升级详解)
            * [偏向锁](#偏向锁)
            * [轻量级锁](#轻量级锁)
            * [重量级锁](#重量级锁)
        * [解读monitorenter指令和ACC_SYNCHRONIZED标志](#解读monitorenter指令和acc_synchronized标志)
        * [解读synchronized如何解决并发安全性问题](#解读synchronized如何解决并发安全性问题)
            * [保证原子性](#保证原子性)
            * [保证可见性、有序性](#保证可见性、有序性)
    * [总结](#总结)
    * [参考](#参考)


# 并发之 synchronized 解读

## 与锁的关联

> 在计算机科学中，锁是在执行多线程时用于强行限制资源访问的同步机制，即用于在并发控制中保证对互斥要求的满足。 -- 维基百科

`synchronized` 就是Java中最常见的锁实现方案，硬要套个锁名号，你可以说它是一个悲观锁，但它并不能准确的说是互斥锁，这涉及 `synchronized` 锁升级，具体见下文。


## 与并发安全性问题的关联

针对三大并发安全性问题的说明，可以[参考本文](https://blog.csdn.net/eff666/article/details/66473088)。

在这里，你先记住 `synchronized` 能保证它们，至于如何保证，依旧卖个关子。


## 使用方式

- 作用于普通方法，锁住当前实例对象
```
public synchronized void a(){
...
}
```

- 作用于静态方法，锁住当前Class对象
```
public synchronized static void a(){
...
}
```

- 作用于代码块，锁住指定对象
```
public void c(Object o) {
    synchronized (o) {
    ...
    }
}
```


## 效果解读

针对同一共享数据的`synchronized`加锁，同一时刻有且只有一个线程会取得锁，其他线程在其未释放锁之前在表现形式上都将是被阻塞住（注意这里的说辞！！！）。

从效果来看，只要针对共享变量的操作加了`synchronized`，那么共享变量将必定是安全的，因为对其的并发操作在这里其实被锁成串行化了。


## 可重入性分析

答案很明确，`synchronized`是可重入锁，在其锁住的区域内，对共享数据的再加锁是被视为允许的，具体的原理，下文会讲到。


## 字节码分析

先回到上文再看一下三种使用方式，好了，下面依次贴出三种使用方式测试方法的对应字节码（大概过一下即可，注意下有没有什么特殊的地方）：

```
  public synchronized void a();
    descriptor: ()V
    flags: ACC_PUBLIC, ACC_SYNCHRONIZED
    Code:
      stack=0, locals=1, args_size=1
         0: return
      LineNumberTable:
        line 10: 0
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       1     0  this   Lcom/xyz/study/Main;
```

```
  public static synchronized void b();
    descriptor: ()V
    flags: ACC_PUBLIC, ACC_STATIC, ACC_SYNCHRONIZED
    Code:
      stack=0, locals=0, args_size=0
         0: return
      LineNumberTable:
        line 13: 0
```

```
  public void c(java.lang.Object);
    descriptor: (Ljava/lang/Object;)V
    flags: ACC_PUBLIC
    Code:
      stack=2, locals=4, args_size=2
         0: aload_1
         1: dup
         2: astore_2
         3: monitorenter
         4: aload_2
         5: monitorexit
         6: goto          14
         9: astore_3
        10: aload_2
        11: monitorexit
        12: aload_3
        13: athrow
        14: return
      Exception table:
         from    to  target type
             4     6     9   any
             9    12     9   any
      LineNumberTable:
        line 17: 0
        line 19: 4
        line 20: 14
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0      15     0  this   Lcom/xyz/study/Main;
            0      15     1     o   Ljava/lang/Object;
      StackMapTable: number_of_entries = 2
        frame_type = 255 /* full_frame */
          offset_delta = 9
          locals = [ class com/xyz/study/Main, class java/lang/Object, class java/lang/Object ]
          stack = [ class java/lang/Throwable ]
        frame_type = 250 /* chop */
          offset_delta = 4
```

好了，大概也过了下这三个字节码文件内容，发现的结论如下：

1. `synchronized` 作用于普通方法和静态方法时对比不加时唯一的变动的就是flags里多了一个 `ACC_SYNCHRONIZED` 标志
2. `synchronized` 作用于代码块时在指令里确实看到了不一样的地方，就是多了 `monitorenter` 和 `monitorexit`

带着这两点发现和疑惑，我们继续向下看。


## 引出monitor

相信大家对上文字节码分析里的 `monitorenter` 和 `monitorexit` 指令很是好奇，这里就不卖关子，直接引出 `monitor` 的概念，你可以叫它监视器锁，事实上 `monitor` 就是 `synchronized` 加锁的关键点。

每个对象都有一个 `monitor` ，当 `monitor` 被占用时就表示对象处于锁定状态，而 `monitorenter` 指令就是用来获取 `monitor` 的所有权，`monitorexit` 指令的作用就是释放 `monitor` 的所有权。

也就是说 `synchronized` 加锁的流程（只限重量级锁）如下：

![](http://img.yelizi.top/2ccbfcef-0f95-49e2-8a76-2d5a82e7fcfb.jpg$xyz)

线程若未竞争到 `monitor` ，这个拒绝并不代表就真的放弃掉，而是会将其塞入一个阻塞队列，等待唤醒。

![](http://img.yelizi.top/7717c783-1c18-4f1b-ae56-92c30207d79d.jpg$xyz)


至此，围绕 `monitor` 本身的理解，我们大概可以得出这整个的加锁释放锁竞争锁的流程：

1. 线程尝试获取 `monitor` 所有权
2. 若获取成功，就继续处理；若获取失败，则将线程加入同步阻塞队列中等待唤醒
3. 当前 `monitor` 占用线程释放锁后，系统会主动唤醒同步阻塞队列的第一个，但若是此时有新的线程进来抢占了 `monitor` ，那么唤醒失败（所以 `synchronized` 你还有认为是一个非公平锁）
4. 若获取成功，线程占用     `monitor` ，并将计数+1，同一个线程可重复进入，计数一直+1即可（这就是可重入性的体现）
5. 当前monitor占用线程执行完会通过 `monitorexit` 指令对计数依次-1，直到计数为0，释放线程对 `monitor` 的占用


## 扩展分析

上面虽然分析了 `monitor` 看似对 `synchronized` 加锁已经了解了，但这才是刚开始，JDK6以后对 `synchronized` 进行了优化，具体优化了啥呢？还有 `synchronized` 又是如何解决并发安全性问题的呢？

别急，下文开始才是重头戏，解读总会是一个吃席循序渐进的过程。我假设，到这里为止，我们基本已经对 `synchronized` 加锁有了一个认知，包括如何使用、`monitor`是啥、加锁流程等等。


### 对象结构的补充

为什么要讲到对象结构呢，根据 `synchronized` 使用方式，我们发现它总是会对一个对象去加锁。而上文我提及了每一个对象都有一个 `monitor`，这句话又是什么意思呢？所以，无论如何对对象结构一探究竟对我们分析 `synchronized` 的锁肯定是有极大帮助的。

[Java对象结构解读链接](https://juejin.cn/post/6993308982081224711#heading-3)


若对单独去看Java对象结构不感兴趣，我这里简单说下和 `synchronized` 的关联（当然我建议完整的看一下）：

1. 每个对象都有一个对象头，对象头中有一片区域称为 `Mark Word` ，它占据4个字节（32位系统）或8个字节（64位系统），简单的说 `Mark Word` 中会有一些标记相关的记录，包括对象锁状态就记录在这个 `Mark Word` 中；
2. `Mark Word` 中有2个标记位：偏向锁定标记（1bit），锁状态（2bit），这两个组合就有了如下5个状态：
    - 0:01 无锁
    - 1:01 偏向锁
    - 0:00 轻量级锁
    - 0:10 重量级锁
    - 0:11 GC标记
3. 第2点中锁状态为 0:10 时，也就是重量级锁时，`Mark Word` 还会记录指向 `monitor` 的指针（这就解释了为什么每一个对象都有一个 `monitor`）；
4. 第2点中偏向锁定标记你可以理解为区分无锁和偏向锁的一个标记。

### 锁升级详解

上文对象结构里涉及的锁状态抛开 GC标记 ，有4个和锁挂钩的状态：无锁、偏向锁、轻量级锁、重量级锁。

其实排除无锁其它3个都可以认为是 `synchronized` 锁，这是JDK6开始对 `synchronized` 的优化，优化就是 `synchronized` 锁是经过升级一步步变重（仅限理解意义上的）：

无锁 -> 偏向锁 -> 轻量级锁 -> 重量级锁


#### 偏向锁

引入偏向锁是为了在无多线程竞争的情况下尽量减少不必要的加锁、解锁开销。

偏向锁默认是开启的，通过 `-XX:-UseBiasedLocking=false` 参数可控制关闭。

若开启偏向锁，对象初始化时 `Mark Word` 中偏向锁定标记位为 `1` 且锁状态为 `01` ，还有一片区域用来记录偏向线程ID（即持有偏向锁的线程），这个偏向线程ID默认为0，仅初始化的对象可以说是处于是匿名偏向状态（指仍未有线程持有偏向锁）。


偏向锁的加锁流程如下（仅供理解）：

![](http://img.yelizi.top/88f1eb08-feae-4217-b210-5ab12bdfd894.jpg$xyz)

针对流程图的几点说明：

1. 再次强调，流程图仅供理解，实际执行有能耐请看HotSpot源码；
2. 为什么在偏向锁撤销时有个先转为无锁状态又升级成轻量级锁的看似多此一举的步骤，依照我目前认知水平，这也是我想问的为什么；
3. CAS设置偏向线程ID可以认为是 `update ThreadID = xx where ThreadID = 0` 这么个原子指令过程；
4. 全局安全点（savepoint）是JVM的概念，这个点可以无忧的进行STW；
5. 如何理解有两个分支走到偏向锁撤销逻辑
    1. CAS设置偏向线程ID失败：很好理解，对象初始化为可偏向状态，假设有多个线程都判断完偏向线程ID为0，所以要进行一次CAS原子指令竞争设置偏向线程ID。若没有竞争失败，那就获取偏向锁，竞争失败，那偏向锁就直接失去作用了，所以需要撤销；
    2. 偏向线程ID非自己： 也就是已经有线程占据过本对象的偏向锁，为什么说是占据过，因为偏向锁一旦获取不论线程是否走完同步代码块还是已经死亡，都不会去主动释放这个偏向线程ID。而为什么该情况也需要走偏向锁撤销，因为当前线程并不知道原偏向线程的状态，并且出现新的线程想要获取偏向锁，就有理由相信当前状况已经不符合偏向锁含义了。

总体可以理解为一旦有超过一个线程想获取锁，就有理由相信偏向锁失效了。所以我觉得偏向锁更像是一个一次性的用来应对特殊场景的无锁方案。


#### 轻量级锁

引入轻量级锁是为了减少竞争不激烈的多线程场景下不断阻塞、唤醒带来的开销。

当锁升级为轻量级锁时，`Mark Word` 中锁状态位为 `01`（这里就不用关注偏向锁定标记位了），并在当前线程的栈帧中建立一个名为锁记录（Lock Record）的空间，用来存储当前锁对象的 `Mark Word` 拷贝。

轻量级锁的加锁流程如下（仅供理解，比起偏向锁好理解太多...）：

![](http://img.yelizi.top/6957d43f-e8bc-4248-9a02-9d05dab4ef64.jpg$xyz)

针对流程图的几点说明：

1. CAS替换Mark Word中的锁记录指针可以理解为 `update pointer=xxx where pointer=0` 这样的原子指令；
2. 释放锁的流程图中表示的不太明确，补充下：
    - CAS替换Mark Word为线程锁记录中拷贝的那份。成功那么释放成功；失败则说明锁已经膨胀为重量级锁，那就按重量级锁的逻辑唤醒阻塞住的线程


#### 重量级锁

重量级锁则就是一般意义上的 `synchronized` 锁，也就是上文提到的 `monitor` 方案。

当锁升级为重量级锁时，`Mark Word` 中锁状态位为 `10`，还有一片区域维护 `monitor` 的指针。

至于重量级锁的加锁流程回顾上文[引出monitor](#引出monitor)


### 解读monitorenter指令和ACC_SYNCHRONIZED标志

`ACC_SYNCHRONIZED` 标志其实就是将整个方法识别为同步代码块，在调用方法前会执行 `monitorenter` 指令，在调用结束后会执行 `monitorexit` 指令。

`monitorenter` 指令也不是一下子就用到重量级锁的实现方案 `monitor`，会经历偏向锁、轻量级锁的过程。

想要了解的更多去看源码吧。


### 解读synchronized如何解决并发安全性问题

#### 保证原子性

`synchronized` 能保证原子性源于它是一个独享锁，同一时刻只有一个线程能执行同步代码块，就算因为时间片用完发生线程切换，对于这个同步代码块依旧只有当前线程能执行。


#### 保证可见性、有序性

`monitorenter` 指令之前会施加 `Load屏障（读屏障）`，而在 `monitorexit` 指令之后会施加 `Store屏障（写屏障）`。

了解屏障指令的就知道如何保证了可见性和有序性，理解上其实等同于 `volatile`。


## 总结

`synchronized` 是Java的一个非常优秀的解决并发安全性问题的锁方案，不要惧怕使用，而是要在合适的场景合理运用它来武装我们的代码。

本篇总体来说完成的还是很仓促（偏向锁太搞心态了，源码看不太明白，搜了很多地方，理解真是哈姆雷特），细节很多并没有考虑，尤其是讲 `synchronized` 如何解决并发安全性问题，直接一笔带过了...后面有时间再慢慢雕琢它吧。


## 参考

- [https://wiki.openjdk.java.net/display/HotSpot/Synchronization](https://wiki.openjdk.java.net/display/HotSpot/Synchronization)
- [https://tech.meituan.com/2018/11/15/java-lock.html](https://tech.meituan.com/2018/11/15/java-lock.html)
- [https://www.zhihu.com/question/57774162/answer/154298044](https://www.zhihu.com/question/57774162/answer/154298044)
- [https://cloud.tencent.com/developer/article/1344227](https://cloud.tencent.com/developer/article/1344227)