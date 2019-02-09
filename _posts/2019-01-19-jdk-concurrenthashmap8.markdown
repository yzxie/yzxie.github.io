---
layout:     post
title:      "JDK源码分析：ConcurrentHashMap（JDK1.8版本）"
subtitle:   "JDK ConcurrentHashMap1.8"
date:       2019-01-18 07:12:00
author:     "XYZ"
header-img: "img/article-bg1.jpg"
tags:
    - JDK源码分析
---
**主文章：**[JDK源码分析：ConcurrentHashMap（JDK1.7和JDK1.8），HashTable与Collections.synchronizedMap](https://blog.csdn.net/u010013573/article/details/86776131)
### 概述
* 在JDK1.7主要通过定义Segment分段锁来实现多个线程对ConcurrentHashMap的数据的并发读写操作。整个ConcurrentHashMap由多个Segment组成，每个Segment保存整个ConcurrentHashMap的一部分数据，Segment结合ReentrantLock，即Segment继承于ReentrantLock，来实现写互斥，读共享，具体为有多少个Segment，则任何时候可以最多支持这么多个线程同时进行写操作，任意多个线程进行读操作。在ConcurrentHashMap的Segment实现中，对写操作使用ReentrantLock来进行加锁，读操作不加锁，通过volatile来实现线程之间的可见性。
* 在JDK1.8中为了进一步优化ConcurrentHashMap的性能，去掉了Segment分段锁的设计，在数据结构方面，则是跟HashMap一样，使用一个哈希表table数组，即数组 + 链表或者数组 + 红黑树。而线程安全方面是结合CAS机制来实现的，CAS底层依赖JDK的UNSAFE所提供的硬件级别的原子操作。同时在HashMap的基础上，对哈希表table数组和链表节点的value，next指针等使用volatile来修饰，从而实现线程可见性。

### 核心字段
* 哈希表table数组，如下与HashMap一样也是使用一个Node类型的数组table来定义的，不同之处是使用volatile修饰该数组。
* baseCount和counterCells：用来记录当前ConcurrentHashMap存在多少个元素使用的，在进行增删链表节点时，默认是更新baseCount的值即可，如果同时存在多个线程并发进行对链表节点的增删操作，则放弃更新baseCount，而是counterCells数组中添加一个CounterCell，之后在计算size的时候，累加baseCount和遍历并累加counterCells。

    ```java
    /* ---------------- Fields -------------- */
    
    /**
     * The array of bins. Lazily initialized upon first insertion.
     * Size is always a power of two. Accessed directly by iterators.
     */
    transient volatile Node<K,V>[] table;
    
    
    /**
     * Base counter value, used mainly when there is no contention,
     * but also as a fallback during table initialization
     * races. Updated via CAS.
     */
    private transient volatile long baseCount;
    
    /**
     * Table of counter cells. When non-null, size is a power of 2.
     */
    private transient volatile CounterCell[] counterCells;
    ```
* 链表节点Node的定义：value，next使用volatile修饰保证线程可见性。

    ```java
    /**
     * Key-value entry.  This class is never exported out as a
     * user-mutable Map.Entry (i.e., one supporting setValue; see
     * MapEntry below), but can be used for read-only traversals used
     * in bulk tasks.  Subclasses of Node with a negative hash field
     * are special, and contain null keys and values (but are never
     * exported).  Otherwise, keys and vals are never null.
     */
    static class Node<K,V> implements Map.Entry<K,V> {
        final int hash;
        final K key;
        volatile V val;
        volatile Node<K,V> next;
    
        Node(int hash, K key, V val, Node<K,V> next) {
            this.hash = hash;
            this.key = key;
            this.val = val;
            this.next = next;
        }
    
        public final K getKey()       { return key; }
        public final V getValue()     { return val; }
     
        ...
        
        /**
         * Virtualized support for map.get(); overridden in subclasses.
         */
        Node<K,V> find(int h, Object k) {
            Node<K,V> e = this;
            if (k != null) {
                do {
                    K ek;
                    if (e.hash == h &&
                        ((ek = e.key) == k || (ek != null && k.equals(ek))))
                        return e;
                } while ((e = e.next) != null);
            }
            return null;
        }
    }
    ```
### 核心方法
#### UNSAFE：硬件级别的原子操作
* 主要定义了获取，更新，添加链表节点Node的方法，具体为基于UNSAFE类提供的硬件级别的原子操作来保证线程安全，而不是通过加锁机制，如synchronized关键字，ReentrantLock重入锁来实现，即无锁化。

    ```java
    /* ---------------- Table element access -------------- */
    
    /*
     * Volatile access methods are used for table elements as well as
     * elements of in-progress next table while resizing.  All uses of
     * the tab arguments must be null checked by callers.  All callers
     * also paranoically precheck that tab's length is not zero (or an
     * equivalent check), thus ensuring that any index argument taking
     * the form of a hash value anded with (length - 1) is a valid
     * index.  Note that, to be correct wrt arbitrary concurrency
     * errors by users, these checks must operate on local variables,
     * which accounts for some odd-looking inline assignments below.
     * Note that calls to setTabAt always occur within locked regions,
     * and so in principle require only release ordering, not
     * full volatile semantics, but are currently coded as volatile
     * writes to be conservative.
     */
    
    @SuppressWarnings("unchecked")
    
    // 原子获取链表节点
    static final <K,V> Node<K,V> tabAt(Node<K,V>[] tab, int i) {
        return (Node<K,V>)U.getObjectVolatile(tab, ((long)i << ASHIFT) + ABASE);
    }
    
    // CAS更新或新增链表节点
    static final <K,V> boolean casTabAt(Node<K,V>[] tab, int i,
                                        Node<K,V> c, Node<K,V> v) {
        return U.compareAndSwapObject(tab, ((long)i << ASHIFT) + ABASE, c, v);
    }
    
    // 原子新增链表节点
    static final <K,V> void setTabAt(Node<K,V>[] tab, int i, Node<K,V> v) {
        U.putObjectVolatile(tab, ((long)i << ASHIFT) + ABASE, v);
    }
    ```
#### put写操作
* 写操作主要在putVal方法定义，实现逻辑与HashMap的putVal基本一致，只是相关操作，如获取链表节点，更新链表节点的值value和新增链表节点，都会使用到UNSAFE提供的硬件级别的原子操作，而如果是更新链表节点的值或者在一个已经存在的链表中新增节点，则是通过synchronized同步锁来实现线程安全性。

    ```java
    /**
     * Maps the specified key to the specified value in this table.
     * Neither the key nor the value can be null.
     *
     * <p>The value can be retrieved by calling the {@code get} method
     * with a key that is equal to the original key.
     *
     * @param key key with which the specified value is to be associated
     * @param value value to be associated with the specified key
     * @return the previous value associated with {@code key}, or
     *         {@code null} if there was no mapping for {@code key}
     * @throws NullPointerException if the specified key or value is null
     */
    public V put(K key, V value) {
        return putVal(key, value, false);
    }
    
    /** Implementation for put and putIfAbsent */
    final V putVal(K key, V value, boolean onlyIfAbsent) {
        if (key == null || value == null) throw new NullPointerException();
        int hash = spread(key.hashCode());
        int binCount = 0;
        for (Node<K,V>[] tab = table;;) {
            Node<K,V> f; int n, i, fh;
            if (tab == null || (n = tab.length) == 0)
                tab = initTable();
            
            // i为该key在table数组的下标
            // null表示该key对应的链表（具体为链表头结点）
            // 在哈希表table中还不存在
            else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            
                // 新增链表头结点，cas方式添加到哈希表table
                if (casTabAt(tab, i, null,
                             new Node<K,V>(hash, key, value, null)))
                    break;                   // no lock when adding to empty bin
            }
            else if ((fh = f.hash) == MOVED)
                tab = helpTransfer(tab, f);
            else {
                V oldVal = null;
                
                // f为链表头结点，使用synchronized加锁
                // 则整条链表则被加锁了
                synchronized (f) {
                
                    // 再次检查，即double check
                    // 即避免进入同步块之前，链表被修改了
                    if (tabAt(tab, i) == f) {
                    
                        // hash值大于0
                        if (fh >= 0) {
                            binCount = 1;
                            for (Node<K,V> e = f;; ++binCount) {
                                K ek;
                                if (e.hash == hash &&
                                    ((ek = e.key) == key ||
                                     (ek != null && key.equals(ek)))) {
                                     
                                    // 节点已经存在，更新value即可
                                    oldVal = e.val;
                                    if (!onlyIfAbsent)
                                        e.val = value;
                                    break;
                                }
                                
                                // 该key对应的节点不存在，
                                // 则新增节点并添加到该链表的末尾
                                Node<K,V> pred = e;
                                if ((e = e.next) == null) {
                                    pred.next = new Node<K,V>(hash, key,
                                                              value, null);
                                    break;
                                }
                            }
                        }
                        
                        // 红黑树节点，
                        // 则往该红黑树更新或添加该节点即可
                        else if (f instanceof TreeBin) {
                            Node<K,V> p;
                            binCount = 2;
                            if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                           value)) != null) {
                                oldVal = p.val;
                                if (!onlyIfAbsent)
                                    p.val = value;
                            }
                        }
                    }
                }
                
                // 判断是否需要将链表转为红黑树
                if (binCount != 0) {
                    if (binCount >= TREEIFY_THRESHOLD)
                        treeifyBin(tab, i);
                    if (oldVal != null)
                        return oldVal;
                    break;
                }
            }
        }
        
        // 递增ConcurrentHashMap的节点总个数
        addCount(1L, binCount);
        return null;
    }
    ```
    1. 如果当前需要put的key对应的链表在哈希表table中还不存在，即还没添加过该key的hash值对应的链表，则调用UNSAFE的casTabAt方法，基于CAS机制来实现添加该链表头结点到哈希表table中，避免该线程在添加该链表头结的时候，其他线程也在添加的并发问题；
    2. 如果需要添加的链表已经存在哈希表table中，则通过UNSAFE的tabAt方法，基于volatile机制，获取当前最新的链表头结点f，由于f指向的是ConcurrentHashMap的哈希表table的某条链表的头结点，故虽然f是临时变量，由于是引用共享的该链表头结点，所以可以使用synchronized关键字来同步多个线程对该链表的访问。在synchronized(f)同步块里面，则是与HashMap一样遍历该链表，如果该key对应的链表节点已经存在，则更新，否则在链表的末尾新增该key对应的链表节点。

* 使用synchronized同步锁的原因：因为如果该key对应的节点所在的链表已经存在的情况下，可以通过UNSAFE的tabAt方法基于volatile获取到该链表最新的头节点，但是需要通过遍历该链表来判断该节点是否存在，如果不使用synchronized对链表头结点进行加锁，则在遍历过程中，其他线程可能会添加这个节点，导致重复添加的并发问题。故通过synchronized锁住链表头结点的方式，保证任何时候只存在一个线程对该链表进行更新操作。
* 锁的范围缩小：相对于JDK1.7的Segment分段锁，即分段锁的写操作，在操作之前需要先获取lock锁，即不管是（1）链表不存在，添加链表头结点，（2）还是更新链表节点，（3）还是在已经存在的链表中添加节点，都需要先获取lock锁，而在JDK1.8的写操作中，（1）如果该链表不存在，添加链表头的时候是不需要加锁的，因为是往哈希表table数组的某个位置填充值，不需要遍历链表之类的，所以可以基于UNSAFE的casTabAt方法，即CAS机制检查table数组的该位置是否存在元素（链表头结点）来实现线程安全，这是写操作最先检查的；如果该链表已经存在，即（2）（3）则需要通过synchronized来锁住该链表头结点，所以这里也是一个性能提升的地方，缩小了锁的范围。

#### get读操作
* get读操作由于是从哈希表中查找并读取链表节点数据，不会对链表进行写更新操作，故基于volatile的happend-before原则保证的线程可见性（即一个线程的操作对其他线程可见），即可保证get读取到该key对应的最新链表节点，整个过程不需要进行加锁。
* 具体为table和Node的value和next均是volatile修饰。

    ```java
    /**
     * Returns the value to which the specified key is mapped,
     * or {@code null} if this map contains no mapping for the key.
     *
     * <p>More formally, if this map contains a mapping from a key
     * {@code k} to a value {@code v} such that {@code key.equals(k)},
     * then this method returns {@code v}; otherwise it returns
     * {@code null}.  (There can be at most one such mapping.)
     *
     * @throws NullPointerException if the specified key is null
     */
    public V get(Object key) {
        Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
        int h = spread(key.hashCode());
        
        // 获取链表头结点
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (e = tabAt(tab, (n - 1) & h)) != null) {
            
            // 头结点的key相等说明找到了，直接返回
            if ((eh = e.hash) == h) {
                if ((ek = e.key) == key || (ek != null && key.equals(ek)))
                    return e.val;
            }
            else if (eh < 0)
                return (p = e.find(h, key)) != null ? p.val : null;
            
            // 遍历该链表，查找对应的节点
            while ((e = e.next) != null) {
                if (e.hash == h &&
                    ((ek = e.key) == key || (ek != null && key.equals(ek))))
                    return e.val;
            }
        }
        return null;
    }
    ```

#### size容量计算
* size方法为计算当前ConcurrentHashMap一共存在多少链表节点，与JDK1.7中每次需要遍历segments数组来计算不同的是，在JDK1.8中，使用baseCount和counterCells数组，在增删链表节点时，实时更新来统计，在size方法中直接返回即可。整个过程不需要加锁。
* 并发修改异常处理：CounterCell的value值为1，作用是某个线程在更新baseCount时，如果存在其他线程同时在更新，则放弃更新baseCount的值，即保持baseCount不变，而是各自往counterCells数组添加一个counterCell元素，在size方法中，累加counterCells数组的value，然后与baseCount相加，从而获取准确的大小。
    ```java
    /**
     * {@inheritDoc}
     */
    public int size() {
        long n = sumCount();
        return ((n < 0L) ? 0 :
                (n > (long)Integer.MAX_VALUE) ? Integer.MAX_VALUE :
                (int)n);
    }
    
    /* ---------------- Counter support -------------- */
    
    /**
     * A padded cell for distributing counts.  Adapted from LongAdder
     * and Striped64.  See their internal docs for explanation.
     */
    @sun.misc.Contended static final class CounterCell {
        volatile long value;
        CounterCell(long x) { value = x; }
    }
    
    final long sumCount() {
        CounterCell[] as = counterCells; CounterCell a;
        
        // sum初始化为baseCount
        long sum = baseCount;
        if (as != null) {
        
            // 遍历counterCells并累加其value到sum
            for (int i = 0; i < as.length; ++i) {
                if ((a = as[i]) != null)
                    sum += a.value;
            }
        }
        return sum;
    }
    ```
* addCount：在put写操作之后，递增baseCount值。在putVal中调用addCount(1L, binCount);，即递增1，在其批量操作中，则可以是批量的数量作为参数，如addCount(100L, binCount)

    ```java
    /**
     * Adds to count, and if table is too small and not already
     * resizing, initiates transfer. If already resizing, helps
     * perform transfer if work is available.  Rechecks occupancy
     * after a transfer to see if another resize is already needed
     * because resizings are lagging additions.
     *
     * @param x the count to add
     * @param check if <0, don't check resize, if <= 1 only check if uncontended
     */
    private final void addCount(long x, int check) {
        CounterCell[] as; long b, s;
        if ((as = counterCells) != null ||
        
            // CAS更新baseCount失败，表示存在并发异常
            !U.compareAndSwapLong(this, BASECOUNT, b = baseCount, s = b + x)) {
            CounterCell a; long v; int m;
            boolean uncontended = true;
            if (as == null || (m = as.length - 1) < 0 ||
                (a = as[ThreadLocalRandom.getProbe() & m]) == null ||
                !(uncontended =
                  U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))) {
                  
                // CAS更新失败时，在counterCells数组添加一个counterCell对象
                fullAddCount(x, uncontended);
                return;
            }
            if (check <= 1)
                return;
            s = sumCount();
        }
        if (check >= 0) {
            Node<K,V>[] tab, nt; int n, sc;
            while (s >= (long)(sc = sizeCtl) && (tab = table) != null &&
                   (n = tab.length) < MAXIMUM_CAPACITY) {
                int rs = resizeStamp(n);
                if (sc < 0) {
                    if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                        sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                        transferIndex <= 0)
                        break;
                    if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                        transfer(tab, nt);
                }
                else if (U.compareAndSwapInt(this, SIZECTL, sc,
                                             (rs << RESIZE_STAMP_SHIFT) + 2))
                    transfer(tab, null);
                s = sumCount();
            }
        }
    }
    
    // 往counterCells数组添加counterCell对象
    // See LongAdder version for explanation
    private final void fullAddCount(long x, boolean wasUncontended) {
        int h;
        if ((h = ThreadLocalRandom.getProbe()) == 0) {
            ThreadLocalRandom.localInit();      // force initialization
            h = ThreadLocalRandom.getProbe();
            wasUncontended = true;
        }
        boolean collide = false;                // True if last slot nonempty
        for (;;) {
            CounterCell[] as; CounterCell a; int n; long v;
            if ((as = counterCells) != null && (n = as.length) > 0) {
                if ((a = as[(n - 1) & h]) == null) {
                    if (cellsBusy == 0) {            // Try to attach new Cell
                        CounterCell r = new CounterCell(x); // Optimistic create
                        if (cellsBusy == 0 &&
                            U.compareAndSwapInt(this, CELLSBUSY, 0, 1)) {
                            boolean created = false;
                            try {               // Recheck under lock
                                CounterCell[] rs; int m, j;
                                if ((rs = counterCells) != null &&
                                    (m = rs.length) > 0 &&
                                    rs[j = (m - 1) & h] == null) {
                                    rs[j] = r;
                                    created = true;
                                }
                            } finally {
                                cellsBusy = 0;
                            }
                            if (created)
                                break;
                            continue;           // Slot is now non-empty
                        }
                    }
                    collide = false;
                }
                else if (!wasUncontended)       // CAS already known to fail
                    wasUncontended = true;      // Continue after rehash
                else if (U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))
                    break;
                else if (counterCells != as || n >= NCPU)
                    collide = false;            // At max size or stale
                else if (!collide)
                    collide = true;
                else if (cellsBusy == 0 &&
                         U.compareAndSwapInt(this, CELLSBUSY, 0, 1)) {
                    try {
                        if (counterCells == as) {// Expand table unless stale
                            CounterCell[] rs = new CounterCell[n << 1];
                            for (int i = 0; i < n; ++i)
                                rs[i] = as[i];
                            counterCells = rs;
                        }
                    } finally {
                        cellsBusy = 0;
                    }
                    collide = false;
                    continue;                   // Retry with expanded table
                }
                h = ThreadLocalRandom.advanceProbe(h);
            }
            else if (cellsBusy == 0 && counterCells == as &&
                     U.compareAndSwapInt(this, CELLSBUSY, 0, 1)) {
                boolean init = false;
                try {                           // Initialize table
                    if (counterCells == as) {
                        CounterCell[] rs = new CounterCell[2];
                        rs[h & 1] = new CounterCell(x);
                        counterCells = rs;
                        init = true;
                    }
                } finally {
                    cellsBusy = 0;
                }
                if (init)
                    break;
            }
            else if (U.compareAndSwapLong(this, BASECOUNT, v = baseCount, v + x))
                break;                          // Fall back on using base
        }
    }
    ```