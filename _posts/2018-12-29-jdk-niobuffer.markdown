---
layout:     post
title:      "JDK1.8源码分析：NIO缓冲区Buffer"
subtitle:   "JDK NIO Buffer"
date:       2018-12-29 07:12:00
author:     "XYZ"
header-img: "img/article-bg1.jpg"
tags:
    - JDK源码分析
---
### 概念
* Buffer接口是Java NIO的缓冲区的基础接口，定义了缓冲区操作的相关控制属性和操作方法。缓冲区是一个存放特定基础类型数据，如byte, char, int, long, float, double（不能是boolean），的容器，物理上是一个有界的线性数组。
* Buffer接口的具体实现类，如byte, int, long, float, double等，包含对应的类型的数组，如IntBuffer则包含一个int[]数组，而操作，即读写所需的控制属性和方法，都统一在Buffer接口定义。
* Buffer可以使用堆内存和直接内存，堆内存则使用类似于new int[capacity]来创建内部存储数据数组，每个元素都是具体类型的一个值；如果是使用直接内存，则通过Unsafe来进行内存操作，或者通过MMAP内存映射文件，如MappedByteBuffer。

### 控制属性
* capacity：buffer的容量，即数组的大小，创建Buffer实例时指定之后不可改变。
* limit：limit指向第一个不能进行读写的位置，limit不能大于capacity，主要与position配合算出数组哪些位置可以进行读写，即position和limit之间。
* position：position指向下一个可以进行读或写的位置，position不能超过limit。初始化为0，即可以从头开始读写。
* mark：标记某个位置，当之后调用reset时，可以回到这个位置，进行数据的重新读写，默认为-1。
* 位置前后或大小关系：mark <= position <= limit <= capacity

### 核心方法
* get/put：读/写数据，由每个具体的子类来实现具体的读写操作，可以是不指定索引的相对读写，即根据position位置读取下一个，如下：

```
public byte get() {
    return hb[ix(nextGetIndex())];
}
public ByteBuffer put(byte x) {
    hb[ix(nextPutIndex())] = x;
    return this;
}
```

也可以是指定索引的绝对读写：

```
public byte get(int i) {
    return hb[ix(checkIndex(i))];
}

public ByteBuffer get(byte[] dst, int offset, int length) {
    checkBounds(offset, length, dst.length);
    if (length > remaining())
        throw new BufferUnderflowException();
    System.arraycopy(hb, ix(position()), dst, offset, length);
    position(position() + length);
    return this;
}

public ByteBuffer put(int i, byte x) {
    hb[ix(checkIndex(i))] = x;
    return this;
}

public ByteBuffer put(byte[] src, int offset, int length) {
    checkBounds(offset, length, src.length);
    if (length > remaining())
        throw new BufferOverflowException();
    System.arraycopy(src, offset, hb, ix(position()), length);
    position(position() + length);
    return this;
}
```
除了调用buffer进行读写外，也可以通过Channel来对buffer进行读写，如channel.read(buffer)将channel的数据写到buffer，channel.write(buffer)从buffer读取数据并通过channel进行传输，此时使用的是相对读写。

* mark/reset：标记/回溯，mark方法调用时，可以将mark属性设置为等于或小于当前position；reset方法则将当前position设置为等于mark的值，从而回溯重新读取数据，如果没有调用mark方法，而调用了reset，由于mark值默认为-1，则会抛异常。

```
/**
 * Sets this buffer’s mark at its position.
 *
 * @return  This buffer
 */
public final Buffer mark() {
    mark = position;
    return this;
}
/**
 * Resets this buffer‘s position to the previously-marked position.
 *
 * <p> Invoking this method neither changes nor discards the mark’s
 * value. </p>
 *
 * @return  This buffer
 *
 * @throws  InvalidMarkException
 *          If the mark has not been set
 */
public final Buffer reset() {
    int m = mark;
    if (m < 0)
        throw new InvalidMarkException();
    position = m;
    return this;
}
```
* clear：清空buffer，以便对该buffer进行重新写，但是不是删除数据，而是将position重置为0，limit设置为等于capacity，mark重置为-1。

```
public final Buffer clear() {
    position = 0;
    limit = capacity;
    mark = -1;
    return this;
}
```
调用时机：当使用channel.read(buffer)从channel写数据到buffer，或者调用put方法写数据到buffer之前，调用clear先清空buffer：

```
buf.clear();     // Prepare buffer for reading
in.read(buf);    // Read data</pre></blockquote>
```

* flip：写模式切换读模式。通常在对buffer写入一些数据之后，需要从buffer读出这些数据时，则因为写时，position在递增，故读之前，需要先调用flip方法来将limit设置为等于position，然后将position重置为0，从而position和limit之间的数据则是可以读取的。

```
public final Buffer flip() {
    limit = position;
    position = 0;
    mark = -1;
    return this;
}
```
如调用channel.read(buffer)或者put写入数据到buffer，之后调用channel.write(buffer)或者get读取数据：

```
buf.put(magic);    // Prepend header
in.read(buf);      // Read data into rest of buffer
buf.flip();        // Flip buffer
out.write(buf);    // Write header + data to channel
```
* rewind：将position重置为0，一般用于重复读。

```
public final Buffer rewind() {
    position = 0;
    mark = -1;
    return this;
}
```
如下out.write从buf读取数据然后写出，然后调用rewind将position重置为0，故buf.get可以将一样的数据写到array中。

```
out.write(buf);    // Write remaining data
buf.rewind();      // Rewind buffer
buf.get(array);    // Copy data into array
```
* compact：将未读取的数据拷贝到buffer的头部。

```
public ByteBuffer compact() {
    System.arraycopy(hb, ix(position()), hb, ix(0), remaining());
    position(remaining());
    limit(capacity());
    discardMark();
    return this;
}
public final int remaining() {
    return limit - position;
}
```

即先将position到limit之间的数据拷贝到数组索引0到limit-position的位置，然后将position设置为position的下一个位置，limit设置为等于capacity。compact通常用于在读数据还没完全从buffer读完，又要往buffer写数据的场景，如下：

```
buf.clear();          // Prepare buffer for use
while (in.read(buf) >= 0 || buf.position != 0) {
   buf.flip();
   out.write(buf);
   buf.compact();    // In case of partial write
}
```
从in读取数据并写到buf，调用buf.flip将position重置为0，limit为position；然后out从buf读取数据并写出。out可能没有读完buf中的数据，则调用buf.compact将未读完的数据拷贝到buf的头部位。然后in继续往buf写，之后out继续从position为0开始读，则可以继续读取刚刚没有读完的数据。但是要注意如果buf是只读的，则不能进行这个进行compact操作，会抛ReadOnlyBufferException异常。


### 线程安全
Buffer不是线程安全的，如果Buffer被多个线程共享，则需要使用线程同步机制，如sychronzied锁，来实现线程安全。

### Buffer的类型
ByteBuffer、 CharBuffer、 DoubleBuffer、 FloatBuffer、 IntBuffer、 LongBuffer、ShortBuffer、MappedByteBuffer

 推荐阅读：
[Java NIO？看这一篇就够了！](https://mp.weixin.qq.com/s/c9tkrokcDQR375kiwCeV9w?)