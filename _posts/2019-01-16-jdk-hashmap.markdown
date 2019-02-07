---
layout:     post
title:      "JDK1.8源码分析：HashMap"
subtitle:   "JDK HashMap"
date:       2019-01-16 07:12:00
author:     "XYZ"
header-img: "img/article-bg1.jpg"
tags:
    - JDK源码分析
---
### 数据结构
在JDK1.8之前，HashMap是基于链式哈希实现的，而在JDK1.8之后，为了提高冲突节点的访问性能，在链式哈希实现的基础上，在哈希表大小超过64时，针对冲突节点链条，如果节点数量超过8个，则升级为红黑树，小于等于6个时，则降级为链表结构。

#### 链式哈希
* 链式哈希是一个数组结构，数组元素为链表或者红黑树。如下为HashMap的内部数据存储结构，也是链式哈希的实现。其中Node为一个key的hash值相同的数据的链表（或红黑树）的头节点，即为冲突节点的链表。

    ```java
    /**
     * The table, initialized on first use, and resized as
     * necessary. When allocated, length is always a power of two.
     * (We also tolerate length zero in some operations to allow
     * bootstrapping mechanics that are currently not needed.)
     */
    transient Node<K,V>[] table;
    ```
#### 链表节点
* 如下为链表节点的数据结构设计：包括key，value，key的hash值，当前节点在链表中的下一个节点next。
* 该链表内的所有节点的key的hash值是相同的，链表头结点存放在哈希表table中，基于hash值与table大小获取该链表头结点在table中的位置下标。

    ```java
    /**
     * Basic hash bin node, used for most entries.  (See below for
     * TreeNode subclass, and in LinkedHashMap for its Entry subclass.)
     */
    static class Node<K,V> implements Map.Entry<K,V> {
        final int hash;
        final K key;
        V value;
        Node<K,V> next;
    
        Node(int hash, K key, V value, Node<K,V> next) {
            this.hash = hash;
            this.key = key;
            this.value = value;
            this.next = next;
        }
    
        public final K getKey()        { return key; }
        public final V getValue()      { return value; }
        public final String toString() { return key + "=" + value; }
    
        public final int hashCode() {
            return Objects.hashCode(key) ^ Objects.hashCode(value);
        }
    
        public final V setValue(V newValue) {
            V oldValue = value;
            value = newValue;
            return oldValue;
        }
    
        public final boolean equals(Object o) {
            if (o == this)
                return true;
            if (o instanceof Map.Entry) {
                Map.Entry<?,?> e = (Map.Entry<?,?>)o;
                if (Objects.equals(key, e.getKey()) &&
                    Objects.equals(value, e.getValue()))
                    return true;
            }
            return false;
        }
    }
    ```
#### 红黑树节点
* 与链表节点一样，红黑树中存放key的hash值相同的节点集合，其中红黑树根节点root存放在哈希表table中，对应的table数组下标也是基于key的hash值和数组table大小获取。

    ```java
    /**
     * Entry for Tree bins. Extends LinkedHashMap.Entry (which in turn
     * extends Node) so can be used as extension of either regular or
     * linked node.
     */
    static final class TreeNode<K,V> extends LinkedHashMap.Entry<K,V> {
        TreeNode<K,V> parent;  // red-black tree links
        TreeNode<K,V> left;
        TreeNode<K,V> right;
        TreeNode<K,V> prev;    // needed to unlink next upon deletion
        boolean red;
        TreeNode(int hash, K key, V val, Node<K,V> next) {
            super(hash, key, val, next);
        }
    
        /**
         * Returns root of tree containing this node.
         */
        final TreeNode<K,V> root() {
            for (TreeNode<K,V> r = this, p;;) {
                if ((p = r.parent) == null)
                    return r;
                r = p;
            }
        }
        
        ...
        
    }
    ```

### 核心设计
#### table数组容量与阈值

```java
/**
 * The default initial capacity - MUST be a power of two.
 */
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16

/**
 * The maximum capacity, used if a higher value is implicitly specified
 * by either of the constructors with arguments.
 * MUST be a power of two <= 1<<30.
 */
static final int MAXIMUM_CAPACITY = 1 << 30;

/**
 * The load factor used when none specified in constructor.
 */
static final float DEFAULT_LOAD_FACTOR = 0.75f;
```
* DEFAULT_INITIAL_CAPACITY： table数组默认大小为16，即在应用代码中，创建一个HashMap时，不指定容量，则之后在第一次添加数据到该HashMap，首先分配一个大小为16的数组table。

* MAXIMUM_CAPACITY：数组最大大小，超过则不再进行拓容，大小为2的30次方。

* DEFAULT_LOAD_FACTOR：数组拓容阈值，即在当前数组存放的数据节点达到容量的0.75时，则进行数组拓容，具体拓容逻辑在resize方法定义。

#### 构建红黑树阈值
* 默认哈希表对应的数组的每个元素是一个链表，具体为存放链表头节点，而在冲突节点较多时，即存在较多key的hash值相同的元素，则会导致链表过长，在应用代码中获取某个key的值value，时间复杂度就会增加，即默认的O(1)变为了O(N)，其中N为该链表长度。
* 所以为了解决这种情况下的性能问题，JDK1.8提供了从链表转为红黑树，或者从红黑树还原为链表的设计，在红黑树中获取某个节点的时间复杂度为O(logN)，具体阈值如下：
    ```java
    /**
     * The bin count threshold for using a tree rather than list for a
     * bin.  Bins are converted to trees when adding an element to a
     * bin with at least this many nodes. The value must be greater
     * than 2 and should be at least 8 to mesh with assumptions in
     * tree removal about conversion back to plain bins upon
     * shrinkage.
     */
    static final int TREEIFY_THRESHOLD = 8;
    
    /**
     * The bin count threshold for untreeifying a (split) bin during a
     * resize operation. Should be less than TREEIFY_THRESHOLD, and at
     * most 6 to mesh with shrinkage detection under removal.
     */
    static final int UNTREEIFY_THRESHOLD = 6;
    
    /**
     * The smallest table capacity for which bins may be treeified.
     * (Otherwise the table is resized if too many nodes in a bin.)
     * Should be at least 4 * TREEIFY_THRESHOLD to avoid conflicts
     * between resizing and treeification thresholds.
     */
    static final int MIN_TREEIFY_CAPACITY = 64;
    ```
* TREEIFY_THRESHOLD：构造红黑树的阈值，默认为8，即链表的元素个数超过8个的时候，则将链表转为红黑树，不过前提是数组大小大于MIN_TREEIFY_CAPACITY。
* MIN_TREEIFY_CAPACITY：触发红黑树化的前提条件，默认值为64，即哈希表数组table的大小超过64的时候，如果某个数组元素对应的链表的长度超过TREEIFY_THRESHOLD，则将该数组元素（即链表头结点）对应的链表转为红黑树结构。
* UNTREEIFY_THRESHOLD：从红黑树退化为链表的条件，默认为6，即当红黑树中节点个数不超过6时，则退化为链表。

### 核心操作设计
#### 数组大小capacity调整
* HashMap内部存储数据所用的链式哈希表是基于数组实现的，即数组 + 链表或者数组 + 红黑树，所以在往HashMap存放数据之前，需要指定数组大小来创建这个数组。
* 在存放某个数据时，需要根据该数据的key的hash值与数组大小取模，从而确定该数据所在的链表（具体为链表头结点）在数组中的位置下标。在HashMap的设计中，为了提高性能，采用的是位运算的方式，而不是使用%的方式取模，具体方式如下：

    ```java
    (capacity - 1) & hash
    ```
* 这种方式的实现基础为capacity的值为2的N次方，则capacity - 1的值对应二进制就全是1了，如16的二进制为：10000，15的二进制为：01111，则01111与任何数字的与运算的结果为0到15，如：

    ```
    01111 & 00010 = 00010，即15与2取模等于2
    01111 & 100001 = 00001，即15与33取模等于1
    ```
* 所以在指定HashMap的容量capacity时，则内部会将capacity调整为2的N次方，即调整后的capacity大于或等于应用代码指定的capacity。调整实现为：在tableSizeFor方法定义，如下：

    ```java
    public HashMap(int initialCapacity, float loadFactor) {
        if (initialCapacity < 0)
            throw new IllegalArgumentException("Illegal initial capacity: " +
                                               initialCapacity);
        if (initialCapacity > MAXIMUM_CAPACITY)
            initialCapacity = MAXIMUM_CAPACITY;
        if (loadFactor <= 0 || Float.isNaN(loadFactor))
            throw new IllegalArgumentException("Illegal load factor: " +
                                               loadFactor);
        this.loadFactor = loadFactor;
        this.threshold = tableSizeFor(initialCapacity);
    }
    
    /**
     * Returns a power of two size for the given target capacity.
     */
    static final int tableSizeFor(int cap) {
        int n = cap - 1;
        n |= n >>> 1;
        n |= n >>> 2;
        n |= n >>> 4;
        n |= n >>> 8;
        n |= n >>> 16;
        return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
    }
    ```

#### put设值
* put操作往HashMap存放数据，在内部主要是在putVal方法定义实现逻辑。具体过程请看代码注释：

    ```java
    /**
     * Implements Map.put and related methods
     *
     * @param hash hash for key
     * @param key the key
     * @param value the value to put
     * @param onlyIfAbsent if true, don't change existing value
     * @param evict if false, the table is in creation mode.
     * @return previous value, or null if none
     */
    final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
                   
        // tab：指向哈希表table
        // p：遍历哈希表table时，存放遍历到的数组元素
        Node<K,V>[] tab; Node<K,V> p; int n, i;
        
        // table为null，还没初始化或者大小为0，
        // 则调用resize先创建数组或者对数组进行拓容
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;
        
        // 如果需要存放的元素，在数组中还不存在对应的链表，则创建一个链表头结点存放该元素，
        // 并存放到数据中，存放操作就此完成。
        if ((p = tab[i = (n - 1) & hash]) == null)
            tab[i] = newNode(hash, key, value, null);
            
        // 数组中已经存放该链表，则：
        else {
            Node<K,V> e; K k;
            
            // 1.如果数组元素（链表头结点）就是该节点，
            // 即此次put操作是更新，则更新值value即可；
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;
            
            // 2. 或者之前已经将链表升级为了红黑树，
            // 则往该红黑树插入该节点；
            else if (p instanceof TreeNode)
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            
             // 3. 或者链表头结点不是该元素，
             // 则遍历链表在链表尾部插入该节点，
             // 在链表尾部插入之后，
             // 需要检查一下是否需要将该链表升级为红黑树
            else {
                for (int binCount = 0; ; ++binCount) {
                    if ((e = p.next) == null) {
                        p.next = newNode(hash, key, value, null);
                        
                        // 检查链表长度是否超过了8
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        // 在内部检查哈希表大小，然后决定是否转为红黑树
                            treeifyBin(tab, hash);
                        break;
                    }
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    p = e;
                }
            }
            
            // 更新节点值value
            if (e != null) { // existing mapping for key
                V oldValue = e.value;
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                afterNodeAccess(e);
                
                // 直接返回更新之前的值
                // 更新操作不是结构性修改，故不需要往下继续执行。
                return oldValue;
            }
        }
        
        // 记录结构性修改（增加或删除节点）次数
        // 用于实现迭代器iterator的fail-fast快速失败
        ++modCount;
        
        // 如果插入后，哈希表大小超过了阈值，
        // 则resize拓容
        if (++size > threshold)
            resize();
        afterNodeInsertion(evict);
        return null;
    }
    ```
#### resize扩容
* resize操作主要是对哈希表数组table进行拓容或初始化，拓容为每次增大为原来的2倍，从而保证容量capacity为2的N次方的设计。
* 拓容阈值threshold：threshold = capacity * load factor，即默认为容量capacity的0.75，当存放的元素达到容量的3/4时，进行拓容resize操作。

    ```java
    /**
     * Initializes or doubles table size.  If null, allocates in
     * accord with initial capacity target held in field threshold.
     * Otherwise, because we are using power-of-two expansion, the
     * elements from each bin must either stay at same index, or move
     * with a power of two offset in the new table.
     *
     * @return the table
     */
    final Node<K,V>[] resize() {
        Node<K,V>[] oldTab = table;
        int oldCap = (oldTab == null) ? 0 : oldTab.length;
        int oldThr = threshold;
        int newCap, newThr = 0;
        // 原来的容量大于0，即哈希表数组table已经存在了的
        if (oldCap > 0) {
        
            // 当前容量已经是最大容量，则直接返回，不再进行拓容。
            if (oldCap >= MAXIMUM_CAPACITY) {
                threshold = Integer.MAX_VALUE;
                return oldTab;
            }
            
            // 将容量和拓容阈值都拓展为原来的2倍
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                     oldCap >= DEFAULT_INITIAL_CAPACITY)
                newThr = oldThr << 1; // double threshold
        }
        else if (oldThr > 0) // initial capacity was placed in threshold
        // 旧容量小于0，则容量设值为拓展阈值
            newCap = oldThr;
        
        // 初始化哈希数组，使用默认值
        else {               // zero initial threshold signifies using defaults
            
            // 容量等于初始大小16
            newCap = DEFAULT_INITIAL_CAPACITY;
            
            // 拓容阈值等于16*0.75=12
            newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
        }
        if (newThr == 0) {
            float ft = (float)newCap * loadFactor;
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                      (int)ft : Integer.MAX_VALUE);
        }
        threshold = newThr;
        @SuppressWarnings({"rawtypes","unchecked"})
        
            // 创建一个新的哈希数组，容量为原来的2倍
            Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
        table = newTab;
        if (oldTab != null) {
        
            // 遍历旧的哈希数组，获取元素放到拓容后，新的哈希数组中
            for (int j = 0; j < oldCap; ++j) {
                Node<K,V> e;
                if ((e = oldTab[j]) != null) {
                    // e已经存放了原来的数组元素，
                    // 则将旧数组的元素置为null，便于GC回收
                    
                    oldTab[j] = null;
                    
                    // 旧数组最后一个元素
                    if (e.next == null)
                        newTab[e.hash & (newCap - 1)] = e;
                    
                    // 红黑树节点拆分
                    else if (e instanceof TreeNode)
                        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                    
                    // 将链表（具体为链表头结点）
                    // 复制到新数组中
                    // 由于数组拓容后，大小capacity变大了，
                    // 故单个链表中的元素的hash与capacity取模会发生变化，
                    // 故旧数组的某个数组元素，
                    // 对应的链表也要拆分成两条链表，放在新数组的两个位置中。
                    
                    // 同时需要保持链表中元素的位置，
                    // 即链表前半部分对应的新的子链表对应的新数组中的位置为
                    // 该链表在旧数组的位置下标
                    // 链表后半部分对应的新的子链表放在
                    // 新数组后面的一个位置下标
                    else { // preserve order
                        
                        // lo为链表的前半部分
                        // hi为链表的后半部分
                        Node<K,V> loHead = null, loTail = null;
                        Node<K,V> hiHead = null, hiTail = null;
                        Node<K,V> next;
                        do {
                            next = e.next;
                            
                            // e.hash与oldCap，而不是oldCap-1，而oldCap为2的N次方，二进制为，
                            // 如16的二进制为10000
                            // 则 e.hash & oldCap == 0，表示e所在链表，
                            // 在拓展后在新数组中的位置，
                            // 等于原来在旧数组中的，
                            // 所以放到lo部分。
                            if ((e.hash & oldCap) == 0) {
                                if (loTail == null)
                                    loHead = e;
                                else
                                    loTail.next = e;
                                loTail = e;
                            }
                            
                            // 拓容后，hash值与新的capacity取模不等于与旧的capacity的，
                            // 故放到hi部分。
                            else {
                                if (hiTail == null)
                                    hiHead = e;
                                else
                                    hiTail.next = e;
                                hiTail = e;
                            }
                        } while ((e = next) != null);
                        
                        if (loTail != null) {
                            loTail.next = null;
                            // lo链表放到新数组下标j中
                            newTab[j] = loHead;
                        }
                        if (hiTail != null) {
                        
                            // hi链表放到新数组下标j+oldCap中
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

#### treeifyBin构造红黑树
* 主要用来将某个数组元素对应的链表，转为红黑树结构，从而解决链表长度过长时的访问性能问题。

    ```java
    /**
     * Replaces all linked nodes in bin at index for given hash unless
     * table is too small, in which case resizes instead.
     */
    final void treeifyBin(Node<K,V>[] tab, int hash) {
        int n, index; Node<K,V> e;
        
        // 当哈希表为null即还没初始化，
        // 或者哈希表对应的数组大小小于64时，
        // 则resize，先不将该链表转为红黑树，
        // 即使该链表长度超过了8，
        // 主要是因为当前的哈希表还是比较小的。
        if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY)
            resize();
        
        // 从哈希表中取出该元素，将该元素（链表头结点）
        // 对应的链表调整为红黑树
        else if ((e = tab[index = (n - 1) & hash]) != null) {
            TreeNode<K,V> hd = null, tl = null;
            do {
                TreeNode<K,V> p = replacementTreeNode(e, null);
                if (tl == null)
                    hd = p;
                else {
                    p.prev = tl;
                    tl.next = p;
                }
                tl = p;
            } while ((e = e.next) != null);
            if ((tab[index] = hd) != null)
                hd.treeify(tab);
        }
    }
    ```
#### clear情况
* 清空HashMap，具体为将哈希表数组table的每个元素置为null，则该元素对应的链表或红黑树则没有被引用了，可以被GC回收。

    ```java
    /**
     * Removes all of the mappings from this map.
     * The map will be empty after this call returns.
     */
    public void clear() {
        Node<K,V>[] tab;
        modCount++;
        if ((tab = table) != null && size > 0) {
            size = 0;
            for (int i = 0; i < tab.length; ++i)
                tab[i] = null;
        }
    }
    ```

### 哈希表节点Node集合
#### EntrySet：哈希表所有节点集合
* 从数组的每个元素对应的链表中取出链表节点放到该集合
#### keySet：哈希表key集合
* 哈希表所有节点的key集合
#### Values：哈希表value集合
* 哈希表所有节点的value集合

#### 链表迭代器基类：HashIterator
* 主要被以上集合用来遍历哈希表数组，然后遍历每个数组对应的链表，从而获取所有的数据，即Node节点，然后放到集合中。
* 派生类包括：KeyIterator，ValueIterator，EntryIterator，其中下一个节点都依赖nextNode方法，从链表头结点开始，获取该链表的下一个节点。
* 迭代器是快速失败fail-fast的，即在使用过程中，如果往HashMap中添加或删除了数据，则抛异常退出。

    ```java
    abstract class HashIterator {
        Node<K,V> next;        // next entry to return
        Node<K,V> current;     // current entry
        int expectedModCount;  // for fast-fail
        int index;             // current slot
    
        HashIterator() {
            expectedModCount = modCount;
            Node<K,V>[] t = table;
            
            // 初始化为null
            current = next = null;
            
            // 从数组table第一个数组元素开始
            index = 0;
            if (t != null && size > 0) { // advance to first entry
            
            // 遍历哈希表数组table，
            // 获取第一个数组节点（即某个链表头节点）
            // 不为null的节点，
            // 即整个哈希表从数组和链表两个角度，
            // 第一个节点Node作为迭代的开始节点。
                do {} while (index < t.length && (next = t[index++]) == null);
            }
        }
    
        public final boolean hasNext() {
            return next != null;
        }
    
        final Node<K,V> nextNode() {
            Node<K,V>[] t;
            Node<K,V> e = next;
            
             // 并发进行了结构性修改
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
            if (e == null)
                throw new NoSuchElementException();
            
            // 先遍历数组的元素对应的链表
            // 遍历完该链表后，
            // 遍历数组下一个元素对应的链表
            if ((next = (current = e).next) == null && (t = table) != null) {
                do {} while (index < t.length && (next = t[index++]) == null);
            }
            return e;
        }
    
        public final void remove() {
            Node<K,V> p = current;
            if (p == null)
                throw new IllegalStateException();
                
            // 并发进行了结构性修改
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
            current = null;
            K key = p.key;
            removeNode(hash(key), key, null, false, false);
            expectedModCount = modCount;
        }
    }
    ```
### 线程安全
* HashMap不是线程安全的，故如果需要保证多线程访问的并发安全性，需要使用ConcurrentHashMap，或者使用Collections.synchronizedMap方法来对HashMap进行包装成类型为SynchronizedMap的，线程安全的map。
* ConcurrentHashMap主要是基于CAS来实现乐观锁来实现线程安全；
* SynchronizedMap的内部实现，主要是通过synchronized关键字结合一个对象锁mutex，来对会产生并发问题的操作，如get，put，remove，clear，contains操作进行同步操作。