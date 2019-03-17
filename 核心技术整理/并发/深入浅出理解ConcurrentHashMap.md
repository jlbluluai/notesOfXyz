参考：

https://segmentfault.com/a/1190000009001468

https://www.cnblogs.com/ITtangtang/p/3948786.html#undefined（补充JDK1.7ConcurrentHashMap）

https://www.cnblogs.com/lujiango/p/7580558.html

https://www.jianshu.com/p/2829fe36a8dd（专门讲transfer）



### 前言

​	HashMap大家都听说过，也应该知道HashMap是非线程安全的。有人说要想线程安全可以用HashTable，没错，但HashTable实现线程的方式是同步，有点牺牲性能的。有时候强硬的有性能要求的时候，Java也是提供策略的，Java额外的专门提供了并发的工具包，java.util.concurrent，我们今天要介绍的就是HashMap的并发替代的ConcurrentHashMap。



### 背景

​	所以说为啥HashMap是线程不安全的呢？这里不展开叙述线程安全性的问题，举个简单的例子，线程A和B都在执行put方法，里面必然有个++size的操作，比如两个线程执行前size是1，两个线程很大可能的会执行完size是2而不是理想的3。

​	HashTable的效率低就低在无脑的使用syncronized，所以它锁住的将会是整个map，所以高并发下会有种这个并发咋效果不太好的感觉，因为都串行化了。

​	那么ConcurrentHashMap在并发下的性能是如何保证的呢？那就是分段锁，在JDK1.8下ConcurrentHashMap的结构基本与HashMap一致了，先是有一个table数组，每个节点衍生出一个链表，当链表长度大于等于8时就转换成红黑树，我们姑且称table的每个节点的内容是一个bucket（桶），那么ConcurrentHashMap怎么实现分段锁的呢，每个bucket实现一个锁，这样就把原本HashTable的包场缩减到整个table的每个节点，那自然加锁时的等待概率就低很多了。



### ConcurrentHashMap的基本数据结构（持续补充）

- transient volatile Node<K,V>[] table;	键值对桶数组
- private transient volatile Node<K,V>[] nextTable;	扩容时用到的新键值对数组
- private transient volatile long baseCount;	记录当前键值对总数，通过CAS更新，对所有线程可见
- private transient volatile int sizeCtl;	表示键值对总数阈值，通过CAS更新
  - 小于0，多个线程等待扩容
  - 等于0，默认值
  - 大于0，扩容的阈值



#### 键值对内部类Node<K, V>

```java
static class Node<K,V> implements Map.Entry<K,V> {
    final int hash;
    final K key;
    // 键值对的value和next均为volatile类型
    volatile V val;
    volatile Node<K,V> next;
    ...
}
```

​	其中主要提一下next，JDK1.7版本并不是volatile修饰的，是final修饰的，大家也知道final修饰是具有不可变性的，所以当时的删除中间节点的数据时是把前面的节点的都拷贝一份然后接续后面的节点的。JDK1.8将final修饰去掉并添加volatile，可能在操作上会有什么不一样吧。



### ConcurrentHashMap重要方法分析

​	先看ConcurrentHashMap的空构造函数。

```java
    public ConcurrentHashMap() {
    }
```

​	没错，前面也说了JDK1.8中ConcurrentHashMap的结构基本与HashMap一致，在HashMap的专栏里我也提及过，调用默认构造函数时，初始时是为分配任何容量的，更像时只是构造了一个实例化对象，分配了内存空间，在第一次加入数据时才会分配初始容量为16。



#### put方法

​	话不多说，直接去看加数据的put方法。

```java
    public V put(K key, V value) {
        return putVal(key, value, false);
    }
```

​	和HashMap的处理方式一致，都是传入key-value，然后调用了putVal方法，不过显然这边putVal方法的参数只有三个，对比HashMap中的有五个还是少了不少了，看来我们不能简单去类比了，毕竟还涉及了分段锁的概念，让我们一探究竟。

```java
    final V putVal(K key, V value, boolean onlyIfAbsent) {
        if (key == null || value == null) throw new NullPointerException();
        int hash = spread(key.hashCode());
        int binCount = 0;
        /**
         *	这边的死循环的意义就是对于里面一个CAS探测空桶成功却因另一个线程先行占有而返回false准备的，
         *	毕竟再往下就是抢占锁了，那个抢不到线程会直接进入阻塞状态的，这个死循环没有意义，意义就在于
         *	那唯一用到CAS的空桶探测了
         */
        for (Node<K,V>[] tab = table;;) {
            Node<K,V> f; int n, i, fh;
            //若为空表（table数组为空），进行初始化
            if (tab == null || (n = tab.length) == 0)
                tab = initTable();
            /**
             *	CAS探测空桶
             *	i=(n-1)&hash计算出key所在的bucket表中数组索引，通俗的讲，它是哪个桶的
             *	f取得的就是桶的首位元素了
             *	为空时说明该桶还未有数据，直接通过CAS方式加入（万一两个线程都在加，	  
             *	所以这边用无锁的同步方式很合适）
             */
            else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
                if (casTabAt(tab, i, null,
                             new Node<K,V>(hash, key, value, null)))
                    break;                   // no lock when adding to empty bin
            }
            //检测到tab[i]桶正在进行rehash，啥意思呢，通俗的讲，map在扩容，helpTransfer是帮助扩容的
            //意思，最后得到新的table再往下操作，详细的helpTransfer待整理
            else if ((fh = f.hash) == MOVED)
                tab = helpTransfer(tab, f);
            else {
                V oldVal = null;
                //对桶的首位元素上锁独占，相当于每个桶都有自己的锁，加值时只要不在同一个桶，就不会冲突
                synchronized (f) {
                    if (tabAt(tab, i) == f) {
                        //桶组成是链表的情况
                        if (fh >= 0) {
                            binCount = 1;
                            for (Node<K,V> e = f;; ++binCount) {
                                K ek;
                                //查找到对应键值对，更新值
                                if (e.hash == hash &&
                                    ((ek = e.key) == key ||
                                     (ek != null && key.equals(ek)))) {
                                    oldVal = e.val;
                                    if (!onlyIfAbsent)
                                        e.val = value;
                                    break;
                                }
                                //桶中找不到对应的键值对，则在链表尾部插值
                                Node<K,V> pred = e;
                                if ((e = e.next) == null) {
                                    pred.next = new Node<K,V>(hash, key,
                                                              value, null);
                                    break;
                                }
                            }
                        }
                        //桶组成是红黑树的情况
                        else if (f instanceof TreeBin) {
                            Node<K,V> p;
                            binCount = 2;
                            //执行putTreeVal方法往红黑树里加值，若是已存在，则替换旧值
                            if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                           value)) != null) {
                                oldVal = p.val;
                                if (!onlyIfAbsent)
                                    p.val = value;
                            }
                        }
                    }
                }
                if (binCount != 0) {
                    //如果键值对数大于等于8执行treeifyBin方法，也就是链表转红黑树
                    if (binCount >= TREEIFY_THRESHOLD)
                        treeifyBin(tab, i);
                    if (oldVal != null)
                        return oldVal;
                    break;
                }
            }
        }
        //键值对数加1，该方法内部也可能执行transfer扩容操作
        addCount(1L, binCount);
        return null;
    }
```

​	putVal方法中最引人注意的当属synchronized (f) {这句话了，f是啥呢，每个桶的首位元素，对其加锁等同于对一个桶加锁，自然我们也明白了分段锁的运用了，每个桶都有自己的锁，就不至于像HashTable一样包场了。

​	针对主体的一个大大的死循环，我在注释处也详细的分析了我自己的看法。其中涉及到helpTransfer，putTreeVal，addCount方法后面有机会展开分析。



#### get方法

​	提完加值的put方法，那肯定少不了取值的get方法了，ConcurrentHashMap只针对写使用了分段锁的概念，针对读操作是没有加锁的，不过因为写锁是禁止读写的，所以有线程占据某个桶的写锁时，对这个桶的读自然会被阻塞。

```java
    public V get(Object key) {
        Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
        int h = spread(key.hashCode());
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (e = tabAt(tab, (n - 1) & h)) != null) {
            //头节点就是寻找的value的情况
            if ((eh = e.hash) == h) {
                if ((ek = e.key) == key || (ek != null && key.equals(ek)))
                    return e.val;
            }
            //map扩容或是红黑树的情况寻值
            else if (eh < 0)
                return (p = e.find(h, key)) != null ? p.val : null;
            //循环链表找寻值
            while ((e = e.next) != null) {
                if (e.hash == h &&
                    ((ek = e.key) == key || (ek != null && key.equals(ek))))
                    return e.val;
            }
        }
        return null;
    }
```

​	

#### remove方法

​	看JDK1.8源码时，觉得这个很有意思，在JDK1.7中，ConcurrentHashMap采用的是一个叫Segment 的玩意存储的，当时里面的里面的next节点是final修饰的，上面介绍Node内部类也提及过，自然这样删除就得把前面的节点全部拷贝一份接续后面的节点。JDK1.8中取消了final修饰，自然就能按照我们的思路，删除一个节点，将上一节点与下一节点联系起来即可。

```java
    public V remove(Object key) {
        return replaceNode(key, null, null);
    }
```

​	JDK1.8中的删除操作似乎是放在replaceNode中实现的，走了一个判断的分支。

```java
    final V replaceNode(Object key, V value, Object cv) {
        int hash = spread(key.hashCode());
        //死循环确保最终是确实去链表或红黑树中尝试替换过
        for (Node<K,V>[] tab = table;;) {
            Node<K,V> f; int n, i, fh;
            //桶数组为空或者桶的首位元素为空自然是说明这个key没有value，直接跳出循环返回null
            if (tab == null || (n = tab.length) == 0 ||
                (f = tabAt(tab, i = (n - 1) & hash)) == null)
                break;
            //和put操作中的理解一样
            else if ((fh = f.hash) == MOVED)
                tab = helpTransfer(tab, f);
            else {
                V oldVal = null;
                boolean validated = false;
                //和加值一样，删值也是更新，得加桶锁
                synchronized (f) {
                    if (tabAt(tab, i) == f) {
                        //链表的情况
                        if (fh >= 0) {
                            validated = true;
                            //循环遍历链表找key
                            for (Node<K,V> e = f, pred = null;;) {
                                K ek;
                                if (e.hash == hash &&
                                    ((ek = e.key) == key ||
                                     (ek != null && key.equals(ek)))) {
                                    V ev = e.val;
                                    if (cv == null || cv == ev ||
                                        (ev != null && cv.equals(ev))) {
                                        oldVal = ev;
                                        //正常的替换，传入的value肯定不是null
                                        if (value != null)
                                            e.val = value;
                                        //非头节点命中删除key的话，pred会记录上一节点，
                                        //直接将上一节点的下一节点设为本节点的下一节点
                                        //完成删除
                                        else if (pred != null)
                                            pred.next = e.next;
                                        //头节点命中key的话，通过setTabAt方法重新设置
                                        //下一节点为头节点，完成删除
                                        else
                                            setTabAt(tab, i, e.next);
                                    }
                                    break;
                                }
                                //该轮未命中，pred记录当前节点作为下一轮的上一节点用
                                pred = e;
                                /*	直到链表最后跳出循环（意思头节点命中也得循环完最多7个节点的
                                 *	链表？），不过因为转换红黑树的存在，性能倒不会有多大损失
                                 */
                                if ((e = e.next) == null)
                                    break;
                            }
                        }
                        //红黑树的情况
                        else if (f instanceof TreeBin) {
                            validated = true;
                            TreeBin<K,V> t = (TreeBin<K,V>)f;
                            TreeNode<K,V> r, p;
                            if ((r = t.root) != null &&
                                (p = r.findTreeNode(hash, key, null)) != null) {
                                V pv = p.val;
                                if (cv == null || cv == pv ||
                                    (pv != null && cv.equals(pv))) {
                                    oldVal = pv;
                                    if (value != null)
                                        p.val = value;
                                    //通过红黑树的方式删除，待展开
                                    else if (t.removeTreeNode(p))
                                        setTabAt(tab, i, untreeify(t.first));
                                }
                            }
                        }
                    }
                }
                //validated保证确实是去链表或是红黑树中找值过
                if (validated) {
                    if (oldVal != null) {
                        //若为删值，addCount方法计数减1
                        if (value == null)
                            addCount(-1L, -1);
                        //替换成功或是删除成功都返回旧值
                        return oldVal;
                    }
                    //validated保证确实是去链表或是红黑树中找值过跳出循环，否则死循环
                    break;
                }
            }
        }
        return null;
    }
```



#### 原子操作方法

​	前面讲述时经常会看到tabAt，casTabAt，setTabAt这样的方法，这些都是桶数组的原子操作，各自发挥着巨大的作用。

##### tabAt方法

```java
static final <K,V> Node<K,V> tabAt(Node<K,V>[] tab, int i) {
    return (Node<K,V>)U.getObjectVolatile(tab, ((long)i << ASHIFT) + ABASE);
}
```
​	这个方法就是读取tab[i]，有人纳闷了，这还得搞得这个复杂，看这个getObjectVolatile也有点复杂啊。回过头我没看ConcurrentHashMap定义的table是用什么修饰的，上面我们提过，volatile，这是用来解决可见性问题的，根据Happens-Before规则，对volatile的写操作Happens-Before对volatile的读操作，因此，这边通过这种方式获取值是可见的。

##### casTabAt

```java
    static final <K,V> boolean casTabAt(Node<K,V>[] tab, int i,
                                        Node<K,V> c, Node<K,V> v) {
        return U.compareAndSwapObject(tab, ((long)i << ASHIFT) + ABASE, c, v);
    }
```

​	执行put方法时，会涉及一个CAS空桶检测，就用到这个方法，实现线程安全的加桶首位值。

##### setTabAt方法

```java
    static final <K,V> void setTabAt(Node<K,V>[] tab, int i, Node<K,V> v) {
        U.putObjectVolatile(tab, ((long)i << ASHIFT) + ABASE, v);
    }
```

​	该方法在replaceNode方法中有涉及，作用就是设置桶的首位节点。



#### 扩容机制

​	在put操作和remove操作时都有涉及一个方法helpTransfer，是检测到正在扩容就帮助扩容，而其中调用了transfer方法就是真正的扩容方法，我们详细来解读下这个方法。

```java
    private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
        int n = tab.length, stride;
        /*	
         *	CPU核数大于1时划分每个核处理数量，桶数组长度/8/核数，若小于16则使用16，
         *	这步主要是均分桶到CPU各个核，避免出现转移任务不均匀的现象
         */
        if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
            stride = MIN_TRANSFER_STRIDE; // subdivide range
        //新的桶数组尚未初始化先初始化
        if (nextTab == null) {            // initiating
            //按2倍扩容
            try {
                @SuppressWarnings("unchecked")
                Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];
                nextTab = nt;
            } catch (Throwable ex) {      // try to cope with OOME
                sizeCtl = Integer.MAX_VALUE;
                return;
            }
            //将初始化的新的桶数组更新到类成员变量
            nextTable = nextTab;
            //记录转移下标，即原本桶数组的长度
            transferIndex = n;
        }
        //记录新桶数组的长度
        int nextn = nextTab.length;
        ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);
        /**
         *	首次推进为true，为ture再次推进一个下标（i--），
         *	若为false，那么不能推进下标，当前下标处理完毕才能推进
         */
        boolean advance = true;
        boolean finishing = false; // to ensure sweep before committing nextTab
        //死循环，i表示下标，bound表示当前线程可以处理的当前桶区间的最小下标
        for (int i = 0, bound = 0;;) {
            Node<K,V> f; int fh;
            while (advance) {
                int nextIndex, nextBound;
                if (--i >= bound || finishing)
                    advance = false;
                else if ((nextIndex = transferIndex) <= 0) {
                    i = -1;
                    advance = false;
                }
                else if (U.compareAndSwapInt
                         (this, TRANSFERINDEX, nextIndex,
                          nextBound = (nextIndex > stride ?
                                       nextIndex - stride : 0))) {
                    bound = nextBound;
                    i = nextIndex - 1;
                    advance = false;
                }
            }
            if (i < 0 || i >= n || i + n >= nextn) {
                int sc;
                if (finishing) {
                    nextTable = null;
                    table = nextTab;
                    sizeCtl = (n << 1) - (n >>> 1);
                    return;
                }
                if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
                    if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
                        return;
                    finishing = advance = true;
                    i = n; // recheck before commit
                }
            }
            else if ((f = tabAt(tab, i)) == null)
                advance = casTabAt(tab, i, null, fwd);
            else if ((fh = f.hash) == MOVED)
                advance = true; // already processed
            else {
                synchronized (f) {
                    if (tabAt(tab, i) == f) {
                        Node<K,V> ln, hn;
                        if (fh >= 0) {
                            int runBit = fh & n;
                            Node<K,V> lastRun = f;
                            for (Node<K,V> p = f.next; p != null; p = p.next) {
                                int b = p.hash & n;
                                if (b != runBit) {
                                    runBit = b;
                                    lastRun = p;
                                }
                            }
                            if (runBit == 0) {
                                ln = lastRun;
                                hn = null;
                            }
                            else {
                                hn = lastRun;
                                ln = null;
                            }
                            for (Node<K,V> p = f; p != lastRun; p = p.next) {
                                int ph = p.hash; K pk = p.key; V pv = p.val;
                                if ((ph & n) == 0)
                                    ln = new Node<K,V>(ph, pk, pv, ln);
                                else
                                    hn = new Node<K,V>(ph, pk, pv, hn);
                            }
                            setTabAt(nextTab, i, ln);
                            setTabAt(nextTab, i + n, hn);
                            setTabAt(tab, i, fwd);
                            advance = true;
                        }
                        else if (f instanceof TreeBin) {
                            TreeBin<K,V> t = (TreeBin<K,V>)f;
                            TreeNode<K,V> lo = null, loTail = null;
                            TreeNode<K,V> hi = null, hiTail = null;
                            int lc = 0, hc = 0;
                            for (Node<K,V> e = t.first; e != null; e = e.next) {
                                int h = e.hash;
                                TreeNode<K,V> p = new TreeNode<K,V>
                                    (h, e.key, e.val, null, null);
                                if ((h & n) == 0) {
                                    if ((p.prev = loTail) == null)
                                        lo = p;
                                    else
                                        loTail.next = p;
                                    loTail = p;
                                    ++lc;
                                }
                                else {
                                    if ((p.prev = hiTail) == null)
                                        hi = p;
                                    else
                                        hiTail.next = p;
                                    hiTail = p;
                                    ++hc;
                                }
                            }
                            ln = (lc <= UNTREEIFY_THRESHOLD) ? untreeify(lo) :
                                (hc != 0) ? new TreeBin<K,V>(lo) : t;
                            hn = (hc <= UNTREEIFY_THRESHOLD) ? untreeify(hi) :
                                (lc != 0) ? new TreeBin<K,V>(hi) : t;
                            setTabAt(nextTab, i, ln);
                            setTabAt(nextTab, i + n, hn);
                            setTabAt(tab, i, fwd);
                            advance = true;
                        }
                    }
                }
            }
        }
    }
```

