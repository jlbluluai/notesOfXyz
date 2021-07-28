# Table of Contents

* [显示锁之Lock解读](#显示锁之lock解读)
  * [由来](#由来)
  * [核心方法](#核心方法)
  * [实战演练](#实战演练)
    * [lock()方法](#lock方法)
    * [lockInterruptibly()方法](#lockinterruptibly方法)
    * [tryLock()方法](#trylock方法)
    * [tryLock(long time, TimeUnit unit)方法](#trylocklong-time-timeunit-unit方法)
    * [newCondition()方法](#newcondition方法)

# 显示锁之Lock解读

## 由来

synchronized加锁实战大家都耳熟能详了，这个锁虽然简单易用，但也有其局限性，无法中断、不支持超时控制、使用不够灵活等等，这种种的局限性在高并发场景越来越多的年代让其也倍感局促，所以肯定要有其替代品，于是就有了显示的Lock，并且Java提供了一个其实现基础AQS（单独开篇讲解）。然后根据这个接口Lock，Java在并发包里提供了大量关于显示锁的骚操作。


## 核心方法

那就看看这个Lock到底定义了啥。

```
// 获取锁 意义上和synchronized类似
void lock();

// 获取锁 允许打断
void lockInterruptibly() throws InterruptedException;

// 尝试获取锁 获取返回true 否则false
boolean tryLock();

// 尝试获取锁【等待时长time】 若获取返回true 超过等待时长还未获取返回false
boolean tryLock(long time, TimeUnit unit) throws InterruptedException;

// 解锁 
void unlock();

// 新增一个绑定到当前Lock的条件
Condition newCondition();
```

## 实战演练

**锁实例用ReentrantLock**

```
private final Lock lock = new ReentrantLock();
```

### lock()方法

测试方法
```
public void testLock(int a) {
    try {
        lock.lock();
        log.info("time={} a={}", LocalDateTime.now(), a);
        sleep(1000);
    } catch (Exception e) {
        e.printStackTrace();
    } finally {
        lock.unlock();
    }
}
```

测试用例
```
Thread t1 = new Thread(()->lockTest.testLock(1));
t1.start();

Thread t2 = new Thread(()->lockTest.testLock(2));
t2.start();
```

结果
```
17:46:49.761 [Thread-0] INFO com.xyz.study.common.owner.concurrent.LockTest - time=2021-07-27T17:46:49.759 a=1
17:46:50.767 [Thread-1] INFO com.xyz.study.common.owner.concurrent.LockTest - time=2021-07-27T17:46:50.767 a=2
```

分析日志时间，两个线程虽几乎同时开启，但是线程t2等了t1 1s，该用法效果其实和synchronized差不多，只不过这个显示的自己加锁释放锁。


### lockInterruptibly()方法

测试方法
```
public void testLockInterrupt(int a) {
    try {
        lock.lockInterruptibly();
        log.info("time={} a={}", LocalDateTime.now(), a);
        sleep(1000);
    } catch (Exception e) {
        e.printStackTrace();
    } finally {
        lock.unlock();
    }
}
```

测试用例
```
Thread t1 = new Thread(()->lockTest.testLockInterrupt(1));
t1.start();

Thread t2 = new Thread(()->lockTest.testLockInterrupt(2));
t2.start();
t2.interrupt();
```

测试结果
```
java.lang.InterruptedException
	at java.util.concurrent.locks.AbstractQueuedSynchronizer.acquireInterruptibly(AbstractQueuedSynchronizer.java:1220)
	at java.util.concurrent.locks.ReentrantLock.lockInterruptibly(ReentrantLock.java:335)
	at com.xyz.study.common.owner.concurrent.LockTest.testLockInterrupt(LockTest.java:38)
	at com.xyz.study.common.owner.concurrent.LockTest.lambda$main$1(LockTest.java:70)
	at java.lang.Thread.run(Thread.java:748)
Exception in thread "Thread-1" java.lang.IllegalMonitorStateException
	at java.util.concurrent.locks.ReentrantLock$Sync.tryRelease(ReentrantLock.java:151)
	at java.util.concurrent.locks.AbstractQueuedSynchronizer.release(AbstractQueuedSynchronizer.java:1261)
	at java.util.concurrent.locks.ReentrantLock.unlock(ReentrantLock.java:457)
	at com.xyz.study.common.owner.concurrent.LockTest.testLockInterrupt(LockTest.java:44)
	at com.xyz.study.common.owner.concurrent.LockTest.lambda$main$1(LockTest.java:70)
	at java.lang.Thread.run(Thread.java:748)
17:51:34.817 [Thread-0] INFO com.xyz.study.common.owner.concurrent.LockTest - time=2021-07-27T17:51:34.815 a=1
```

可以看到由于线程t1先抢占了锁，线程t2还在等待，这时t2调了中断，于是就中断了，这样看好像没啥感觉，我们把两个线程调取的方法改成测试lock()的，启动，你会发现`t2.interrupt();`没有任何作用。这个锁允许中断的含义体现在这里。


### tryLock()方法

测试方法
```
public void testTryLock(int a) {
    log.info("try get lock a={} now={}", a, LocalDateTime.now());
    boolean flag = lock.tryLock();
    if (!flag) {
        log.info("cannot get lock a={} now={}", a, LocalDateTime.now());
        return;
    }
    log.info("time={} a={}", LocalDateTime.now(), a);
    sleep(5000);
    lock.unlock();
}
```

测试用例
```
Thread t1 = new Thread(() -> lockTest.testTryLock(1));
t1.start();

Thread t2 = new Thread(() -> lockTest.testTryLock(2));
t2.start();
```

测试结果
```
18:01:55.004 [Thread-1] INFO com.xyz.study.common.owner.concurrent.LockTest - try get lock a=2 now=2021-07-27T18:01:55.002
18:01:55.004 [Thread-0] INFO com.xyz.study.common.owner.concurrent.LockTest - try get lock a=1 now=2021-07-27T18:01:55.002
18:01:55.008 [Thread-1] INFO com.xyz.study.common.owner.concurrent.LockTest - time=2021-07-27T18:01:55.008 a=2
18:01:55.008 [Thread-0] INFO com.xyz.study.common.owner.concurrent.LockTest - cannot get lock a=1 now=2021-07-27T18:01:55.008
```

仔细看没抢占到锁的`Thread-0`，开始尝试获取锁到获取失败返回false几乎没有停顿。



### tryLock(long time, TimeUnit unit)方法

测试方法
```
public void testTryLockWithTimeout(int a) {
    log.info("try get lock a={} now={}", a, LocalDateTime.now());
    boolean flag = false;
    try {
        flag = lock.tryLock(2, TimeUnit.SECONDS);
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
    if (!flag) {
        log.info("cannot get lock a={} now={}", a, LocalDateTime.now());
        return;
    }
    log.info("time={} a={}", LocalDateTime.now(), a);
    sleep(5000);
    lock.unlock();
}
```

测试用例
```
Thread t1 = new Thread(() -> lockTest.testTryLockWithTimeout(1));
t1.start();

Thread t2 = new Thread(() -> lockTest.testTryLockWithTimeout(2));
t2.start();
```

测试结果
```
13:49:38.045 [Thread-1] INFO com.xyz.study.common.owner.concurrent.LockTest - try get lock a=2 now=2021-07-28T13:49:38.042
13:49:38.045 [Thread-0] INFO com.xyz.study.common.owner.concurrent.LockTest - try get lock a=1 now=2021-07-28T13:49:38.042
13:49:38.051 [Thread-1] INFO com.xyz.study.common.owner.concurrent.LockTest - time=2021-07-28T13:49:38.051 a=2
13:49:40.056 [Thread-0] INFO com.xyz.study.common.owner.concurrent.LockTest - cannot get lock a=1 now=2021-07-28T13:49:40.056
```

注意看`Thread-0`的日志，打印没获取到锁是在上一条日志都2s后，说明尝试获取锁是经过了2s的超时设置的。


### newCondition()方法

该方法产生了一个Condition，看着懵逼，其实它就是替代配合synchronized的wait/notify机制来配合Lock。看其注释，用法上都差不多，当前线程必须持有锁才能调用Condition都await，那不就是wait只能在synchronized中使用是一个意思吗？

> Before waiting on the condition the lock must be held by the current thread. A call to Condition.await() will atomically release the lock before waiting and re-acquire the lock before the wait returns.

看概念还是很抽象，下面直接上实战。

定义公共的条件
```
private final Condition condition = lock.newCondition();
```

测试方法
```
public void testNewCondition(int a) {
    try {
        lock.lock();
        if (a == 1) {
            condition.await();
        }
        log.info("time={} a={}", LocalDateTime.now(), a);
        sleep(2000);
        condition.signalAll();
    } catch (Exception e) {
        e.printStackTrace();
    } finally {
        lock.unlock();
    }
}
```

测试用例
```
Thread t1 = new Thread(() -> lockTest.testNewCondition(1));
t1.start();

try {
    Thread.sleep(1000);
} catch (InterruptedException e) {
    e.printStackTrace();
}

Thread t2 = new Thread(() -> lockTest.testNewCondition(2));
t2.start();
```

测试结果
```
14:15:02.995 [Thread-1] INFO com.xyz.study.common.owner.concurrent.LockTest - time=2021-07-28T14:15:02.993 a=2
14:15:05.002 [Thread-0] INFO com.xyz.study.common.owner.concurrent.LockTest - time=2021-07-28T14:15:05.002 a=1
```

首先condition必须是公用的，这里不像使用wait时有个明确的对象，如果你每次用newCondition获取条件，那就是不一样的条件，那么通过不一样的条件去唤醒等待线程自然是不可能的。

为了排除干扰，特地还在t1和t2中间睡眠了1s，确保t1先拿到锁，观察日志，显然t1的await生效了，然后释放了锁，t2再拿到锁，执行并sleep2s后唤醒t1，t1才继续执行。

