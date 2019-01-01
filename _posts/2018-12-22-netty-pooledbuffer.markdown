---
layout:     post
title:      "Netty源码分析-Netty4缓冲区Buffer性能优化之Pooled池化机制"
subtitle:   "Netty Buffer"
date:       2018-12-22 07:12:00
author:     "XYZ"
header-img: "img/article-bg1.jpg"
tags:
    - Netty源码分析
---
可以先看[Netty源码分析-缓冲区Buffer体系结构设计](https://blog.csdn.net/u010013573/article/details/85503612)再看这篇分析。
### 性能优化：在对象引用之上的对象池化机制
* 在对象引用的实现中，每当一个Buffer实例没有被引用时，则会销毁该对象实例，如被GC回收，但是Buffer对象创建时的内存分配开销是比较大的，如果频繁创建Buffer对象，频繁进行内存分配释放，则开销较大，影响性能，故在netty4中新增了对象池化机制，即Buffer对象没有被引用时，可以放到一个对象缓存池中，而不是马上销毁，当需要时，则重新从对象缓存池中取出，而不需要重新创建。

### 类结构设计
* PooledByteBuf继承于AbstractReferenceCountedByteBuf，在引用计数的基础上，添加池化机制减少对象创建，内存分配释放，提高性能。

```
// 泛型T控制底层底层存放数据的实现
// 如字节数组byte[]，Java NIO的ByteBuffer
abstract class PooledByteBuf<T> extends AbstractReferenceCountedByteBuf {
    // 核心字段recyclerHandle
    // 每个对象实例包含一个recyclerHandle字段，
    // 该字段作为中介，将该对象实例放到该对象实例所在的类的对象池（类级别字段）中
    // Handle类的value字段指向该对象
    private final Recycler.Handle<PooledByteBuf<T>> recyclerHandle;

    protected PoolChunk<T> chunk;
    protected long handle;
    protected T memory;
    protected int offset;
    protected int length;
    int maxLength;
    PoolThreadCache cache;
    private ByteBuffer tmpNioBuf;
    private ByteBufAllocator allocator;
    
    // 中间省略其他方法
    ...
    
    // 该对象引用计数为0时，释放该对象
    // 调用recycle方法，将该对象实例放到其所在类的对象缓存池中
    @Override
    protected final void deallocate() {
        if (handle >= 0) {
            final long handle = this.handle;
            this.handle = -1;
            memory = null;
            tmpNioBuf = null;
            chunk.arena.free(chunk, handle, maxLength, cache);
            chunk = null;
            recycle();
        }
    }
    
    private void recycle() {
        recyclerHandle.recycle(this);
    }

    
}
```
* PooledDirectByteBuf：直接内存的池化实现类

（1）RECYCLER为PooledDirectByteBuf类的对象实例缓存池；

（2）newInstance方法，从缓存池RECYCLER获取一个DirectByteBuf的对象实例，然后调用reuse重置该buf，然后返回给调用方。

```
final class PooledDirectByteBuf extends PooledByteBuf<ByteBuffer> {

    private static final Recycler<PooledDirectByteBuf> RECYCLER = new Recycler<PooledDirectByteBuf>() {
        @Override
        protected PooledDirectByteBuf newObject(Handle<PooledDirectByteBuf> handle) {
            return new PooledDirectByteBuf(handle, 0);
        }
    };

    static PooledDirectByteBuf newInstance(int maxCapacity) {
        PooledDirectByteBuf buf = RECYCLER.get();
        buf.reuse(maxCapacity);
        return buf;
    }
    
    ...
    
}

PooledDirectByteBuf的reuse实现：
/**
 * Method must be called before reuse this {@link PooledByteBufAllocator}
 */
final void reuse(int maxCapacity) {
    maxCapacity(maxCapacity);
    setRefCnt(1);
    setIndex0(0, 0);
    discardMarks();
}
```

* PooledHeapByteBuf：堆内存的池化实现类

```
class PooledHeapByteBuf extends PooledByteBuf<byte[]> {

    private static final Recycler<PooledHeapByteBuf> RECYCLER = new Recycler<PooledHeapByteBuf>() {
        @Override
        protected PooledHeapByteBuf newObject(Handle<PooledHeapByteBuf> handle) {
            return new PooledHeapByteBuf(handle, 0);
        }
    };

    static PooledHeapByteBuf newInstance(int maxCapacity) {
        PooledHeapByteBuf buf = RECYCLER.get();
        buf.reuse(maxCapacity);
        return buf;
    }
    
    ...

}
```

### 对象缓存池Recycler
* Recycler是使用ThreadLocal（具体为FastThreadLocal）封装了一个栈Stack作为底层容器，实现的一个轻量级、线程安全的对象缓存池。Recycler是一个泛型抽象类，根据泛型参数T指定该对象缓存池所缓存的对象的类型。

```
public abstract class Recycler<T>
```
* 以上两个类的RECYCLER域，为Recycler接口实现类的实例。如在PooledHeapByteBuf中Recycler是一个static final域：

```
private static final Recycler<PooledHeapByteBuf> RECYCLER = new Recycler<PooledHeapByteBuf>() {
    @Override
    protected PooledHeapByteBuf newObject(Handle<PooledHeapByteBuf> handle) {
        return new PooledHeapByteBuf(handle, 0);
    }
};
```
* 故需要保证RECYCLER被多个线程并发访问的安全性，在Recycler中是通过一个FastThreadLocal（Netty实现的一个ThreadLocal的变体，性能更高，具体之后分析）来保存一个Stack，通过该Stack来缓存每个线程自身的，针对该T类型的对象实例，如下是Recycler的threadLocal定义：

```
private final FastThreadLocal<Stack<T>> threadLocal = new FastThreadLocal<Stack<T>>() {
    @Override
    protected Stack<T> initialValue() {
        return new Stack<T>(Recycler.this, Thread.currentThread(), maxCapacityPerThread, maxSharedCapacityFactor,
                ratioMask, maxDelayedQueuesPerThread);
    }

    @Override
    protected void onRemoval(Stack<T> value) {
        // Let us remove the WeakOrderQueue from the WeakHashMap directly if its safe to remove some overhead
        if (value.threadRef.get() == Thread.currentThread()) {
           if (DELAYED_RECYCLED.isSet()) {
               DELAYED_RECYCLED.get().remove(value);
           }
        }
    }
};
```
#### Recycler#get方法实现

```
public final T get() {
    if (maxCapacityPerThread == 0) {
        return newObject((Handle<T>) NOOP_HANDLE);
    }
    Stack<T> stack = threadLocal.get();
    DefaultHandle<T> handle = stack.pop();
    if (handle == null) {
        handle = stack.newHandle();
        handle.value = newObject(handle);
    }
    return (T) handle.value;
}
```
1. 获取一个buf实例时，都是首先判断每个线程最大允许缓存的实例个数是否为0，默认不为0，为4k个，如下：
```
DEFAULT_INITIAL_MAX_CAPACITY_PER_THREAD = 4 * 1024; // Use 4k instances as default.
```

2. 是0，则表示不能缓存对象实例，其实就是没办法池化了，调用newObject每次创建一个对象，如下是PooledHeapByteBuf的newObject实现：
```
private static final Recycler<PooledHeapByteBuf> RECYCLER = new Recycler<PooledHeapByteBuf>() {
    @Override
    protected PooledHeapByteBuf newObject(Handle<PooledHeapByteBuf> handle) {
        return new PooledHeapByteBuf(handle, 0);
    }
};
```
3. 不是0，则通过threadLocal，从当前线程取出关联的Stack，获取栈顶的handler，handler的value即是对象实例。如下为DefaultHandler的定义：

```
static final class DefaultHandle<T> implements Handle<T> {
    private int lastRecycledId;
    private int recycleId;

    boolean hasBeenRecycled;

    private Stack<?> stack;
    private Object value;

    DefaultHandle(Stack<?> stack) {
        this.stack = stack;
    }

    @Override
    public void recycle(Object object) {
        if (object != value) {
            throw new IllegalArgumentException("object does not belong to handle");
        }
        stack.push(this);
    }
}
```
   * value保存对象实例。如果调用handler的recycle方法，则复用该对象实例，或者说复用该handler，底层实现就是将该handler重新入栈Stack，则下次该线程调用get时，则可以获取该线程上次创建的那个对象实例，实现对象的复用。
 
#### 对象缓存池Stack的容量控制

每个线程所绑定的Stack最多可以容纳该类型对象实例的个数，主要通过maxCapacityPerThread和ratioMask两个参数来控制，即控制缓存池的大小，避免一直创建并缓存新对象实例，导致内存爆炸。
   * maxCapacityPerThread：每个线程在一个对象缓存池中最多可以缓存多少个对象实例。
   1. 小于0，则使用默认大小4k个，这个是默认值；
   2. 等于0，则不缓存，即每次都是新建一个该类型对象实例，相当于不缓存；
   3. 大于0，则当Stack大小等于maxCapacityPerThread时，push入栈时，丢弃新的对象实例；
   * ratioMask：控制可以放到对象缓存池，即入栈成功的对象实例的比例，默认为第一个可以，之后每8次尝试，有一个不同的对象实例（可以是一个新的对象实例尝试8次，或者多个新的对象实例一起尝试8次，其中有一个可以入栈）可以放到缓存池中缓存，ratioMask默认值为7。
      1. 当缓冲池该类型对象少于maxCapacityPerThread时，调用push入栈时，也不一定就一直缓存新的对象实例，要看当前是第几次尝试，默认为每8次成功一个。
      2. ratioMask的值也可以通过构造函数传入，则netty会调整为2的次方减一大小，这个设计是参照了hashmap的，2的次方减一的数字，转为二进制就全部都是1了，这样方便进行位运算，如判断逻辑：++handleRecycleCount & ratioMask != 0，其中handleRecycleCount默认值为-1，如ratioMask为7，则只有第1次，第8次，第16次，这个才会返回false，才能入栈。

* Stack的核心源码实现如下：
```
1. maxCapacityPerThread的默认大小：
private static final int DEFAULT_INITIAL_MAX_CAPACITY_PER_THREAD = 4 * 1024; // Use 4k instances as default.

2. maxCapacityPerThread为0时，每次新建对象实例Recycle的get方法：
public final T get() {
    if (maxCapacityPerThread == 0) {
        return newObject((Handle<T>) NOOP_HANDLE);
    }
    Stack<T> stack = threadLocal.get();
    DefaultHandle<T> handle = stack.pop();
    if (handle == null) {
        handle = stack.newHandle();
        handle.value = newObject(handle);
    }
    return (T) handle.value;
}

3. ratioMask的默认大小：-1，则第一个入栈的实例可以成功。
// By default we allow one push to a Recycler for each 8th try on handles that were never recycled before.
// This should help to slowly increase the capacity of the recycler while not be too sensitive to allocation
// bursts.
RATIO = safeFindNextPositivePowerOfTwo(SystemPropertyUtil.getInt("io.netty.recycler.ratio", 8));

4. ratioMask调整为2的次方减一：Recycler的构造函数
protected Recycler(int maxCapacityPerThread, int maxSharedCapacityFactor,
                   int ratio, int maxDelayedQueuesPerThread) {
    // ratio为的2的次方，如传入ratio为11，则调整后ratioMask等于15               
    ratioMask = safeFindNextPositivePowerOfTwo(ratio) - 1;
    if (maxCapacityPerThread <= 0) {
        this.maxCapacityPerThread = 0;
        this.maxSharedCapacityFactor = 1;
        this.maxDelayedQueuesPerThread = 0;
    } else {
        this.maxCapacityPerThread = maxCapacityPerThread;
        this.maxSharedCapacityFactor = max(1, maxSharedCapacityFactor);
        this.maxDelayedQueuesPerThread = max(0, maxDelayedQueuesPerThread);
    }
}

5. Stack入栈实例对象判断是否入栈缓存该对象的逻辑：
（1）Stack的入栈push：size >= maxCapacity || dropHandle(item)判断

private void pushNow(DefaultHandle<?> item) {
    if ((item.recycleId | item.lastRecycledId) != 0) {
        throw new IllegalStateException("recycled already");
    }
    item.recycleId = item.lastRecycledId = OWN_THREAD_ID;

    int size = this.size;
    if (size >= maxCapacity || dropHandle(item)) {
        // Hit the maximum capacity or should drop - drop the possibly youngest object.
        return;
    }
    if (size == elements.length) {
        elements = Arrays.copyOf(elements, min(size << 1, maxCapacity));
    }

    elements[size] = item;
    this.size = size + 1;
}

（2）dropHandle的实现：
boolean dropHandle(DefaultHandle<?> handle) {
    if (!handle.hasBeenRecycled) {
        // handleRecycleCount默认值为-1，如ratioMask为7，
        // 只有第1次，第8次，第16次，这个才会返回false，才能入栈。
        if ((++handleRecycleCount & ratioMask) != 0) {
            // Drop the object.
            return true;
        }
        handle.hasBeenRecycled = true;
    }
    return false;
}
```

* 对象缓存池一般为每个类型一个，如下：
PooledHeapByteBuf类和PooledDirectByteBuf类各一个：static final域
   
```
第一个对象缓冲区实例
class PooledHeapByteBuf extends PooledByteBuf<byte[]> {

    private static final Recycler<PooledHeapByteBuf> RECYCLER = new Recycler<PooledHeapByteBuf>() {
        @Override
        protected PooledHeapByteBuf newObject(Handle<PooledHeapByteBuf> handle) {
            return new PooledHeapByteBuf(handle, 0);
        }
    };
    ...
}

第二个对象缓冲池实例
final class PooledDirectByteBuf extends PooledByteBuf<ByteBuffer> {

    private static final Recycler<PooledDirectByteBuf> RECYCLER = new Recycler<PooledDirectByteBuf>() {
        @Override
        protected PooledDirectByteBuf newObject(Handle<PooledDirectByteBuf> handle) {
            return new PooledDirectByteBuf(handle, 0);
        }
    };
    ...
}
```
