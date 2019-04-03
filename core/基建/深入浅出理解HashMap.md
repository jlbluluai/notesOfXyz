参考：



### 前言

​	HashMap的重要性就不需要再说了吧，前一段时间也整理了下ConcurrentHashMap，回过头又关注了下HashMap，之前把它放在集合整理的一个大块里，难免整理上不会那么精细，该篇便专门整理下HashMap。



### 背景

​	在早期Java就已经提供了一种哈希表的实现方式，就是HashTable，并且这个HashTable还是同步的，这个在ConcurrentHashMap中有讲，两者解决并发安全的性能差异在哪。而HashMap则是现在最为广泛使用的哈希表的实现，不论是put还是get都能达到常数级别时间的性能。



### HashMap基本数据结构

- transient Node<K,V>[] table;键值对桶数组
- transient int size;键值对的数量
- final float loadFactor;负载系数
- static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; 默认的初始化的容量16
- static final int MAXIMUM_CAPACITY = 1 << 30;支持的最大的容量2的30次方
- static final float DEFAULT_LOAD_FACTOR = 0.75f;负载系数默认值
- static final int TREEIFY_THRESHOLD = 8;链表转红黑树的阈值，和概念说的8一致
- static final int UNTREEIFY_THRESHOLD = 6;红黑树转链表的阈值，ConcurrentHashMap解读代码时也是6
- static final int MIN_TREEIFY_CAPACITY = 64;红黑树最小容量
- int threshold;容量×负载系数，默认第一次为16*0.75f，超过这个值就扩容
- transient int modCount;记录map的修改次数



#### 键值对内部类Node<K,V>

```java
static class Node<K,V> implements Map.Entry<K,V> {
        final int hash;
        final K key;
    	//区别于ConcurrentHashMap，HashMap不是线程安全的，自然也不需要volatile保证啥可见性
        V value;
        Node<K,V> next;
    	...
}
```



### HashMap重要方法分析

​	我们就直接看空构造函数。

```java
    public HashMap() {
        this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
    }
```

​	初始化不会在这边做，和ConcurrentHashMap中一样，不过 HashMap这边还是做了一个操作的，空构造函数的话，是设置负载系数为默认值，也就是0.75f。



#### put方法

```java
    public V put(K key, V value) {
        return putVal(hash(key), key, value, false, true);
    }
```

​	也是调的putVal方法，直接去看。

```java
    final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        //tab当前桶数组，n为当前桶数组长度
        Node<K,V>[] tab; Node<K,V> p; int n, i;
        /**
         *	若当前桶数组为null或是长度为0执行扩容，
         *	空构造函数未初始化桶数组就会在这边resize中进行，后面会讲解resize
         */
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;
        
        /**
         *	计算当前hash值应分配在桶数组哪位，并判断该位是否为null，
         *	如果为null则直接创建一个Node节点（给这个桶开个头了），结束
         *	为啥用（n-1）&hash确定桶位置呢？
         *	n是2的n次方，假设为16，也就是10000，n-1就是15即01111，
         *	任何值与01111做与运算必然范围为[0,15],这样做了一个巧妙的分配
         */
        if ((p = tab[i = (n - 1) & hash]) == null)
            tab[i] = newNode(hash, key, value, null);
        //不为null的话就开始下面一大串的操作
        else {
            Node<K,V> e; K k;
            //若直接匹配头节点，就省事了，不动
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;
            /**
             *	如果当前桶是树结构，则调用putTreeVal方法去加值，若是key存在覆盖返回旧值，
             *	也就是e不为null，若插入成功返回null
             */
            else if (p instanceof TreeNode)
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            //链表插值的情况
            else {
                //循环直到末端break出来
                for (int binCount = 0; ; ++binCount) {
                    //循环到末端，新建一个节点
                    if ((e = p.next) == null) {
                        p.next = newNode(hash, key, value, null);
                        //若是加完长度大于等于8则执行treeifyBin转为红黑树
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            treeifyBin(tab, hash);
                        break;
                    }
                    //若是匹配到存在的，直接跳出
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    p = e;
                }
            }
            /**
             *	e不为null的情况就是key原本就存在，于是将新值替换旧值并返回旧值（给你提个醒，0.0）
             */
            if (e != null) { // existing mapping for key
                V oldValue = e.value;
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                afterNodeAccess(e);
                return oldValue;
            }
        }
        //若是新增结点而非覆盖结点，就会执行这边最后几句代码
        ++modCount;//加一个结点属于修改，计数加一
        /**
         *	扩容阈值，容量*负载系数（默认0.75f），
         *	若容量大于该阈值后就执行扩容
         *	有一点我没搞懂，比如桶数组长度16，每个桶都有链表，红黑树，为啥以键值对的数量size和
         *	长度*负载系数来判断是否扩容，感觉这样的话，比如链表长度起来了，桶数组本身不是有大量null吗？
         *	
         *	其实可以这么想，数组其实就代表了map的容量，null，链表，红黑树怎么解释呢，
         *	想想数组和链表的优劣，简单说呢，数组查询效率高，链表更新效率高，但是HashMap可是说是
         *	无论查询还是更新都是常数实际级别的呢，这就是这边桶数组和链表结合的原因，数组根据hash值
         *	快速定位小范围，然后再在链表上找，这样不论查询还是更新都有了折中，至于红黑树是JDK1.8加入的，
         *	补充链表长度过长时效率降低。
         *	所以可以类比的说，本来值都是存在数组上，但是为了提升效率，就把他们分配到了其它下标位组成
         *	链表型式存储，所以大量的null空缺是可以记账到存在的结点上，这样就可以理解了。
         *	就是兼顾数组的查询效率和链表的更新效率。
         */
        if (++size > threshold)
            resize();
        afterNodeInsertion(evict);
        return null;//插入成功的情况就返回null
    }
```

​	既然代码将对红黑树的插入单独出在putTreeVal方法中，我们就去一探究竟，这个方法在内部类TreeNode中。

```java
      final TreeNode<K,V> putTreeVal(HashMap<K,V> map, Node<K,V>[] tab,
                                       int h, K k, V v) {
            Class<?> kc = null;
            boolean searched = false;
            TreeNode<K,V> root = (parent != null) ? root() : this;
          	//从树根结点开始遍历红黑树
            for (TreeNode<K,V> p = root;;) {
                int dir, ph; K pk;
                if ((ph = p.hash) > h)
                    dir = -1;
                else if (ph < h)
                    dir = 1;
                else if ((pk = p.key) == k || (k != null && k.equals(pk)))
                    return p;
                else if ((kc == null &&
                          (kc = comparableClassFor(k)) == null) ||
                         (dir = compareComparables(kc, k, pk)) == 0) {
                    if (!searched) {
                        TreeNode<K,V> q, ch;
                        searched = true;
                        if (((ch = p.left) != null &&
                             (q = ch.find(h, k, kc)) != null) ||
                            ((ch = p.right) != null &&
                             (q = ch.find(h, k, kc)) != null))
                            return q;
                    }
                    dir = tieBreakOrder(k, pk);
                }

                TreeNode<K,V> xp = p;
                if ((p = (dir <= 0) ? p.left : p.right) == null) {
                    Node<K,V> xpn = xp.next;
                    TreeNode<K,V> x = map.newTreeNode(h, k, v, xpn);
                    if (dir <= 0)
                        xp.left = x;
                    else
                        xp.right = x;
                    xp.next = x;
                    x.parent = x.prev = xp;
                    if (xpn != null)
                        ((TreeNode<K,V>)xpn).prev = x;
                    moveRootToFront(tab, balanceInsertion(root, x));
                    return null;
                }
            }
        }
```



#### resize方法

​	这是HashMap的扩容方法，这就没有ConcurrentHashMap那么多花花肠子了，不论是代码篇幅还是理解难度上都简单许多，毕竟不用考虑并发安全性的问题。

```java
    final Node<K,V>[] resize() {
        //记录当前桶数组
        Node<K,V>[] oldTab = table;
        //记录当前桶数组长度
        int oldCap = (oldTab == null) ? 0 : oldTab.length;
       	//记录当前临界值
        int oldThr = threshold;
        //新容量。新临界值
        int newCap, newThr = 0;
        //若是当前桶数组长度大于0，则执行resize是真正的扩容
        if (oldCap > 0) {
            //若是桶数组长度大于等于最大值容量，2的30次方，设置临界值为Integer最大值并直接返回原桶数组
            //即不予以扩容
            if (oldCap >= MAXIMUM_CAPACITY) {
                threshold = Integer.MAX_VALUE;
                return oldTab;
            }
            //否则新容量扩充为原本的2倍，新临界值也扩充为原本的2倍
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                     oldCap >= DEFAULT_INITIAL_CAPACITY)
                newThr = oldThr << 1; // double threshold
        }
        //否则若是临界值大于0，说明创建HashMap是调用的有参构造函数（会先算好临界值），初始化
        else if (oldThr > 0) // initial capacity was placed in threshold
            //容量设为临界值（？）
            newCap = oldThr;
        //最后一种情况就是调用的无参构造函数创建的HashMap，初始化
        else {               // zero initial threshold signifies using defaults
            //容量设置为默认初始化容量，16
            newCap = DEFAULT_INITIAL_CAPACITY;
            //临界值就是16*0.75f
            newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
        }
        //出现这种情况就是上面第二个分支，计算这种方式的临界值
        if (newThr == 0) {
            float ft = (float)newCap * loadFactor;
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                      (int)ft : Integer.MAX_VALUE);
        }
        //更新临界值
        threshold = newThr;
        //以newCap为数组大小创建桶数组
        @SuppressWarnings({"rawtypes","unchecked"})
            Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
        //更新桶数组，不过这时只有容量，也就是空的
        table = newTab;
        //确认不是初始化，而是真正的扩容，正式开始扩容
        if (oldTab != null) {
            //遍历老桶数组
            for (int j = 0; j < oldCap; ++j) {
                Node<K,V> e;
                //该处桶不为空才继续
                if ((e = oldTab[j]) != null) {
                    //桶置空（链表还是红黑树已经给e了）
                    oldTab[j] = null;
                    //若是只有头，直接计算新的桶位置将e赋值
                    if (e.next == null)
                        newTab[e.hash & (newCap - 1)] = e;
                    //若是红黑树，通过TreeNode的split定位新的桶位置
                    else if (e instanceof TreeNode)
                        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                    //超过1长度的链表，处理方式类似ConcurrentHashMap中分出高低链表
                    else { // preserve order
                        Node<K,V> loHead = null, loTail = null;
                        Node<K,V> hiHead = null, hiTail = null;
                        Node<K,V> next;
                        do {
                            next = e.next;
                            /**
                             *	这种判断方法ConcurrentHashMap中也有类似的，
                             *	长度是2的n次方，二进制就是00010000这样的数字，做与运算,
                             *	不论hash值，结果只有两种，0和长度本身，所以这边判断分支是这么个意思
                             *	为0时放入低位链表
                             *	不为0时放入高位链表
                             *	为何扩容后要这么分配呢？那显然不是随心所欲的。
                             *	以容量16举例，原本加值取值时是hash&(1111)确定位置，
                             *	现在扩容一倍为32，所以现在是hash&(11111)确定位置
                             *
                             *	与10000做与运算结果0的hash有什么特点呢，高位不谈，运算位
                             *	区间[00000,01111],这样区间的值与1111和11111运算结果是一致的，
                             *	所以称为低位链表，放在新桶数组的原下标处即可
                             *
                             *	与10000做与运算的结果不为0的hash有什么特点呢，运算位区间
                             *	[10000,11111]，这样区间的值与1111和11111运算结果正好相差个10000
                             *	也就是16，称作高位链表，放在新桶数组的原下标加旧数组长度下标处
                             */
                            if ((e.hash & oldCap) == 0) {
                                if (loTail == null)
                                    loHead = e;
                                else
                                    loTail.next = e;
                                loTail = e;
                            }
                            else {
                                if (hiTail == null)
                                    hiHead = e;
                                else
                                    hiTail.next = e;
                                hiTail = e;
                            }
                        } while ((e = next) != null);
                        //原位置处放低位链表
                        if (loTail != null) {
                            loTail.next = null;
                            newTab[j] = loHead;
                        }
                        //原位置+原数组长度处放高位链表
                        if (hiTail != null) {
                            hiTail.next = null;
                            newTab[j + oldCap] = hiHead;
                        }
                    }
                }
            }
        }
        return newTab;
    }
```



#### get方法

​	有了加值自然要取值了，看看HashMap的get长啥样。

```java
    public V get(Object key) {
        Node<K,V> e;
        return (e = getNode(hash(key), key)) == null ? null : e.value;
    }
```

​	看来取值的最终方法是getNode。

```java
    final Node<K,V> getNode(int hash, Object key) {
        Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
        //验证桶数组是否还未初始化以及key的hash值对应的桶位置是否为空
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (first = tab[(n - 1) & hash]) != null) {
            //判断第一位是否匹配
            if (first.hash == hash && // always check first node
                ((k = first.key) == key || (key != null && key.equals(k))))
                return first;
            //若有后续结点，循环
            if ((e = first.next) != null) {
                //若是红黑树，通过TreeNode的getTreeNode方法取得
                if (first instanceof TreeNode)
                    return ((TreeNode<K,V>)first).getTreeNode(hash, key);
                //遍历链表，匹配
                do {
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        return e;
                } while ((e = e.next) != null);
            }
        }
        return null;//未能匹配返回null
    }
```

​	顺便把getTreeNode方法也看了。

```java
        final TreeNode<K,V> getTreeNode(int h, Object k) {
            //父结点不为空，执行root方法得到根结点，否则this就是根结点，根节点调用find去匹配
            return ((parent != null) ? root() : this).find(h, k, null);
        }
```

​	......HashMap城会玩，看看find。

```java
        final TreeNode<K,V> find(int h, Object k, Class<?> kc) {
            //记住当前结点，假设当前结点为根结点
            TreeNode<K,V> p = this;
            //循环
            do {
                int ph, dir; K pk;
                TreeNode<K,V> pl = p.left, pr = p.right, q;
                //当前结点的hash值大于参数hash值，转到左孩子结点
                if ((ph = p.hash) > h)
                    p = pl;
                //当前结点的hash值小于参数hash值，转到右孩子结点
                else if (ph < h)
                    p = pr;
                //走到这一步，当前结点hash值必然等于参数hash值，若是key匹配，返回当前结点
                else if ((pk = p.key) == k || (k != null && k.equals(pk)))
                    return p;
                //若匹配到hash值还未匹配到key（hash值是可能重复的）
                //若左孩子为null，转到右孩子
                else if (pl == null)
                    p = pr;
                //若右孩子为null，转到左孩子
                else if (pr == null)
                    p = pl;
                //左右孩子都不为null的情况，kc是啥？
                else if ((kc != null ||
                          (kc = comparableClassFor(k)) != null) &&
                         (dir = compareComparables(kc, k, pk)) != 0)
                    p = (dir < 0) ? pl : pr;
                //直接先用右孩子调用find方法，若不为null，则直接返回
                else if ((q = pr.find(h, k, kc)) != null)
                    return q;
                //否则进入当前结点的左孩子继续循环（还是左孩子亲啊，自己循环找）
                else
                    p = pl;
            } while (p != null);
            return null;
        }
```



#### remove方法

​	这就该讲删除了。

```
    public V remove(Object key) {
        Node<K,V> e;
        return (e = removeNode(hash(key), key, null, false, true)) == null ?
            null : e.value;
    }
```

```java
    final Node<K,V> removeNode(int hash, Object key, Object value,
                               boolean matchValue, boolean movable) {
        Node<K,V>[] tab; Node<K,V> p; int n, index;
        //若是桶数组未初始化，或是hash值匹配的坐标处为空，直接返回null，不存在该key-value
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (p = tab[index = (n - 1) & hash]) != null) {
            Node<K,V> node = null, e; K k; V v;
            //若是头结点匹配
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                node = p;
            //循环匹配
            else if ((e = p.next) != null) {
                if (p instanceof TreeNode)
                    node = ((TreeNode<K,V>)p).getTreeNode(hash, key);
                else {
                    do {
                        if (e.hash == hash &&
                            ((k = e.key) == key ||
                             (key != null && key.equals(k)))) {
                            node = e;
                            break;
                        }
                        p = e;
                    } while ((e = e.next) != null);
                }
            }
            //若是匹配到则在这一步进行删除
            if (node != null && (!matchValue || (v = node.value) == value ||
                                 (value != null && value.equals(v)))) {
                //调用红黑树的删除方式
                if (node instanceof TreeNode)
                    ((TreeNode<K,V>)node).removeTreeNode(this, tab, movable);
                //头结点匹配，直接将下一结点覆盖入当前桶数组位置即可
                else if (node == p)
                    tab[index] = node.next;
                //非头结点匹配
                else
                    p.next = node.next;
                ++modCount;//删除也涉及修改了，计数加一
                --size;//键值对数量减1
                afterNodeRemoval(node);
                return node;
            }
        }
        return null;
    }
```

