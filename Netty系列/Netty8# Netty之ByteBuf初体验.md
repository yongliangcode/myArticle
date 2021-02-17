---

title: Netty8# Netty之ByteBuf初体验
categories: Netty
tags: Netty
date: 2021-01-02 11:55:01
---



# 前言

字节的流动形成了流，Netty作为优秀的通信框架他的字节是如何流动的，本文就理一下这个事。梳理完Netty的字节流动与JDK提供的ByteBuffer一对比看下Netty方便在哪里。本分从官方文档概念原理入手梳理，然后看下源码解读下这些原理如何实现的，体验一把Netty写入数据自动扩容，探究下这个过程如何实现的。



<!--more-->



# 基本概念

**ByteBuf创建**

使用Unpooled类来创建ByteBuf，不建议使用ByteBuf的构造函数自己去创建。



**读写索引**

ByteBuf提供了两个指针readerIndex和writerIndex，分别记录读、写的开始位置。两个指针将ByteBuf分成了三个区域。

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210102104805.png)

**discardable bytes** 

这个区间的范围为0~readerIndex，已经被读过的、可废弃的区域。通过调用discardReadBytes()，可以释放discardable bytes区域。这个区域释放后，可写区域（writable bytes）部分增多。

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210102141428.png)



**readable bytes**

可读区域的范围为（writerIndex-readerIndex）



**writable bytes**

可写区域的范围为（capacity-writerIndex）



**清理索引**

调用Buffer.clear()后，读写索引全部归零，缓存buffer被释放。

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210102143322.png)



# **ByteBuf的构建**

接下来通过示例窜下上面的知识点，看下源码是如何实现的，示例中将字符串写入ByteBuf中，然后再读出来打印。

```java
@Test
public void testWriteUtf81() {
     String str1 = "瓜农";
     ByteBuf buf = Unpooled.buffer(1);
     buf.writeBytes(str1.getBytes(CharsetUtil.UTF_8));
     ByteBuf readByteBuf = ByteBufUtil.readBytes(UnpooledByteBufAllocator.DEFAULT,buf,str1.getBytes(CharsetUtil.UTF_8).length);
     System.out.print(readByteBuf.toString(CharsetUtil.UTF_8));
}
```



**源码解读** 

```java
public static ByteBuf buffer(int initialCapacity) {
	return ALLOC.heapBuffer(initialCapacity); // 注解@1
}

public ByteBuf heapBuffer(int initialCapacity) {
  return heapBuffer(initialCapacity, DEFAULT_MAX_CAPACITY); // 注解@2
}

InstrumentedUnpooledUnsafeHeapByteBuf(UnpooledByteBufAllocator alloc, int initialCapacity, int maxCapacity) {
  super(alloc, initialCapacity, maxCapacity);
}

public UnpooledHeapByteBuf(ByteBufAllocator alloc, int initialCapacity, int maxCapacity) {
  super(maxCapacity);

  if (initialCapacity > maxCapacity) {
    throw new IllegalArgumentException(String.format(
      "initialCapacity(%d) > maxCapacity(%d)", initialCapacity, maxCapacity));
  }

  this.alloc = checkNotNull(alloc, "alloc");
  setArray(allocateArray(initialCapacity)); // 注解@3
  setIndex(0, 0); // 注解@4
}
```

**注解@1** 使用ByteBufAllocator来分配ByteBuf，默认为UnpooledByteBufAllocator。

**注解@2** initialCapacity为初始容量例子中给的为16，maxCapacity为默认的DEFAULT_MAX_CAPACITY=Integer.MAX_VALUE。

**注解@3** allocateArray()的方法如下，此时使用JDK的byte[]初始化缓存区。通过setArray()，UnpooledHeapByteBuf持有byte[]缓存区。

```java
 protected byte[] allocateArray(int initialCapacity) {
   return new byte[initialCapacity];
 }

private void setArray(byte[] initialArray) {
  array = initialArray;
  tmpNioBuf = null;
}
```

**注解@4** 初始化readerIndex和writerIndex，均为0。

**小结** ByteBuf的构建通过Unpooled来分配，示例中通过UnpooledByteBufAllocator持有byte[]、 readerIndex、writerIndex、maxCapacity完成ByteBuf的初始化。 示例中array数组大小为16；readerIndex=writerIndex=0；maxCapacity=Integer.MAX_VALUE。

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210103092431.png)



#  写入数据

```java
public ByteBuf writeBytes(byte[] src, int srcIndex, int length) {
  ensureWritable(length); // 注解@5
  setBytes(writerIndex, src, srcIndex, length); // 注解@6
  writerIndex += length; // 注解@7
  return this;
}
```

**注解@5** 确保剩余的空间能够容纳需写入的数据。

具体逻辑如下：
如果写入的数据长度小于已经分配的容量空间capacity则允许直接返回；

如果写入的数据长度超过允许的最大容量maxCapacity直接抛出IndexOutOfBoundsException拒绝；

如果写入数据长度大于已经分配的空间capacity但是小于最大最大允许空间maxCapacity，则需要扩容。

```java
final void ensureWritable0(int minWritableBytes) {
    final int writerIndex = writerIndex(); // 注解@5.1
    final int targetCapacity = writerIndex + minWritableBytes; // 注解@5.2
    if (targetCapacity <= capacity()) { // 注解@5.3
        ensureAccessible();
        return;
    }
    if (checkBounds && targetCapacity > maxCapacity) { // 注解@5.4 
        ensureAccessible();
        throw new IndexOutOfBoundsException(String.format(
                "writerIndex(%d) + minWritableBytes(%d) exceeds maxCapacity(%d): %s",
                writerIndex, minWritableBytes, maxCapacity, this));
    }

    // Normalize the target capacity to the power of 2.
    final int fastWritable = maxFastWritableBytes(); // 注解@5.5 
    int newCapacity = fastWritable >= minWritableBytes ? writerIndex + fastWritable
            : alloc().calculateNewCapacity(targetCapacity, maxCapacity); // 注解@5.6

    // Adjust to the new capacity.
    capacity(newCapacity); // 注解@5.7
}
```

**注解@5.1**  获取当前写索引

**注解@5.2** 计算需要的容量

**注解@5.3 ** 与当前已分配的容量capacity进行比较

**注解@5.4** 不能超过最大允许的容量maxCapacity

**注解@5.5** fastWritable = capacity() - writerIndex

**注解@5.6** newCapacity的判断通常走到这里应该为，剩余的空间不够了。所以通常会进入alloc().calculateNewCapacity(targetCapacity, maxCapacity)。

```java
public int calculateNewCapacity(int minNewCapacity, int maxCapacity) {
        // ...
        final int threshold = CALCULATE_THRESHOLD; // 4 MiB page
        if (minNewCapacity == threshold) { // 注解@5.6.1
            return threshold;
        }
        // If over threshold, do not double but just increase by threshold.
        if (minNewCapacity > threshold) { // 注解@5.6.2
            int newCapacity = minNewCapacity / threshold * threshold;
            if (newCapacity > maxCapacity - threshold) {
                newCapacity = maxCapacity;
            } else {
                newCapacity += threshold;
            }
            return newCapacity;
        }

        // Not over threshold. Double up to 4 MiB, starting from 64.
        int newCapacity = 64;
        while (newCapacity < minNewCapacity) { // 注解@5.6.3
            newCapacity <<= 1;
        }

        return Math.min(newCapacity, maxCapacity);
    }
```

**注解@5.6.1** 如果写入的数据长度刚好为4M则返回threshold=4M

**注解@5.6.2** 如果写入的数据长度大于4M，newCapacity不再翻倍增长，通过minNewCapacity / threshold * threshold计算刚容下需要的数据即可。

**注解@5.6.3** 如果写入的数据长度小于4M，则newCapacity从64翻倍增长（128、256、512...），直到newCapacity能够容纳需要写入的数据。

**注解@5.7** 确定了要扩容的容量newCapacity后，我们看下如何扩容的。

```java
public ByteBuf capacity(int newCapacity) {
    checkNewCapacity(newCapacity);
    byte[] oldArray = array;
    int oldCapacity = oldArray.length;
    if (newCapacity == oldCapacity) {
      return this;
    }

    int bytesToCopy;
    if (newCapacity > oldCapacity) {
      bytesToCopy = oldCapacity;
    } else {
      trimIndicesToCapacity(newCapacity);
      bytesToCopy = newCapacity;
    }
    byte[] newArray = allocateArray(newCapacity); // 注解@5.7.1
    System.arraycopy(oldArray, 0, newArray, 0, bytesToCopy); // 注解@5.7.2
    setArray(newArray); // 注解@5.7.3
    freeArray(oldArray); // 注解@5.7.4
    return this;
}
```

**注解@5.7.1** 使用新的容量初始化newArray=new byte[initialCapacity]

**注解@5.7.2** 将旧的oldArray数据拷贝到新的newArray=new中

**注解@5.7.3** 将UnpooledHeapByteBuf的byte[]引用替换为newArray

**注解@5.7.4** oldArray清理操作

**注解@6** 写入数据，通过System.arraycopy将数据写入array中。

```java
public ByteBuf setBytes(int index, byte[] src, int srcIndex, int length) {
  checkSrcIndex(index, length, srcIndex, src.length);
  System.arraycopy(src, srcIndex, array, index, length);
  return this;
}
```

**注解@7** 移动writerIndex指针。



**小结：** 将上面例子的initialCapacity设置成1，促使写入数据时扩充容量。下面运行时截图：array被扩容到64，writerIndex从0位置移动到6. 

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210103110741.png)

在写入数据时，判断剩余容量是否足够；不够则需要扩容，如果写入的数据小于4M，则双倍增长，直到容纳写写入的数据。如果写入的数据大于4M，通过（minNewCapacity / threshold * threshold）计算需要扩容的大小。





# 读出数据

从buf中把刚才写入的数据（”瓜农“）读出来，通过工具类ByteBufUtil.readBytes来实现。

```java
ByteBuf readByteBuf = ByteBufUtil.readBytes(UnpooledByteBufAllocator.DEFAULT,buf,str1.getBytes(CharsetUtil.UTF_8).length);
System.out.print(readByteBuf.toString(CharsetUtil.UTF_8));
```



```java
public static ByteBuf readBytes(ByteBufAllocator alloc, ByteBuf buffer, int length) {
    boolean release = true;
    ByteBuf dst = alloc.buffer(length); // 注解@8
    try {
        buffer.readBytes(dst); // 注解@9
        release = false;
        return dst;
    } finally {
        if (release) {
            dst.release();
        }
    }
}
```

**注解@8** 重新构造了一个ByteBuf（dst）用于存储读取的数据

**注解@9** 读取数据，并移动读索引。

```java
@Override
public ByteBuf readBytes(ByteBuf dst, int length) {
    if (checkBounds) {
        if (length > dst.writableBytes()) {
            throw new IndexOutOfBoundsException(String.format(
                    "length(%d) exceeds dst.writableBytes(%d) where dst is: %s", length, dst.writableBytes(), dst));
        }
    }
    readBytes(dst, dst.writerIndex(), length); // 注解@9.1
    dst.writerIndex(dst.writerIndex() + length); // 注解@9.2
    return this;
}
```

**注解@9.1** 读取字节到新的ByteBuf。

```java

public ByteBuf readBytes(ByteBuf dst, int dstIndex, int length) {
    checkReadableBytes(length);
    getBytes(readerIndex, dst, dstIndex, length); // 注解@9.1.1
    readerIndex += length; // 注解@9.1.2
    return this;
}
```

**注解@9.1.1** 通过native api UNSAFE.copyMemory() 实现byte数组之间的拷贝

**注解@9.1.2** 源byteBuf读索引readerIndex向前移动

**注解@9.2** 数据读入新构建的缓存区dst，dst的写索引向前移动



**小结：** 示例中通过构造一个新的ByteBuf（dst），将源ByteBuf（buf）的数据读入到dst。数据读取结束后，源ByteBuf（buf）readerIndex向前移动；ByteBuf（dst）的writerIndex向前移动。

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210103125302.png)







