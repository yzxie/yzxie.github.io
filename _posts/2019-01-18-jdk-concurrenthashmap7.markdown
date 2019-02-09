---
layout:     post
title:      "JDK源码分析：ConcurrentHashMap（JDK1.7版本）"
subtitle:   "JDK ConcurrentHashMap1.7"
date:       2019-01-18 07:12:00
author:     "XYZ"
header-img: "img/article-bg1.jpg"
tags:
    - JDK源码分析
---
**主文章：**[JDK源码分析：ConcurrentHashMap（JDK1.7和JDK1.8），HashTable与Collections.synchronizedMap
](https://blog.csdn.net/u010013573/article/details/86776131)
#### Segment：分段锁
* 在HashMap中，是使用一个哈希表，即元素为链表结点Node组成的数组table，而在ConcurrentHashMap中是使用多个哈希表，具体为通过定义一个Segment来封装这个哈希表其中Segment继承于ReentrantLock，故自带lock的功能。即每个Segment其实就是相当于一个HashMap，只是结合使用了ReentrantLock来进行并发控制，实现线程安全。
* Segment定义如下：ConcurrentHashMap的一个静态内部类，继承于ReentrantLock，在内部定义了一个HashEntry数组table，HashEntry是链表的节点定义，其中table使用volatile修饰，保证某个线程对table进行新增链表节点（头结点或者在已经存在的链表新增一个节点）对其他线程可见。

    ```java
    /**
     * Segments are specialized versions of hash tables.  This
     * subclasses from ReentrantLock opportunistically, just to
     * simplify some locking and avoid separate construction.
     */
    static final class Segment<K,V> extends ReentrantLock implements Serializable {
        
        ...
        
        /**
         * The maximum number of times to tryLock in a prescan before
         * possibly blocking on acquire in preparation for a locked
         * segment operation. On multiprocessors, using a bounded
         * number of retries maintains cache acquired while locating
         * nodes.
         */
        static final int MAX_SCAN_RETRIES =
            Runtime.getRuntime().availableProcessors() > 1 ? 64 : 1;
    
        /**
         * The per-segment table. Elements are accessed via
         * entryAt/setEntryAt providing volatile semantics.
         */
        transient volatile HashEntry<K,V>[] table;
    
        ...
        
    }
    ```
* HashEntry的定义如下：包含key，value，key的hash，所在链表的下一个节点next。

    ```java
    /**
     * ConcurrentHashMap list entry. Note that this is never exported
     * out as a user-visible Map.Entry.
     */
    static final class HashEntry<K,V> {
        final int hash;
        final K key;
        volatile V value;
        volatile HashEntry<K,V> next;
        
        ...
        
    }
    ```
   由定义可知，value和next均为使用volatile修饰，当多个线程共享该HashEntry所在的Segment时，其中一个线程对该Segment内部的某个链表节点HashEntry的value或下一个节点next修改能够对其他线程可见。而hash和key均使用final修饰，因为创建一个链表节点HashEntry，是根据key的hash值来确定附加到哈希表数组table的某个链表的，即根据hash计算对应的table数组的下标，故一旦添加后是不可变的。

##### Segment的哈希表table数组的容量
* MIN_SEGMENT_TABLE_CAPACITY：table数组的容量最小量，默认为2

    ```java
    /**
     * The minimum capacity for per-segment tables.  Must be a power
     * of two, at least two to avoid immediate resizing on next use
     * after lazy construction.
     */
    static final int MIN_SEGMENT_TABLE_CAPACITY = 2;
    ```
* 在ConcurrentHashMap的构造函数定义实际大小：使用ConcurrentHashMap的整体容量initialCapacity除以Segments数组的大小，得到每个Segment内部的table数组的实际大小。

    ```java
    public ConcurrentHashMap(int initialCapacity,
                             float loadFactor, int concurrencyLevel) {
        if (!(loadFactor > 0) || initialCapacity < 0 || concurrencyLevel <= 0)
            throw new IllegalArgumentException();
        if (concurrencyLevel > MAX_SEGMENTS)
            concurrencyLevel = MAX_SEGMENTS;
        // Find power-of-two sizes best matching arguments
        int sshift = 0;
        int ssize = 1;
        
        // ssize：segments数组的大小
        // 不能小于concurrencyLevel，默认为16
        while (ssize < concurrencyLevel) {
            ++sshift;
            ssize <<= 1;
        }
        this.segmentShift = 32 - sshift;
        this.segmentMask = ssize - 1;
        if (initialCapacity > MAXIMUM_CAPACITY)
            initialCapacity = MAXIMUM_CAPACITY;
        int c = initialCapacity / ssize;
        if (c * ssize < initialCapacity)
            ++c;
            
        // cap：Segment内部HashEntry数组的大小
        // 最小为MIN_SEGMENT_TABLE_CAPACITY，默认为2
        // 实际大小根据c来计算，而c是由上面代码，
        // 根据initialCapacity / ssize得到，
        // 即整体容量大小除以Segment数组的数量，则
        // 得到每个Segment内部的table的大小
        int cap = MIN_SEGMENT_TABLE_CAPACITY;
        while (cap < c)
            cap <<= 1;
        // create segments and segments[0]
        Segment<K,V> s0 =
            new Segment<K,V>(loadFactor, (int)(cap * loadFactor),
                             (HashEntry<K,V>[])new HashEntry[cap]);
        Segment<K,V>[] ss = (Segment<K,V>[])new Segment[ssize];
        UNSAFE.putOrderedObject(ss, SBASE, s0); // ordered write of segments[0]
        this.segments = ss;
    }
    ```

#### 并发性控制
* ConcurrentHashMap通过多个Segment，即内部包含一个Segment数组，来支持多个线程同时分别对这些Segment进行读写操作，从而提高并发性。
* 并发性：默认通过DEFAULT_CONCURRENCY_LEVEL来定义Segment的数量，默认为16，即创建大小为16的Segment数组，这样在任何时刻最多可以支持16个线程同时对ConcurrentHashMap进行写操作，即每个Segment都可以有一个线程在进行写操作。也可以在ConcurrentHashMap的构造函数中指定concurrencyLevel的值。
* MAX_SEGMENTS：定义Segment数组的容量最大值，默认值为2的16次方。

    ```java
    /**
     * The default concurrency level for this table, used when not
     * otherwise specified in a constructor.
     */
    static final int DEFAULT_CONCURRENCY_LEVEL = 16;
    
     /**
     * The maximum number of segments to allow; used to bound
     * constructor arguments. Must be power of two less than 1 << 24.
     */
    static final int MAX_SEGMENTS = 1 << 16; // slightly conservative
    ```
#### put写操作
* 首先通过key的hash确定segments数组的下标，即需要往哪个segment存放数据。确定好segment之后，则调用该segment的put方法，写到该segment内部的哈希表table数组的某个链表中，链表的确定也是根据key的hash值和segment内部table大小取模。
* 在ConcurrentHashMap中的put操作是没有加锁的，而在Segment中的put操作，通过ReentrantLock加锁：

    ```java
    /**
     * Maps the specified key to the specified value in this table.
     * Neither the key nor the value can be null.
     *
     * <p> The value can be retrieved by calling the <tt>get</tt> method
     * with a key that is equal to the original key.
     *
     * @param key key with which the specified value is to be associated
     * @param value value to be associated with the specified key
     * @return the previous value associated with <tt>key</tt>, or
     *         <tt>null</tt> if there was no mapping for <tt>key</tt>
     * @throws NullPointerException if the specified key or value is null
     */
    @SuppressWarnings("unchecked")
    public V put(K key, V value) {
        Segment<K,V> s;
        if (value == null)
            throw new NullPointerException();
        int hash = hash(key);
        int j = (hash >>> segmentShift) & segmentMask;
        
        // 根据key的hash只，确定具体的Segment
        if ((s = (Segment<K,V>)UNSAFE.getObject          // nonvolatile; recheck
             (segments, (j << SSHIFT) + SBASE)) == null) //  in ensureSegment
            
            // 如果segments数组的该位置还没segment
            // 则数组下标j对应的segment实例
            s = ensureSegment(j);
        
        // 往该segment实例设值
        return s.put(key, hash, value, false);
    }
    ```
* Segment类的put操作定义：首先获取lock锁，然后根据key的hash值，获取在segment内部的HashEntry数组table的下标，从而获取对应的链表，具体为链表头。
    
    ```java
    final V put(K key, int hash, V value, boolean onlyIfAbsent) {
        
        // tryLock：非阻塞获取lock，如果当前没有其他线程持有该Segment的锁，则返回null，继续往下执行；
        // scanAndLockForPut：该segment锁被其他线程持有了，则非阻塞重试3次，超过3次则阻塞等待锁。之后返回对应的链表节点。
        HashEntry<K,V> node = tryLock() ? null :
            scanAndLockForPut(key, hash, value);
        V oldValue;
        try {
            HashEntry<K,V>[] tab = table;
            int index = (tab.length - 1) & hash;
            
            // 链表头结点
            HashEntry<K,V> first = entryAt(tab, index);
            for (HashEntry<K,V> e = first;;) {
            
                // 已经存在，则更新value值
                if (e != null) {
                    K k;
                    if ((k = e.key) == key ||
                        (e.hash == hash && key.equals(k))) {
                        oldValue = e.value;
                        if (!onlyIfAbsent) {
                            e.value = value;
                            // 更新value时，也递增modCount，而在HashMap中是结构性修改才递增。
                            ++modCount;
                        }
                        break;
                    }
                    e = e.next;
                }
                else {
                    // 注意新增节点时，是在头部添加的，即最后添加的节点是链表头结点
                    // 这个与HashMap是不一样的，HashMap是在链表尾部新增节点。
                    if (node != null)
                        node.setNext(first);
                    else
                        node = new HashEntry<K,V>(hash, key, value, first);
                    int c = count + 1;
                    if (c > threshold && tab.length < MAXIMUM_CAPACITY)
                        rehash(node);
                    else
                    // 当前新增的该节点作为链表头结点放在哈希表table数组中
                        setEntryAt(tab, index, node);
                    ++modCount;
                    // 递增当前整体的元素个数
                    count = c;
                    oldValue = null;
                    break;
                }
            }
        } finally {
            // 释放lock锁
            unlock();
        }
        // 返回旧值
        return oldValue;
    }
    ```
    scanAndLockForPut的实现：获取该segment锁，从而可以进行put写操作，同时获取当前需要写的值对应的HashEntry链表节点，查找获取之前已经存在的或者创建一个新的。
    
    ```java
    /**
     * Scans for a node containing given key while trying to
     * acquire lock, creating and returning one if not found. Upon
     * return, guarantees that lock is held. UNlike in most
     * methods, calls to method equals are not screened: Since
     * traversal speed doesn't matter, we might as well help warm
     * up the associated code and accesses as well.
     *
     * @return a new node if key not found, else null
     */
    private HashEntry<K,V> scanAndLockForPut(K key, int hash, V value) {
        HashEntry<K,V> first = entryForHash(this, hash);
        HashEntry<K,V> e = first;
        HashEntry<K,V> node = null;
        int retries = -1; // negative while locating node
        
        // 非阻塞自旋获取lock锁
        while (!tryLock()) {
            HashEntry<K,V> f; // to recheck first below
            if (retries < 0) {
                if (e == null) {
                    if (node == null) // speculatively create node
                        node = new HashEntry<K,V>(hash, key, value, null);
                    retries = 0;
                }
                else if (key.equals(e.key))
                    retries = 0;
                else
                    e = e.next;
            }
            
            // MAX_SCAN_RETRIES为2，尝试3次后，则当前线程阻塞等待lock锁
            else if (++retries > MAX_SCAN_RETRIES) {
                lock();
                break;
            }
            else if ((retries & 1) == 0 &&
                     (f = entryForHash(this, hash)) != first) {
                e = first = f; // re-traverse if entry changed
                retries = -1;
            }
        }
        return node;
    }
    ```

#### get读操作
* 获取指定的key对应的value，get读操作是不用获取lock锁的，即不加锁的，通过使用UNSAFE的volatile版本的方法保证线程可见性。实现如下：

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
        Segment<K,V> s; // manually integrate access methods to reduce overhead
        HashEntry<K,V>[] tab;
        int h = hash(key);
        long u = (((h >>> segmentShift) & segmentMask) << SSHIFT) + SBASE;
        
        // 获取segment
        if ((s = (Segment<K,V>)UNSAFE.getObjectVolatile(segments, u)) != null &&
            (tab = s.table) != null) {
            
            // 通过hash值计算哈希表table数组的下标，从而获取对应链表的头结点
            // 从链表头结点遍历链表
            for (HashEntry<K,V> e = (HashEntry<K,V>) UNSAFE.getObjectVolatile
                     (tab, ((long)(((tab.length - 1) & h)) << TSHIFT) + TBASE);
                 e != null; e = e.next) {
                K k;
                if ((k = e.key) == key || (e.hash == h && key.equals(k)))
                    return e.value;
            }
        }
        return null;
    }
    ```
   1. 通过key的hash值，调用UNSAFE.getObjectVolatile方法，由于是volatile版本，可以实现线程之间的可见性（遵循happend-before原则），故可以从最新的segments数组中获取该key所在的segment；
   2. 然后根据key的hash值，获取该segment内部的哈希表table数组的下标，从而获取该key所在的链表的头结点，然后从链表头结点开始遍历该链表，最终如果没有找到，则返回null或者找到对应的链表节点，返回该链表节点的value。


#### size容量计算
* size方法主要是计算当前hashmap中存放的元素的总个数，即累加各个segments的内部的哈希表table数组内的所有链表的所有链表节点的个数。
* 实现逻辑为：整个计算过程刚开始是不对segments加锁的，重复计算两次，如果前后两次hashmap都没有修改过，则直接返回计算结果，如果修改过了，则再加锁计算一次。

    ```java
    /**
     * Returns the number of key-value mappings in this map.  If the
     * map contains more than <tt>Integer.MAX_VALUE</tt> elements, returns
     * <tt>Integer.MAX_VALUE</tt>.
     *
     * @return the number of key-value mappings in this map
     */
    public int size() {
        // Try a few times to get accurate count. On failure due to
        // continuous async changes in table, resort to locking.
        
        // 所有的segments
        final Segment<K,V>[] segments = this.segments;
        int size;
        boolean overflow; // true if size overflows 32 bits
        
        // 累加modCounts，这个在每次计算都是重置为0
        long sum;         // sum of modCounts
        
        // 记录前一次累加的modCounts
        long last = 0L;   // previous sum
        int retries = -1; // first iteration isn't retry
        try {
            for (;;) {
            
                // RETRIES_BEFORE_LOCK值为2
                // retries初始值为-1
                // retries++ == RETRIES_BEFORE_LOCK，表示已经是第三次了，故需要加锁，前两次计算modCounts不一样，即期间有写操作。
                if (retries++ == RETRIES_BEFORE_LOCK) {
                    for (int j = 0; j < segments.length; ++j)
                    // 每个segment都加锁，此时不能执行写操作了
                        ensureSegment(j).lock(); // force creation
                }
                // sum重置为0
                sum = 0L;
                size = 0;
                overflow = false;
                
                // 遍历每个segment
                for (int j = 0; j < segments.length; ++j) {
                    Segment<K,V> seg = segmentAt(segments, j);
                    if (seg != null) {
                    
                        // 累加各个segment的modCount，以便与上一次的modCount进行比较，
                        // 看在这期间是否对segment修改过
                        sum += seg.modCount;
                        
                        // segment使用count记录该segment的内部的所有链表的所有节点的总个数
                        int c = seg.count;
                        // size为记录所有节点的个数，用于作为返回值，使用size += c来累加
                        if (c < 0 || (size += c) < 0)
                            overflow = true;
                    }
                }
                
                // 前后两次都相等，
                // 则说明在这期间没有写的操作，
                // 故可以直接返回了
                if (sum == last)
                    break;
                last = sum;
            }
        } finally {
        
            // retries 大于 RETRIES_BEFORE_LOCK，
            // 说明加锁计算过了，需要释放锁
            if (retries > RETRIES_BEFORE_LOCK) {
                for (int j = 0; j < segments.length; ++j)
                    segmentAt(segments, j).unlock();
            }
        }
        return overflow ? Integer.MAX_VALUE : size;
    }
    ```