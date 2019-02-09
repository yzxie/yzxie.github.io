---
layout:     post
title:      "JDK源码分析：ConcurrentHashMap（JDK1.7和JDK1.8），HashTable与Collections.synchronizedMap"
subtitle:   "JDK HashMap"
date:       2019-01-17 07:12:00
author:     "XYZ"
header-img: "img/article-bg1.jpg"
tags:
    - JDK源码分析
---
### 概述
* ConcurrentHashMap是线程安全的HashMap，提供与HashTable一样的线程安全特性。
* 与HashMap不同的是，除了线程安全之外，HashMap的key和value都支持null，而HashTable和ConcurrentHashMap的key和value都不允许是null。
* 与HashTable不同的是，ConcurrentHashMap不需要对所有需要线程安全保证的方法使用同步锁synchronized来进行加锁，synchronized加锁会导致任何时候只能存在一个线程进行读写操作。
* ConcurrentHash使用ReentrantLock（JDK1.7），或者CAS结合synchronized（JDK1.8）来实现线程安全，可以支持多个线程同时进行读写操作，提高并发读写的性能。
* HashTable与HashMap的数据结构基本一样，都是基于链式哈希实现的，具体为采用数组 + 链表（HashMap在JDK1.8也使用数组 + 红黑树）的实现方式。
* ConcurrentHashMap的实现方式和所使用的数据结构在JDK1.8中相对JDK1.7改动较大。

### JDK1.7-ConcurrentHashMap
* [JDK源码分析：ConcurrentHashMap（JDK1.7版本）
](https://blog.csdn.net/u010013573/article/details/86776146)
### JDK1.8-ConcurrentHashMap
* [JDK源码分析：ConcurrentHashMap（JDK1.8版本）
](https://blog.csdn.net/u010013573/article/details/86777752)
### HashTable
* HashTable是同步版本的HashMap，内部数据结构与HashMap一样也都是使用链式哈希来实现的，即数组 + 链表（JDK1.8版本的HashMap新增了数组 + 红黑树的优化）。
* HashTable通过在方法中使用synchronized关键字修饰来进行线程同步，实现线程安全，但是由于是方法级别使用synchronized，即是使用HashTable对象自身作为monitor锁，故多个线程的读读，读写，写写都是互斥，即任何使用只能存在一个线程对HashTable对象进行访问，所以HashTable的并发性能是较低的。
* 链表节点定义：与HashMap一样都是继承于Map.Entry：value和next都是不需要使用volatile修饰的，因为synchronized同时具有线程同步和线程可见性的作用。

	```java
	/**
	 * Hashtable bucket collision list entry
	 */
	private static class Entry<K,V> implements Map.Entry<K,V> {
	    final int hash;
	    final K key;
	    V value;
	    Entry<K,V> next;
	
	    protected Entry(int hash, K key, V value, Entry<K,V> next) {
	        this.hash = hash;
	        this.key =  key;
	        this.value = value;
	        this.next = next;
	    }
	
		...
		
	}
	```
* 方法级别的synchronized同步：get，put等方法均是使用synchronized修饰

	```java
	// get读操作
	public synchronized V get(Object key) {
	    Entry<?,?> tab[] = table;
	    int hash = key.hashCode();
	    
		// %取模，而HashMap为通过位运算取模，性能较高
	    int index = (hash & 0x7FFFFFFF) % tab.length;
	    for (Entry<?,?> e = tab[index] ; e != null ; e = e.next) {
	        if ((e.hash == hash) && e.key.equals(key)) {
	            return (V)e.value;
	        }
	    }
	    return null;
	}
	
	// put写操作
	public synchronized V put(K key, V value) {
	    // Make sure the value is not null
	    if (value == null) {
	        throw new NullPointerException();
	    }
	
	    // Makes sure the key is not already in the hashtable.
	    Entry<?,?> tab[] = table;
	    int hash = key.hashCode();
	    int index = (hash & 0x7FFFFFFF) % tab.length;
	    @SuppressWarnings("unchecked")
	
		// 更新
	    Entry<K,V> entry = (Entry<K,V>)tab[index];
	    for(; entry != null ; entry = entry.next) {
	        if ((entry.hash == hash) && entry.key.equals(key)) {
	            V old = entry.value;
	            entry.value = value;
	            return old;
	        }
	    }
	
		// 新增节点
	    addEntry(hash, key, value, index);
	    return null;
	}
	
	private void addEntry(int hash, K key, V value, int index) {
	    modCount++;
	
	    Entry<?,?> tab[] = table;
	    if (count >= threshold) {
	        // Rehash the table if the threshold is exceeded
	        rehash();
	
	        tab = table;
	        hash = key.hashCode();
	        index = (hash & 0x7FFFFFFF) % tab.length;
	    }
	
	    // Creates the new entry.
	    @SuppressWarnings("unchecked")
	    Entry<K,V> e = (Entry<K,V>) tab[index];
	    tab[index] = new Entry<>(hash, key, value, e);
	    count++;
	}
	```
* 如果需要使用线程安全的HashMap，则优先考虑使用ConcurrentHashMap，或者使用Collections.synchronizedMap来将HashMap包装成线程安全的SynchronizedMap。因为ConcurrentHashMap和HashMap的实现更加高效，如根据key的hash计算哈希表table数组的下标的实现，HashTable是使用%取余，而ConcurrentHashMap和HashMap都是使用位运算，效率更高，所以整体性能高于HashTable。
### Collections.synchronizedMap
* Collections.synchronizedMap的方法定义如下：将传入的Map使用SynchronizedMap包装后返回一个SynchronizedMap实例，SynchronizedMap是线程安全的，这是一种包装器设计模式的使用。

	```java
	/**
	 * Returns a synchronized (thread-safe) map backed by the specified
	 * map.  In order to guarantee serial access, it is critical that
	 * <strong>all</strong> access to the backing map is accomplished
	 * through the returned map.<p>
	 *
	 * It is imperative that the user manually synchronize on the returned
	 * map when iterating over any of its collection views:
	 * <pre>
	 *  Map m = Collections.synchronizedMap(new HashMap());
	 *      ...
	 *  Set s = m.keySet();  // Needn't be in synchronized block
	 *      ...
	 *  synchronized (m) {  // Synchronizing on m, not s!
	 *      Iterator i = s.iterator(); // Must be in synchronized block
	 *      while (i.hasNext())
	 *          foo(i.next());
	 *  }
	 * </pre>
	 * Failure to follow this advice may result in non-deterministic behavior.
	 *
	 * <p>The returned map will be serializable if the specified map is
	 * serializable.
	 *
	 * @param <K> the class of the map keys
	 * @param <V> the class of the map values
	 * @param  m the map to be "wrapped" in a synchronized map.
	 * @return a synchronized view of the specified map.
	 */
	public static <K,V> Map<K,V> synchronizedMap(Map<K,V> m) {
	    return new SynchronizedMap<>(m);
	}
	```
* SynchronizedMap的定义：SynchronizedMap的实现主要是在内部定义了一个被包装的map的引用，一个对象锁mutex；
* 在方法内使用synchronized(mutex)，即使用同步代码块的方式来替代HashTable的使用同步方法的方式，缩小同步范围，提高性能，在代码块中对被包装的map进行操作，从而实现线程安全。

	```java
    private static class SynchronizedMap<K,V>
        implements Map<K,V>, Serializable {
        private static final long serialVersionUID = 1978198479659022715L;

        private final Map<K,V> m;     // Backing Map
        final Object      mutex;        // Object on which to synchronize

        SynchronizedMap(Map<K,V> m) {
            this.m = Objects.requireNonNull(m);
            mutex = this;
        }

        SynchronizedMap(Map<K,V> m, Object mutex) {
            this.m = m;
            this.mutex = mutex;
        }

        public int size() {
            synchronized (mutex) {return m.size();}
        }
        public boolean isEmpty() {
            synchronized (mutex) {return m.isEmpty();}
        }
        public boolean containsKey(Object key) {
            synchronized (mutex) {return m.containsKey(key);}
        }
        public boolean containsValue(Object value) {
            synchronized (mutex) {return m.containsValue(value);}
        }
        public V get(Object key) {
            synchronized (mutex) {return m.get(key);}
        }

        public V put(K key, V value) {
            synchronized (mutex) {return m.put(key, value);}
        }
        public V remove(Object key) {
            synchronized (mutex) {return m.remove(key);}
        }
        public void putAll(Map<? extends K, ? extends V> map) {
            synchronized (mutex) {m.putAll(map);}
        }
        public void clear() {
            synchronized (mutex) {m.clear();}
        }
		
		...
	}
	```
