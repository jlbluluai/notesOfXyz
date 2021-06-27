参考：

Java并发编程实战（极客时间-王宝令）

[TOC]

### 前言

&emsp;&emsp;并发中去实现锁可以通过管程模型也可以通过信号量模型，其实信号量诞生的更早，但是实现繁琐，所以仅仅在解决互斥上Java选择更“灵性”的管程模型，但在Java 并发中却保留了信号量的实现——Semaphore，其存在必有其存在的道理，那么本篇便算作并发的一个番外篇去了解下信号量以及在Java中的功效。



### 原理分析

&emsp;&emsp;信号量模型你要去分析也很简单，就三点：一个计数器，一个等待队列和三个方法（init、down、up）。这个模型用图抽象下就是这样。

![](https://raw.githubusercontent.com/jlbluluai/notesOfXyz/master/img/core/bingfa003.png)

&emsp;&emsp;init方法初始化计数器的初始值，down方法则是决定当前线程能否进入临界区，计数器减一，若小于0则说明已经没有空位，当前线程只能进入等待队列，up方法表示线程占用临界区结束，计数器加1，若小于等于0则说明等待队列是有线程阻塞的，则唤醒一个线程。用代码表示的话就是这样。

```java
class Semaphore{
  // 计数器
  int count;
  // 等待队列
  Queue queue;
  // 初始化操作（理解为初始化多大容量的池）
  Semaphore(int c){
    this.count=c;
  }
  // 
  void down(){
    this.count--;
      //减1计数就小于0表示池占满，阻塞
    if(this.count<0){
      // 将当前线程插入等待队列
      // 阻塞当前线程
    }
  }
  void up(){
    this.count++;
      //每执行down都会减1，不管阻塞不阻塞，所以加一还是小于等于0说明之前有线程阻塞，唤醒。
    if(this.count<=0) {
      // 移除等待队列中的某个线程 T
      // 唤醒线程 T
    }
  }
}
```



### 互斥的实现

&emsp;&emsp;其实很好想，互斥是保证同一时刻只有一个线程能进入临界区，那么其实不就是init方法设置初始值为1的计数器不就行了嘛。用Java 并发包中的Semaphore模拟下。

```java
static int count;
// 初始化信号量
static final Semaphore s 
    = new Semaphore(1);
// 用信号量保证互斥    
static void addOne() {
  s.acquire();
  try {
    count+=1;
  } finally {
    s.release();
  }
}
```

&emsp;&emsp;执行acquire方法（语义就是down），计数器减一，就为0了，这时候后面的线程再减就小于0了，就会阻塞进入等待队列，当前线程处理完finally代码块中必然执行release方法（语义就是up），计数器加以，若是等待队列有线程，自然小于等于0，就会唤醒一个线程。这样互斥就通过信号量模型实现了，其实大体看和管程模型是不是有异曲同工之妙呢。



### 限流器的实现

&emsp;&emsp;要说实现互斥，不谈我们的理解看法，Java在历史上已经做出了正确的选择，选用了更好的管程模型，那要是信号量模型只能实现互斥显然Java没必要多次一举在并发包中加入Semaphore这个类（Java又不是博物馆，历史还要保留啥的），存在必合理，肯定有什么信号量模型能做到的而管程模型做不到的。我们品味品味信号量模型和管程模型最大的不同就是信号量模型允许多个线程进入临界区（实现互斥也是把1作为这个“多”），而管程模型只能是1（可以看出管程模型更像专门为互斥而生的），那么这个多个线程进入临界区在实际中有什么用处吗？想想池化资源。

&emsp;&emsp;正如线程池、数据库连接池、对象池之类的之类的池化资源不正是手中有多个资源，允许多个线程进入的吗，以对象池为例写个例子。所谓对象池，一次性创建多个对象，多个线程重复利用这些对象。

```java
class ObjPool<T, R> {
  final List<T> pool;
  // 用信号量实现限流器
  final Semaphore sem;
  // 构造函数
  ObjPool(int size, T t){
    pool = new Vector<T>(){};
    for(int i=0; i<size; i++){
      pool.add(t);
    }
    //构造同等大小的信号器
    sem = new Semaphore(size);
  }
  // 利用对象池的对象，调用 func
  //Function是Java8提供的函数式接口，语义：接受一个参数，返回一个结果，具体啥看自己需求，
  R exec(Function<T,R> func) {
    T t = null;
    sem.acquire();
    try {
      t = pool.remove(0);
      return func.apply(t);
    } finally {
      pool.add(t);
      sem.release();
    }
  }
}
// 创建对象池
ObjPool<Long, String> pool = 
  new ObjPool<Long, String>(10, 2);
// 通过对象池获取 t，之后执行  
pool.exec(t -> {
    System.out.println(t);
    return t.toString();
});
```

