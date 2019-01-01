---
layout:     post
title:      "Netty源码分析-缓冲区Buffer体系结构设计"
subtitle:   "Netty Buffer"
date:       2018-12-21 07:12:00
author:     "XYZ"
header-img: "img/article-bg1.jpg"
tags:
    - Netty源码分析
---
### 概念
* Netty的缓冲区Buffer是一个与Netty框架相对独立的模块，在Java NIO中使用ByteBuffer作为缓冲区实现，但是ByteBuffer是使用单指针position来控制对缓冲区的读写操作的，在读写之间需用通过flip函数来切换position指针，若忘记调用flip，则可能在写了之后读不出来，用法比较繁琐，而且容易出错，关于Java NIO的设计，可以参考我的另外一篇文章[：JDK源码分析-NIO缓冲区Buffer](https://blog.csdn.net/u010013573/article/details/85418594)
* Netty的缓冲区Buffer不是对Java NIO的Buffer的简单封装，而是重新实现了一个缓冲区体系结构。与Java NIO的Buffer使用单指针不同的是，Netty的Buffer对读写操作，分别使用了读写指针，即读指针为readerIndex，写指针为writerIndex，在进行读写操作时，移动自身的指针即可，而不需要调用类似于flip函数进行指针切换。不过Netty的Buffer也提供了将自身管理的底层缓存数据结构，如字节数组byte[]，包装成Java NIO的ByteBuffer的方法。
* 除此之外，Netty的Buffer还新增了对象缓冲池机制来提供性能和复合缓冲区机制来实现对多个ByteBuf的管理。

### ByteBuf接口设计
* ByteBuf接口主要声明了对缓冲区进行操作的相关方法，如对各种基本数据类型byte, short, int, long, double，以及boolean（Java的ByteBuffer是不支持boolean的）的读写操作；数据独立副本copy，数据共享副本duplicate，数据分片slice，包装成Java的ByteBuffer，返回底层数组array，以及其他一些判断是否可读可写，是否为只读Buffer等方法。而具体的方法骨架实现主要是在AbstractByteBuf中定义。
* AbstractByteBuf主要定义了读、写、标记指针以及各种读写方法的骨架实现，而具体实现则交给具体实现类，如PooledHeapByteBuf，UnpooledHeapByteBuf。如下为AbstractByteBuf的源码核心：

```
public abstract class AbstractByteBuf extends ByteBuf {
    private static final InternalLogger logger = InternalLoggerFactory.getInstance(AbstractByteBuf.class);
    private static final String PROP_MODE = "io.netty.buffer.bytebuf.checkAccessible";
    private static final boolean checkAccessible;

    static {
        checkAccessible = SystemPropertyUtil.getBoolean(PROP_MODE, true);
        if (logger.isDebugEnabled()) {
            logger.debug("-D{}: {}", PROP_MODE, checkAccessible);
        }
    }

    static final ResourceLeakDetector<ByteBuf> leakDetector =
            ResourceLeakDetectorFactory.instance().newResourceLeakDetector(ByteBuf.class);

    int readerIndex;
    int writerIndex;
    private int markedReaderIndex;
    private int markedWriterIndex;
    private int maxCapacity;

    protected AbstractByteBuf(int maxCapacity) {
        if (maxCapacity < 0) {
            throw new IllegalArgumentException("maxCapacity: " + maxCapacity + " (expected: >= 0)");
        }
        this.maxCapacity = maxCapacity;
    }
    ...
}
```
读指针：readerIndex
写指针：writerIndex
最大容量：maxCapacity
* 字节byte的读取

（1）AbstractByteBuf的字节的读取方法：即从该Buffer中的字节读取出来，放到其他地方dst，如Java的ByteBuffer，ByteBuf，byte[]，Channel等，同时递增读指针readerIndex：
 从ByteBuf自身的readerIndex开始，读取length这么多的bytes到
 目标缓冲区dst，同时修改dst的写指针writerIndex
```
@Override
public ByteBuf readBytes(ByteBuf dst, int length) {
    if (length > dst.writableBytes()) {
        throw new IndexOutOfBoundsException(String.format(
                "length(%d) exceeds dst.writableBytes(%d) where dst is: %s", length, dst.writableBytes(), dst));
    }
    readBytes(dst, dst.writerIndex(), length);
    dst.writerIndex(dst.writerIndex() + length);
    return this;
}
```
调用getBytes方法读取ByteBuf中的数据到dst中，同时递增readerIndex，getBytes方法在具体的实现类实现
```
@Override
public ByteBuf readBytes(ByteBuf dst, int dstIndex, int length) {
    checkReadableBytes(length);
    getBytes(readerIndex, dst, dstIndex, length);
    readerIndex += length;
    return this;
}
```
（2） UnpooledHeapByteBuf：AbstractByteBuf的具体实现类，提供getBytes的实现：
```
@Override
public ByteBuf getBytes(int index, ByteBuf dst, int dstIndex, int length) {
    checkDstIndex(index, length, dstIndex, dst.capacity());
    if (dst.hasMemoryAddress()) {
        PlatformDependent.copyMemory(array, index, dst.memoryAddress() + dstIndex, length);
    } else if (dst.hasArray()) {
        getBytes(index, dst.array(), dst.arrayOffset() + dstIndex, length); // 下面是实现
    } else {
        dst.setBytes(dstIndex, array, index, length);
    }
    return this;
}
```
即通过System.arraycopy方法，将Buffer自身的字节数组array，从index，即readerIndex，开始，读取length个bytes到dst字节数组中。
```
@Override
public ByteBuf getBytes(int index, byte[] dst, int dstIndex, int length) {
    checkDstIndex(index, length, dstIndex, dst.length);
    System.arraycopy(array, index, dst, dstIndex, length);
    return this;
}
```
* 其他类型数据的读取，以读int类型数据为例

（1）AbstractByteBuf的readInt只是检查一下当前可读的字节是否够，如int，要大于或等于4个字节，同时递增readerIndex。而_getInt为abstract方法，在具体实现类中实现。
```
@Override
public int readInt() {
    checkReadableBytes0(4);
    int v = _getInt(readerIndex);
    readerIndex += 4;
    return v;
}
protected abstract int _getInt(int index);
```

（2）UnpooledHeapByteBuf：AbstractByteBuf的具体实现类，提供_getInt的实现：
```
@Override
protected int _getInt(int index) {
    return HeapByteBufUtil.getInt(array, index);
}

HeapByteBufUtil.getInt：
static int getInt(byte[] memory, int index) {
    return  (memory[index]     & 0xff) << 24 |
            (memory[index + 1] & 0xff) << 16 |
            (memory[index + 2] & 0xff) <<  8 |
            memory[index + 3] & 0xff;
}
```

### Buffer的分类
#### 以内存类型分类
* 根据内存类型，可以分为使用堆内存Buffer和直接内存Buffer，其中堆内存可以使用Java的自动垃圾回收，而如果使用了直接内存，则需要记得使用完之后，free掉该内存，否则会造成内存泄露，在netty中对内存分配和释放进行了封装，使用netty的API不需要显示调用。
* 在netty中对堆内存提供了自定义实现ByteBuf，而不是对Java NIO的ByteBuffer进行封装，而直接内存是对Java NIO的DirectByteBuffer进行了封装。
如下UnpooledDirectByteBuf的构造函数和对应的内存分配和释放函数：
```
**
 * Creates a new direct buffer.
 *
 * @param initialCapacity the initial capacity of the underlying direct buffer
 * @param maxCapacity     the maximum capacity of the underlying direct buffer
 */
public UnpooledDirectByteBuf(ByteBufAllocator alloc, int initialCapacity, int maxCapacity) {
    super(maxCapacity);
    // 中间省略一些其他逻辑
    ...

    this.alloc = alloc;
    // 核心逻辑：调用Java NIO的ByteBuffer.allocateDirect
    // 获取一块直接内存
    setByteBuffer(ByteBuffer.allocateDirect(initialCapacity));
}

// UnpooledDirectByteBuf#setByteBuffer的实现
private void setByteBuffer(ByteBuffer buffer) {
    ByteBuffer oldBuffer = this.buffer;
    if (oldBuffer != null) {
        if (doNotFree) {
            doNotFree = false;
        } else {
        // 调用freeDirect释放原来的直接内存
            freeDirect(oldBuffer);
        }
    }

    this.buffer = buffer;
    tmpNioBuf = null;
    capacity = buffer.remaining();
}

// freeDirect释放直接内存
/**
 * Free a direct {@link ByteBuffer}
 */
protected void freeDirect(ByteBuffer buffer) {
    PlatformDependent.freeDirectBuffer(buffer);
}

// 对外提供的释放直接内存方法，主要是在基类AbstractReferenceCountedByteBuf的release方法中调用，即Buffer对象都是继承于AbstractReferenceCountedByteBuf，使用引用计数的方式来协助GC回收Buffer对象实例。
@Override
protected void deallocate() {
    ByteBuffer buffer = this.buffer;
    if (buffer == null) {
        return;
    }

    this.buffer = null;

    if (!doNotFree) {
        freeDirect(buffer);
    }
}

AbstractReferenceCountedByteBuf的release0
private boolean release0(int decrement) {
    int oldRef = refCntUpdater.getAndAdd(this, -decrement);
    if (oldRef == decrement) {
        deallocate();
        return true;
    } else if (oldRef < decrement || oldRef - decrement > oldRef) {
        // Ensure we don't over-release, and avoid underflow.
        refCntUpdater.getAndAdd(this, decrement);
        throw new IllegalReferenceCountException(oldRef, -decrement);
    }
    return false;
}
```

#### 以存放ByteBuffer个数分类
根据存放ByteBuffer个数，可以分为单ByteBuffer，即底层只有一个buffer组成，或者说只有一个字节数组；复合ByteBuffer，CompositeByteBuffer，底层是包含一个多个ByteBuffer组成的列表，如下列表定义：

```
private static final class ComponentList extends ArrayList<Component> {
    ComponentList(int initialCapacity) {
        super(initialCapacity);
    }

    // Expose this methods so we not need to create a new subList just to remove a range of elements.
    @Override
    public void removeRange(int fromIndex, int toIndex) {
        super.removeRange(fromIndex, toIndex);
    }
}
```

#### 以是否进行对象池化Pooled分类
对象池化是Netty4引入的特性，主要是通过对象缓存池减少Buffer对象的创建来提高性能。具体可以查看我的另外一篇文章。[Netty源码分析-Netty4缓冲区Buffer性能优化之Pooled池化机制](https://blog.csdn.net/u010013573/article/details/85549276)

#### 处理数据类型分类，如byte, int, long等
Netty的Buffer设计没有针对每种类型单独设计一个具体实现类，而是通过方法的方式，如getInt，setInt的方式，以及根据每种类型的字节个数来操作底层bytes，来返回对应数据类型的数据。所以需要注意字节的获取应该与字节的填充严格一致，否则会导致数据错误，如先setInt，再setLong，则也要依次调用getInt(0), getLong(1)来获取。

### Buffer对象生命周期管理
#### Buffer的顶层接口和抽象类设计
先回顾一些Buffer的接口设计
* Buffer接口：声明对buffer进行容量控制，读写等操作的一些方法；
* AbstractByteBuf：定义Buffer读写操作所需的元数据，如读索引readerIndex，写索引writerIndex，以及读写等方法的骨架实现；

```
public abstract class AbstractByteBuf extends ByteBuf
```
#### 基于引用计数的对象生命周期管理
* AbstractReferenceCountedByteBuf：AbstractReferenceCountedByteBuf继承于AbstractByteBuf，添加引用计数的功能，即Buffer对象实例的生命周期是使用引用计数的方法来控制的，当一个Buffer对象没有被任何一个其他对象引用时，则可以在deallocate方法定义如何释放占用的内存。
1. AbstractReferenceCountedByteBuf类声明
 
```
public abstract class AbstractReferenceCountedByteBuf extends AbstractByteBuf {
    // 该类的对象实例的refCut字段原子更新类
    private static final AtomicIntegerFieldUpdater<AbstractReferenceCountedByteBuf> refCntUpdater =
            AtomicIntegerFieldUpdater.newUpdater(AbstractReferenceCountedByteBuf.class, "refCnt");
            
    // 每个对象实例通过refCnt来记录该对象实例被引用的次数
    // volatile保证refCnt在多个引用线程的可见性
    private volatile int refCnt;

    protected AbstractReferenceCountedByteBuf(int maxCapacity) {
        super(maxCapacity);
        refCntUpdater.set(this, 1);
    }
    
    // 中间省略其他方法
    ...
    
    // 递减引用计数，直到0，则调用deallocate完成内存释放
    private boolean release0(int decrement) {
        int oldRef = refCntUpdater.getAndAdd(this, -decrement);
        if (oldRef == decrement) {
            deallocate();
            return true;
        } else if (oldRef < decrement || oldRef - decrement > oldRef) {
            // Ensure we don't over-release, and avoid underflow.
            refCntUpdater.getAndAdd(this, decrement);
            throw new IllegalReferenceCountException(oldRef, -decrement);
        }
        return false;
    }
    
    // protected方法，由子类实现具体的内存释放逻辑
    /**
     * Called once {@link #refCnt()} is equals 0.
     */
    protected abstract void deallocate();
}
```
2. 堆内存：UnpooledHeapByteBuf实现类的deallocate实现

```
public class UnpooledHeapByteBuf extends AbstractReferenceCountedByteBuf {

    private final ByteBufAllocator alloc;
    byte[] array;
    private ByteBuffer tmpNioBuf;

    // 中间省略其他方法
    ...
    
    // deallocate将array置空
    @Override
    protected void deallocate() {
        freeArray(array);
        array = EmptyArrays.EMPTY_BYTES;
    }
}

public static final byte[] EMPTY_BYTES = {};
```
3. 直接内存：UnpooledDirectByteBuf实现类的

```
public class UnpooledDirectByteBuf extends AbstractReferenceCountedByteBuf {

    private final ByteBufAllocator alloc;

    private ByteBuffer buffer;
    private ByteBuffer tmpNioBuf;
    private int capacity;
    private boolean doNotFree;
    
    // 中间省略其他方法
    ...
    
    
    // 将buffer置null便于GC回收对象引用
    // freeDirect释放底层的直接内存
    @Override
    protected void deallocate() {
        ByteBuffer buffer = this.buffer;
        if (buffer == null) {
            return;
        }

        this.buffer = null;

        if (!doNotFree) {
            freeDirect(buffer);
        }
    }
    
}
```
#### 性能优化：在对象引用之上的对象池化机制
[Netty源码分析-Netty4缓冲区Buffer性能优化之Pooled池化机制](https://blog.csdn.net/u010013573/article/details/85549276)