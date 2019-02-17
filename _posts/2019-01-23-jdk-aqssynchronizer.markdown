---
layout:     post
title:      "JUC并发包基于AQS实现的线程同步器的案例分析"
subtitle:   "Java AQS Synchronizers"
date:       2019-01-23 07:12:00
author:     "XYZ"
header-img: "img/article-bg1.jpg"
tags:
    - JDK源码分析
---
以下是JUC并发包提供的基于AQS实现的线程同步器。
### ReentrantLock：可重入锁
* 通常用于多线程操作进行同步，实现线程安全。存在公平和非公平两种实现，默认为非公平。
* 如在LinkedBlockingQueue中使用putLock来同步多个生产者的写操作，使用takeLock来同步多个消费者的读操作。

    ```java
    public boolean offer(E e, long timeout, TimeUnit unit)
        throws InterruptedException {
    
        if (e == null) throw new NullPointerException();
        long nanos = unit.toNanos(timeout);
        int c = -1;
        final ReentrantLock putLock = this.putLock;
        final AtomicInteger count = this.count;
        // 获取写锁，成功则其他线程在这个地方等待
        putLock.lockInterruptibly();
        try {
            while (count.get() == capacity) {
                if (nanos <= 0)
                    return false;
                nanos = notFull.awaitNanos(nanos);
            }
            enqueue(new Node<E>(e));
            c = count.getAndIncrement();
            if (c + 1 < capacity)
                notFull.signal();
        } finally {
            // 释放写锁，成功则其他线程（其中一个）可以从上面那里进行继续执行
            putLock.unlock();
        }
        if (c == 0)
            signalNotEmpty();
        return true;
    }
    ```

### CoutDownLatch：倒计时器
* 相当于一个倒计时器，需要在应用代码中来执行countDown递减该计时器。
* 等到指定数量的线程都完成了工作，则在主线程中执行汇总等操作，如一个复杂的计算分成多个小任务，在多个子线程里执行，最后在主线程来汇总；或者是实现控制N个子线程同时开始执行。

    ```java
    class Driver2 { // ...
      void main() throws InterruptedException {
         // 大任务拆成N个小任务
         CountDownLatch doneSignal = new CountDownLatch(N);
         Executor e = ...
    
         for (int i = 0; i < N; ++i) // create and start threads
           e.execute(new WorkerRunnable(doneSignal, i));
    
         doneSignal.await();           // wait for all to finish
       }
    }
     
    class WorkerRunnable implements Runnable {
      private final CountDownLatch doneSignal;
      private final int i;
      WorkerRunnable(CountDownLatch doneSignal, int i) {
        this.doneSignal = doneSignal;
        this.i = i;
      }
      public void run() {
        try {
          doWork(i);
          doneSignal.countDown();
        } catch (InterruptedException ex) {} // return;
      }
    
      void doWork() { ... }
    }
    ```
* 控制N个子线程同时开始执行

    ```java
    class Driver { // ...
      void main() throws InterruptedException {
        // 开始控制开关
        CountDownLatch startSignal = new CountDownLatch(1);
        
        // N个线程都准备就绪开关
        CountDownLatch doneSignal = new CountDownLatch(N);
     
        for (int i = 0; i < N; ++i) // create and start threads
          new Thread(new Worker(startSignal, doneSignal)).start();
     
        doSomethingElse();            // don't let run yet
        
        // 打开控制开关
        startSignal.countDown();      // let all threads proceed
        doSomethingElse();
        
        // 等待N个线程都完成
        doneSignal.await();           // wait for all to finish
       }
     }
     
    class Worker implements Runnable {
      private final CountDownLatch startSignal;
      private final CountDownLatch doneSignal;
      Worker(CountDownLatch startSignal, CountDownLatch doneSignal) {
        this.startSignal = startSignal;
        this.doneSignal = doneSignal;
      }
      public void run() {
        try {
          // 等待主线程打开控制开关
          startSignal.await();
          
          doWork();
          
          // 该子线程完成了自身工作
          doneSignal.countDown();
        } catch (InterruptedException ex) {} // return;
      }
     
      void doWork() { ... }
    }
    ```

### CyclicBarric：栅栏
* 可以理解为一个栅栏，拦住线程直到所有线程到达则打开该栅栏让所有线程继续执行。
* 与CountDownLatch功能差不多，只是计数可重置为初始值，重复使用，具体为调用reset方法来实现，如果在执行过程中执行reset，则子线程会抛异常退出；同时可以指定一个Runnable任务在栅栏打开时来执行。由于可以重复使用，每次符合条件打开该栅栏时，都重复执行该Runnable任务。
* 内部也是使用ReentrantLock和Condition来实现的。
* CyclicBarric使用all-or-none的执行模式，即要么所有成功，要么所有失败，任何一个子线程被中断或者异常，超时退出，则所有线程都退出。
* 以下例子为处理矩阵，每个线程处理矩阵的一行，当矩阵的所有行都处理完毕之后，使用barrierAction这个Runnable实例来执行汇总操作mergeRows。

    ```java
    class Solver {
       final int N;
       final float[][] data;
       final CyclicBarrier barrier;
    
       class Worker implements Runnable {
         int myRow;
         Worker(int row) { myRow = row; }
         public void run() {
           while (!done()) {
             processRow(myRow);
    
             try {
               // 等待其他线程处理完毕则返回，
               // 即N个线程都调用了barrier.await
               barrier.await();
             } catch (InterruptedException ex) {
               return;
             } catch (BrokenBarrierException ex) {
               return;
             }
           }
         }
       }
    
       public Solver(float[][] matrix) {
         data = matrix;
         N = matrix.length;
         // 等栅栏打开，自动执行该任务
         Runnable barrierAction =
           new Runnable() { public void run() { mergeRows(...); }};
         
         // 新建栅栏，指定线程集合的大小
         barrier = new CyclicBarrier(N, barrierAction);
    
         List<Thread> threads = new ArrayList<Thread>(N);
         for (int i = 0; i < N; i++) {
           Thread thread = new Thread(new Worker(i));
           threads.add(thread);
           thread.start();
         }
    
         // wait until done
         for (Thread thread : threads)
           thread.join();
       }
    }
    ```

### Semaphore：信号量
* 限制某种资源的可用数量，通常可以用于实现池化机制。也存在公平和非公平两种实现，默认为非公平。

    ```java
    class Pool {
       private static final int MAX_AVAILABLE = 100;
       private final Semaphore available = new Semaphore(MAX_AVAILABLE, true);
     
       public Object getItem() throws InterruptedException {
         available.acquire();
         return getNextAvailableItem();
       }
    
       public void putItem(Object x) {
         if (markAsUnused(x))
           available.release();
       }
    
       // Not a particularly efficient data structure; just for demo
    
       protected Object[] items = ... whatever kinds of items being managed
       protected boolean[] used = new boolean[MAX_AVAILABLE];
    
       protected synchronized Object getNextAvailableItem() {
         for (int i = 0; i < MAX_AVAILABLE; ++i) {
           if (!used[i]) {
              used[i] = true;
              return items[i];
           }
         }
         return null; // not reached
       }
    
       protected synchronized boolean markAsUnused(Object item) {
         for (int i = 0; i < MAX_AVAILABLE; ++i) {
           if (item == items[i]) {
              if (used[i]) {
                used[i] = false;
                return true;
              } else
                return false;
           }
         }
         return false;
       }
    }
    ```