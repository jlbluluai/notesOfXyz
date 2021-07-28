# Table of Contents

* [隐式锁之synchronized解读](#隐式锁之synchronized解读)
  * [简介](#简介)
  * [使用方式](#使用方式)
  * [效果解读](#效果解读)
  * [原理分析](#原理分析)
    * [作用于普通方法分析](#作用于普通方法分析)
    * [作用于静态方法分析](#作用于静态方法分析)
    * [作用于代码块分析](#作用于代码块分析)
      * [monitor介绍](#monitor介绍)
  * [没竞争到monitor的线程会怎样？](#没竞争到monitor的线程会怎样？)
  * [实战演练](#实战演练)
    * [单例模式-懒汉式双重检测](#单例模式-懒汉式双重检测)
  * [synchronized的优化](#synchronized的优化)
    * [偏向锁](#偏向锁)
    * [轻量级锁](#轻量级锁)
      * [自旋策略](#自旋策略)
    * [重量级锁](#重量级锁)
    * [总结](#总结)



# 隐式锁之synchronized解读


## 简介

synchronized是Java的关键字，常用于解决并发安全性问题，简言之就是加锁实现。

在Java6以前，synchronized就是很重的实现，也就是我们说的悲观锁，所以普遍认知下synchronized一直是个重锁，这也没啥问题。

Java6以后开始对synchronized进行了一些优化，稍微改观了一点，当然论轻量级依旧是没法和灵活的Lock比较的，具体使用还得看场景具体分析。


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

synchronized既然是悲观锁，那就看下悲观锁的定义。

> 悲观锁，正如其名，具有强烈的独占和排他特性。它指的是对数据被外界（包括本系统当前的其他事务，以及来自外部系统的事务处理）修改持保守态度，因此，在整个数据处理过程中，将数据处于锁定状态。悲观锁的实现，往往依靠数据库提供的锁机制（也只有数据库层提供的锁机制才能真正保证数据访问的排他性，否则，即使在本系统中实现了加锁机制，也无法保证外部系统不会修改数据）。 - 百度百科

简单来说，被synchronized锁住后，原本可以并发执行的同时只能有一个执行了，也就是俗称的变成串行化操作。


## 原理分析

既然synchronized是关键字，鉴于我们对Java的了解，总会在JVM层面对其进行特殊处理。实际上呢，我们反编译class文件看字节码文件就能看出猫腻。


### 作用于普通方法分析

字节码
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

flags（标识）里多了一个ACC_SYNCHRONIZED，在JVM，线程如若要调用该方法，需要先获取monitor，获取成功后才继续，结束后释放monitor。


### 作用于静态方法分析

字节码
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

分析同[作用于普通方法分析](#作用于普通方法分析)


### 作用于代码块分析

字节码
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

可以发现字节码中已经没有明确的synchronized了，取而代之的是monitorenter和monitorexit指令。

这里扩展下monitor的知识

#### monitor介绍

每个对象都有一个monitor（监视器锁），当monitor被占用时就表示对象处于锁定状态，而monitorenter指令就是用来获取monitor的所有权，monitorexit指令的作用就是释放monitor的所有权。

总结下流程：

![](http://106.15.233.185:8983/2ccbfcef-0f95-49e2-8a76-2d5a82e7fcfb.jpg)



## 没竞争到monitor的线程会怎样？

先来张图

![](http://106.15.233.185:8983/7717c783-1c18-4f1b-ae56-92c30207d79d.jpg)

1. 线程尝试获取monitor所有权
2. 若获取失败，则将线程加入同步阻塞队列中等待
3. 当前monitor占用线程释放后，系统会主动唤醒同步阻塞队列的第一个，但若是此时有新的线程进来抢占了monitor，那么唤醒失败（非公平锁的体现）
4. 若获取成功，线程进入monitor，并将计数+1，同一个线程可重复进入，计数一直+1即可（可重入锁的体现）
5. 线程执行完会monitorexit指令对计数依次-1，直到计数为0，释放线程对monitor的占用


所以没竞争到monitor的线程会暂时进入同步阻塞队列等待唤醒


## 实战演练


### 单例模式-懒汉式双重检测

``` 
public class IdGenerator {

    /**
     * 使用原子类 防止并发安全性问题
     */
    private AtomicLong id = new AtomicLong(0);

    /**
     * 懒汉式延迟初始化出实例
     */
    private static IdGenerator INSTANCE;

    /**
     * 空私有构造函数 防止外部new出实例
     */
    private IdGenerator() {
    }

    public static IdGenerator getInstance() {
        // 加锁外加双重验证 多个线程同时进来，恰巧实例没有初始化过，都不能通过第一层校验，于是一个抢到锁，进去创建实例，
        // 若不加二层校验，其他线程等待到锁后，进入后又会重新创建实例，造成并发安全性问题
        if (Objects.isNull(INSTANCE)) {
            synchronized (IdGenerator.class) {
                if (Objects.isNull(INSTANCE)) {
                    INSTANCE = new IdGenerator();
                }
            }
        }
        return INSTANCE;
    }

    public long getId() {
        return id.incrementAndGet();
    }

}
```

这里就不过多的叙述单例模式相关，直接主题-synchronized，懒汉式核心就是用到时才初始化实例，这里就会涉及到一个并发问题，多个线程同时需要实例时同时来获取，若不加处理，后果就是假设10个线程同时判断实例没有初始化都去初始化，就会初始化10次。

所以这里加了一层synchronized，同时只有一个线程能初始化实例，并且再加个双重检测，几乎可以避免这里的并发安全性问题


## synchronized的优化

Java6后引入偏向锁和轻量级锁对原本只是重量级锁的synchronized进行了优化，主要采取了对synchronized进行锁升级的策略。

该策略：<br>
偏向锁 -> 轻量级锁 -> 重量级锁

这里还要提到对象头中的Mark World，就是由它来配合锁策略。


### 偏向锁

synchronized同步代码块时，默认上偏向锁，当有线程访问同步块时，只需要判断对象头的Mark World中是否有偏向锁指向了线程ID。

有什么优势呢，回顾synchronized的原理，每当线程获取锁时需要去monitor竞争下，但是现在只需要根据对象头的Mark World判断一下是当前线程占有偏向锁，那就直接获取锁。

**优化点**：针对没有竞争时，同时只有单个线程在工作，但要多次获取锁，这样后续获取锁就可以避免竞争再判断，减少了开销。


### 轻量级锁

首先满足只有单线程工作的时候，这时候是偏向锁，然后新的线程来获取锁，这时候会发现Mark World中记录的线程ID不是自己，就会进行CAS操作获取锁。若是获取到锁，那还是说明没有竞争，那就把线程ID替换成自己，继续维持偏向锁；若是没有获取到锁，那说明存在竞争，那么就会触发锁升级，升级称为轻量级锁。

这时候我们的自旋策略就该上场了。


#### 自旋策略

轻量级锁的时候，竞争锁失败后会咋办，这时候就会进行自旋，怎么理解呢，简单来说就是再次尝试，尝试个几次，万一就获取锁了呢。

当然次数不是无限的，自旋也是占CPU资源的，如果长期自旋，那就是白白浪费资源，Java6默认了10次并提供开关以及次数配置的启动参数，但是Java7就取消了，默认开启并且自适应调整次数。

**优化点**：针对线程竞争锁，我们乐观的认为能拿到锁，并配合自旋减少线程阻塞再唤醒的开销。

### 重量级锁

如果轻量级锁自旋超过次数还没有获取到锁时咋办呢，这说明经过尝试自旋失败，那很大概率再自旋也会失败，锁就会升级为重量级锁，回归synchronized本来的样子。


### 总结

经过这么一轮优化，synchronized就不是一直都是重量级选手，而是按照并发的程度依次升级，其实也有一种懒加载的含义在里面，等要用到的时候再升级。
