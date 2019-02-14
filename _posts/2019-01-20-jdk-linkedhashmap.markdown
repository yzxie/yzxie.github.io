---
layout:     post
title:      "JDK1.8源码分析：LinkedHashMap与LRU缓存设计思路"
subtitle:   "JDK LinkedHashMap"
date:       2019-01-20 07:12:00
author:     "XYZ"
header-img: "img/article-bg1.jpg"
tags:
    - JDK源码分析
---
### 概述
* LinkedHashMap继承于HashMap，在HashMap的基础上，新增了两个特性：
 1. 支持以节点的插入顺序来迭代该map内的所有节点；
 2. 支持缓存设计中LRU的特性，即LinkedHashMap支持按访问顺序来排序节点，具体在内部实现为如果开启了这个特性，则每次通过get方法访问了一个节点，则该节点会被移动到内部的双向链表的末尾，故双向链表的头结点是最近最少访问的节点，尾节点为刚刚访问过的节点，中间节点依次类推。
* 以上两个特性是互斥存在的，默认是以节点插入顺序来排序节点，可以通过设置构造函数中的accessOrder为true来开启按节点访问顺序排序。

    ```java
    /**
     * Constructs an empty <tt>LinkedHashMap</tt> instance with the
     * specified initial capacity, load factor and ordering mode.
     *
     * @param  initialCapacity the initial capacity
     * @param  loadFactor      the load factor
     * @param  accessOrder     the ordering mode - <tt>true</tt> for
     *         access-order, <tt>false</tt> for insertion-order
     * @throws IllegalArgumentException if the initial capacity is negative
     *         or the load factor is nonpositive
     */
    public LinkedHashMap(int initialCapacity,
                         float loadFactor,
                         boolean accessOrder) {
        super(initialCapacity, loadFactor);
        this.accessOrder = accessOrder;
    }
    ```
* 以上两个特性都是基于在LinkedHashMap中额外维护了一个双向链表来实现。
* 以上两个特性都是在迭代器中体现，具体为entrySet方法，keySet方法，values方法，在for循环遍历这些方法返回的集合。

### 数据结构与核心字段
* LinkedHashMap继承于HashMap，节点数据也是存储在HashMap的哈希表table数组中。
* 为了支持以上两个特性，在LinkedHashMap内部额外维护了一个双向链表的数据结构：对HashMap的节点Node进行了拓展，定义了双向链表的节点数据结构Entry，增加了before和after两个指针，分别为指向前节点和后节点，从而实现双向链表的特性。
* 如下为在LinkedHashMap内部定义的双向链表的链表节点Entry，双向链表的头结点指针head，双向链表的尾节点指针tail：

    ```java
    // 双向链表数据结构
    /**
     * HashMap.Node subclass for normal LinkedHashMap entries.
     */
    static class Entry<K,V> extends HashMap.Node<K,V> {
        Entry<K,V> before, after;
        Entry(int hash, K key, V value, Node<K,V> next) {
            super(hash, key, value, next);
        }
    }
    
    /**
     * The head (eldest) of the doubly linked list.
     */
    transient LinkedHashMap.Entry<K,V> head;
    
    /**
     * The tail (youngest) of the doubly linked list.
     */
    transient LinkedHashMap.Entry<K,V> tail;
    ```
* 注意LinkedHashMap在HashMap的哈希表table数组内的链表的链表数据存储节点，使用的是这个拓展的Entry类；而对于红黑树节点，则还是使用HashMap中定义的。
* 由于双向链表节点是LinkedHashMap额外的维护的结构，所以在增删改父类HashMap中的哈希表table数组中的数据节点时，需要回调LinkedHashMap中的对该双向链表增删改的方法来保持数据同步。

#### accessOrder：访问顺序排序开关
* 在LinkedHashMap中定义了accessOrder字段来控制是否以访问顺序排序双向链表的节点：默认为false，不使用，使用双向链表节点插入顺序来排序。

    ```java
    /**
     * The iteration ordering method for this linked hash map: <tt>true</tt>
     * for access-order, <tt>false</tt> for insertion-order.
     *
     * @serial
     */
    final boolean accessOrder;
    ```
* accessOrder主要是在LinkedHashMap的get方法中使用，即在访问某个key对应的节点时，判断是否需要将在双向链表中对应的节点移动到双向链表末尾，具体在以下分析。

### 核心方法
由于LinkedHashMap继承于HashMap，在内部也是使用HashMap的哈希表table数组来存储数据，LinkedHashMap主要是覆盖HashMap的相关方法或者实现HashMap的增删改回调方法，来对自身的双向链表进行调整，或者是利用自身维护的双向链表来对HashMap中的相关方法进行重写优化。

#### 覆盖HashMap的方法
* 新增节点：newNode方法，覆盖HashMap的新增节点方法，返回的是LinkedHashMap内部定义的Entry节点，故在HashMap的哈希表table数组内部的链表的链表节点类型为Entry了。同时调用linkNodeLast方法将该节点放到内部的双向链表的末尾。

    ```java
    Node<K,V> newNode(int hash, K key, V value, Node<K,V> e) {
        LinkedHashMap.Entry<K,V> p =
            new LinkedHashMap.Entry<K,V>(hash, key, value, e);
        linkNodeLast(p);
        return p;
    }
    
    // 将该节点放到双向链表的末尾
    // link at the end of list
    private void linkNodeLast(LinkedHashMap.Entry<K,V> p) {
        LinkedHashMap.Entry<K,V> last = tail;
        tail = p;
        if (last == null)
            head = p;
        else {
            p.before = last;
            last.after = p;
        }
    }
    ```

* 访问节点：get方法，在内部调用了HashMap的getNode方法来从HashMap的哈希表table数组查找该指定key对应的节点。额外增加通过accessOrder的判断来决定是否对自身的双向链表节点进行调整。

    ```java
        /**
     * Returns the value to which the specified key is mapped,
     * or {@code null} if this map contains no mapping for the key.
     *
     * <p>More formally, if this map contains a mapping from a key
     * {@code k} to a value {@code v} such that {@code (key==null ? k==null :
     * key.equals(k))}, then this method returns {@code v}; otherwise
     * it returns {@code null}.  (There can be at most one such mapping.)
     *
     * <p>A return value of {@code null} does not <i>necessarily</i>
     * indicate that the map contains no mapping for the key; it's also
     * possible that the map explicitly maps the key to {@code null}.
     * The {@link #containsKey containsKey} operation may be used to
     * distinguish these two cases.
     */
    public V get(Object key) {
        Node<K,V> e;
        
        // getNode为在HashMap中定义的方法
        if ((e = getNode(hash(key), key)) == null)
            return null;
            
        // 判断是否以访问顺序排序双向链表节点
        if (accessOrder)
            afterNodeAccess(e);
        return e.value;
    }
    
    // 将当前访问的节点，调整到双向链表的末尾，实现按访问顺序排序的功能
    void afterNodeAccess(Node<K,V> e) { // move node to last
        LinkedHashMap.Entry<K,V> last;
        if (accessOrder && (last = tail) != e) {
            LinkedHashMap.Entry<K,V> p =
                (LinkedHashMap.Entry<K,V>)e, b = p.before, a = p.after;
            p.after = null;
            if (b == null)
                head = a;
            else
                b.after = a;
            if (a != null)
                a.before = b;
            else
                last = b;
            if (last == null)
                head = p;
            else {
                p.before = last;
                last.after = p;
            }
            tail = p;
            ++modCount;
        }
    }
    ```

* containsValue：判断map中是否存在指定value的节点，重写了hashMap的containsValue方法，利用双向链表来查找。在HashMap中需要遍历哈希表table数组，然后遍历数组中每个元素对应的链表，即从链表头开始一个个比较。
    
    ```java
    /**
     * Returns <tt>true</tt> if this map maps one or more keys to the
     * specified value.
     *
     * @param value value whose presence in this map is to be tested
     * @return <tt>true</tt> if this map maps one or more keys to the
     *         specified value
     */
    public boolean containsValue(Object value) {
        for (LinkedHashMap.Entry<K,V> e = head; e != null; e = e.after) {
            V v = e.value;
            if (v == value || (value != null && value.equals(v)))
                return true;
        }
        return false;
    }
    ```
* 清空数据方法：clear，主要是调用父类HashMap的clear来完成对哈希表table数组内部所有数据的清空，在LinkedHashMap中需要将双向链表的头指针和尾指针均置为null。

    ```java
    /**
     * {@inheritDoc}
     */
    public void clear() {
        super.clear();
        head = tail = null;
    }
    ```

#### HashMap的增删改的回调方法
以上方法由于HashMap没有提供回调方法来进行拓展，故需要在LinkedHashMap中显式重写来加入对双向链表的操作。在HashMap中对于增删改节点对应了回调方法，故可以在LinkedHashMap中实现这些回调方法即可。
* 如下为在HashMap中声明的回调方法：

    ```java
    // Callbacks to allow LinkedHashMap post-actions
    void afterNodeAccess(Node<K,V> p) { }
    void afterNodeInsertion(boolean evict) { }
    void afterNodeRemoval(Node<K,V> p) { }
    ```
* afterNodeAccess：节点访问回调，主要在get方法中调用，可以参见以上get方法的分析。

* afterNodeInsertion：节点插入回调，主要是在HashMap的putVal方法实现中最后调用，即在往HashMap的哈希表table数组插入数据相关查找完成后，最后调用afterNodeInsertion。LinkedHashMap的afterNodeInsertion回调实现如下：

    ```java
    void afterNodeInsertion(boolean evict) { // possibly remove eldest
        LinkedHashMap.Entry<K,V> first;
        
        // 判断是否删除最近最少访问的节点
        if (evict && (first = head) != null && removeEldestEntry(first)) {
            K key = first.key;
            // removeNode内部会调用afterNodeRemoval方法来调整该双向链表
            removeNode(hash(key), key, null, false, true);
        }
    }
    ```
    主要用于在基于LinkedHashMap来实现缓存时，实现缓存的LRU特性使用。
    
* afterNodeRemoval：在HashMap删除某个节点时，回调afterNodeRemoval方法。LinkedHashMap的实现为在自身维护的双向链表中删除对应的链表节点：

    ```java
    // 在HashMap中的链表节点e删除后，同步调整该双向链表，删除该节点
    void afterNodeRemoval(Node<K,V> e) { // unlink
        LinkedHashMap.Entry<K,V> p =
            (LinkedHashMap.Entry<K,V>)e, b = p.before, a = p.after;
        p.before = p.after = null;
        if (b == null)
            head = a;
        else
            b.after = a;
        if (a == null)
            tail = b;
        else
            a.before = b;
    }
    ```
### 迭代器
* 在LinkedHashMap中，迭代器相关的操作是基于自身的双向链表，而不是父类HashMap的哈希表table数组来实现的，故迭代顺序是基于双向链表的顺序实现的，即以插入顺序（从前到后：最先插入->最后插入）排序或者访问顺序排序（从前到后：最近最少访问 -> 刚刚访问）。
* LinkedHashMap的方法包括：entrySet方法，keySet方法，values方法
* LinkedHashMap的迭代器定义：主要在构造函数中将next初始化为双向链表的头结点head。

    ```java
    // Iterators
    
    abstract class LinkedHashIterator {
        LinkedHashMap.Entry<K,V> next;
        LinkedHashMap.Entry<K,V> current;
        int expectedModCount;
    
        LinkedHashIterator() {
            // 初始化为双向链表头结点head
            next = head;
            expectedModCount = modCount;
            current = null;
        }
    
        public final boolean hasNext() {
            return next != null;
        }
    
        final LinkedHashMap.Entry<K,V> nextNode() {
            LinkedHashMap.Entry<K,V> e = next;
            // 并发修改异常
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
            if (e == null)
                throw new NoSuchElementException();
            current = e;
            next = e.after;
            return e;
        }
    
        public final void remove() {
            Node<K,V> p = current;
            if (p == null)
                throw new IllegalStateException();
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
            current = null;
            K key = p.key;
            removeNode(hash(key), key, null, false, false);
            expectedModCount = modCount;
        }
    }
    ```

### LRU缓存
* 由于LinkedHashMap支持按访问顺序排序双向链表的特性，故可以基于LinkedHashMap来实现一个LRU缓存，具体为拓展LinkedHashMap，在缓存类中，重写removeEldestEntry方法来定义删除最近最少访问的节点的条件。

    ```java
    /**
     * Returns <tt>true</tt> if this map should remove its eldest entry.
     * This method is invoked by <tt>put</tt> and <tt>putAll</tt> after
     * inserting a new entry into the map.  It provides the implementor
     * with the opportunity to remove the eldest entry each time a new one
     * is added.  This is useful if the map represents a cache: it allows
     * the map to reduce memory consumption by deleting stale entries.
     *
     * <p>Sample use: this override will allow the map to grow up to 100
     * entries and then delete the eldest entry each time a new entry is
     * added, maintaining a steady state of 100 entries.
     * <pre>
     *     private static final int MAX_ENTRIES = 100;
     *
     *     protected boolean removeEldestEntry(Map.Entry eldest) {
     *        return size() &gt; MAX_ENTRIES;
     *     }
     * </pre>
     *
     * <p>This method typically does not modify the map in any way,
     * instead allowing the map to modify itself as directed by its
     * return value.  It <i>is</i> permitted for this method to modify
     * the map directly, but if it does so, it <i>must</i> return
     * <tt>false</tt> (indicating that the map should not attempt any
     * further modification).  The effects of returning <tt>true</tt>
     * after modifying the map from within this method are unspecified.
     *
     * <p>This implementation merely returns <tt>false</tt> (so that this
     * map acts like a normal map - the eldest element is never removed).
     *
     * @param    eldest The least recently inserted entry in the map, or if
     *           this is an access-ordered map, the least recently accessed
     *           entry.  This is the entry that will be removed it this
     *           method returns <tt>true</tt>.  If the map was empty prior
     *           to the <tt>put</tt> or <tt>putAll</tt> invocation resulting
     *           in this invocation, this will be the entry that was just
     *           inserted; in other words, if the map contains a single
     *           entry, the eldest entry is also the newest.
     * @return   <tt>true</tt> if the eldest entry should be removed
     *           from the map; <tt>false</tt> if it should be retained.
     */
    protected boolean removeEldestEntry(Map.Entry<K,V> eldest) {
        return false;
    }
    ```
* 由以上分析可知，removeEldestEntry主要是在HashMap的新增节点的回调afterNodeInsertion中调用。在LinkedHashMap的afterNodeInsertion方法实现如下：

    ```java
    void afterNodeInsertion(boolean evict) { // possibly remove eldest
        LinkedHashMap.Entry<K,V> first;
        
        // 判断是否删除最近最少访问的节点
        if (evict && (first = head) != null && removeEldestEntry(first)) {
            K key = first.key;
            // removeNode内部会调用afterNodeRemoval方法来调整该双向链表
            removeNode(hash(key), key, null, false, true);
        }
    }
    ```
   1. 在afterNodeInsertion中，head头结点就是最近最少访问的节点，故在该缓存类中，需要设置accessOrder为true来开启按访问顺序排序；
   2. 在afterNodeInsertion中会调用HashMap的removeNode方法来删除双向链表头结点head对应的哈希表table的链表的链表节点，在HashMap的removeNode会回调LinkedHashMap的afterNodeRemoval来删除LinkedHashMap内部的双向链表的链表节点；
   3. 故在继承了LinkedHashMap的缓存类只需实现removeEldestEntry方法即可。
* removeEldestEntry的方法实现例子：

    ```java
    public class LRUCache extends LinkedHashMap {
        
        private static final int MAX_ENTRIES = 100;
        
        public LRUCache(int initialCapacity, float loadFactor) {
        	// 第三个参数为accessOrder
            super(initialCapacity, loadFactor, true);
        }
        
        protected boolean removeEldestEntry(Map.Entry eldest) {
            return size() > MAX_ENTRIES;
        }
    }
    ```