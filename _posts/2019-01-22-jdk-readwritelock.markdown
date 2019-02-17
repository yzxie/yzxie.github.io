---
layout:     post
title:      "JDK1.8源码分析：ReentrantReadWriteLock可重入读写锁"
subtitle:   "Java ReentrantReadWriteLock"
date:       2019-01-22 07:12:00
author:     "XYZ"
header-img: "img/article-bg1.jpg"
tags:
    - JDK源码分析
---
### 概述
* ReentrantReadWriteLock包含读写两把锁，如下：

    ```java
    public ReentrantReadWriteLock.WriteLock writeLock() { return writerLock; }
    public ReentrantReadWriteLock.ReadLock  readLock()  { return readerLock; }
    ```
* 对同一线程而言，读，写读（先写后读），写写是共享的，读写（先读后写）是互斥的；对不同线程而言，读是共享的，读写，写读，写写是互斥的。

### 读写锁
* 写锁的请求：
（1）当前没有线程在执行读写，则可以进行写操作；（2）当前存在线程在进行写操作，当前请求写操作的线程就是当前正在进行写操作的线程时，才能继续发起写请求；如果当前是其他线程在写，则当前线程无法请求写；（3）当前不存在写，但是存在线程在读，则无法请求写。

    ```java
    /**
     * Performs tryLock for write, enabling barging in both modes.
     * This is identical in effect to tryAcquire except for lack
     * of calls to writerShouldBlock.
     */
    final boolean tryWriteLock() {
        Thread current = Thread.currentThread();
        // 当前正在访问共享资源的线程数
        int c = getState();
        // 不等于0，表示存在线程对该共享资源的访问
        if (c != 0) {
            // 执行写的数量
            int w = exclusiveCount(c);
            // 没有线程在执行写，即全是读线程，获取写锁失败；
            // 存在线程在写，但不是当前线程，获取写锁失败；
            // 故获取写锁成功的条件是：只有当前线程正在写，则当前线程可以继续执行写，
            // 即当前线程的写可重入
            if (w == 0 || current != getExclusiveOwnerThread())
                return false;
            if (w == MAX_COUNT)
                throw new Error("Maximum lock count exceeded");
        }
        
        // 递增当前线程执行写的次数作为state
        if (!compareAndSetState(c, c + 1))
            return false;
        setExclusiveOwnerThread(current);
        return true;
    }
    ```

* 读锁的请求：当前不存在写的线程，或者当前正在写的线程就是当前请求读的线程，则可以成功请求读；如果当前存在其他线程在写，则无法请求读。

    ```java
    /**
     * Performs tryLock for read, enabling barging in both modes.
     * This is identical in effect to tryAcquireShared except for
     * lack of calls to readerShouldBlock.
     */
    final boolean tryReadLock() {
        Thread current = Thread.currentThread();
        // 自旋
        for (;;) {
            // 获取当前正在进行读写的线程数量
            int c = getState();
            // 存在写线程，而该写线程并不是当前请求读的线程，
            // 即其他线程在写，则当前请求读的线程无法读；
            // 即如果是当前请求读的线程在写，则可以进行请求读，
            // 同一线程，先写后读的情况下，读可重入
            if (exclusiveCount(c) != 0 &&
                getExclusiveOwnerThread() != current)
                return false;
            
            // 如果没有线程在写，或者当前请求读的线程在写
            // ，则可以成功请求读
            int r = sharedCount(c);
            if (r == MAX_COUNT)
                throw new Error("Maximum lock count exceeded");
            
            // 递增读的线程次数
            if (compareAndSetState(c, c + SHARED_UNIT)) {
                if (r == 0) {
                    firstReader = current;
                    firstReaderHoldCount = 1;
                } else if (firstReader == current) {
                    firstReaderHoldCount++;
                } else {
                    HoldCounter rh = cachedHoldCounter;
                    if (rh == null || rh.tid != getThreadId(current))
                        cachedHoldCounter = rh = readHolds.get();
                    else if (rh.count == 0)
                        readHolds.set(rh);
                    rh.count++;
                }
                return true;
            }
        }
    }
    ```
### 总结
* 对于写的请求，要么当前不存在线程在进行读写，要么是该请求写的线程在写，则可以成功获取写锁进行写操作，即写对同一线程是可重入的。如果当前存在其他线程在写，或者存在线程（当前或者其他线程）在读，则无法获取写锁进行写操作。即读写是互斥的，同一线程的写是可以共享的，可重入的。
* 对于读的请求，如果当前不存在线程在写，或者当前在写的线程就是该请求读的线程自身，则可以成功获取读锁进行读操作；如果当前存在其他线程在写，则无法成功执行读请求。
