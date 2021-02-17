---
title: Netty11# 非池化内存分配
categories: Netty
tags: Netty
abbrlink: f74d0239
date: 2021-02-10 11:55:01
---



# 前言

非池化内存的分配由UnpooledByteBufAllocator负责，本文梳理下由其负责分配的堆内存和堆外内存如何实现的 。

Netty在非池化堆内存分配上Java9与Java8以下版本有啥不同呢？Netty堆外内存回收默认机制使用JDK提供的Cleaner吗？



<!--more-->



# 非池化堆内内存分配



下面这小段代码摘自UnpooledByteBufAllocator#newHeapBuffer，通过此方法分析非池化堆内存的分配。

```java
@Override
protected ByteBuf newHeapBuffer(int initialCapacity, int maxCapacity) {
  return PlatformDependent.hasUnsafe() ?
  new InstrumentedUnpooledUnsafeHeapByteBuf(this, initialCapacity, maxCapacity) :
  new InstrumentedUnpooledHeapByteBuf(this, initialCapacity, maxCapacity);
}
```



**解读：** 堆内内存分配由newHeapBuffer方法负责，如果平台支持Unsafe则创建InstrumentedUnpooledUnsafeHeapByteBuf，否则创建

InstrumentedUnpooledHeapByteBuf，下图为非池化相关类图，分别从两个类UnpooledDirectByteBuf和UnpooledHeapByteBuf延伸开来。



![](https://gitee.com/laoliangcode/md-picture/raw/master/img/%E9%9D%9E%E6%B1%A0%E5%8C%96%E7%B1%BB%E5%9B%BE.png)



还是聚集到堆内存的分配上来，主要分析上图中红色部分。InstrumentedUnpooledUnsafeHeapByteBuf和InstrumentedUnpooledHeapByteBuf有啥区别？



### InstrumentedUnpooledUnsafeHeapByteBuf



下面看下InstrumentedUnpooledUnsafeHeapByteBuf其内存分配的行为allocateArray().

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210206144813.png)

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210206144919.png)

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210206145307.png)

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210206180549.png)



**注解@1** 调用了父类UnpooledUnsafeHeapByteBuf的allocateArray()

**注解@2** 父类UnpooledUnsafeHeapByteBuf调用了PlatformDependent#allocateUninitializedArray

**注解@3/@4**  Java9以上版本：如果待分配的内存小于1K使用堆内存，待分配的内存大于等于1K使用堆外内存。

Java8以及以下版本全部在堆内存分配



<u>**小结：**  使用InstrumentedUnpooledUnsafeHeapByteBuf进行内存分配时：</u>

<u>Java9以及以上版本：如果待分配的内存小于1K使用堆内存；待分配的内存大于等于1K使用堆外内存（调用底层PlatformDependent#allocateUninitializedArray）。</u>

<u>Java8以及以下版本：使用堆内存分配。</u>



### InstrumentedUnpooledHeapByteBuf

下面为InstrumentedUnpooledHeapByteBuf的内存分配allocateArray().

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210206150611.png)

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210206150640.png)



**注解@1** 调用父类 UnpooledHeapByteBuf的内存分配

**注解@2** UnpooledHeapByteBuf的通过new byte直接在堆内存分配



<u>**小结：** InstrumentedUnpooledHeapByteBuf直接在堆内存分配空间。</u>



### 数据获取方式



**UnpooledUnsafeHeapByteBuf数据获取** 

UnpooledUnsafeHeapByteBuf的数据获取方式getByte()

```java
 @Override
 protected byte _getByte(int index) {
 		return UnsafeByteBufUtil.getByte(array, index);
 }
```

该方法调用UnsafeByteBufUtil的getByte，跟进去看下

```java
static byte getByte(byte[] array, int index) {
	return PlatformDependent.getByte(array, index);
}
```

底层通过UNSAFE.getByte这种地址+偏移量的方式获取内存中的数据。

```java
static byte getByte(byte[] data, int index) {
    return UNSAFE.getByte(data, BYTE_ARRAY_BASE_OFFSET + index);
}
```



**UnpooledHeapByteBuf数据获取** 



UnpooledHeapByteBuf数据获取方式_getByte()

```java
@Override
    protected byte _getByte(int index) {
        return HeapByteBufUtil.getByte(array, index);
    }
```

该方法调用HeapByteBufUtil.getByte，跟进去看下，即直接从数组中获取数据。

```java
static byte getByte(byte[] memory, int index) {
     return memory[index];
 }
```



<u>**小结：** UnpooledUnsafeHeapByteBuf通过UNSAFE.getByte这种地址+偏移量的方式获取内存中的数据；UnpooledHeapByteBuf通过数组直接从堆内存获取。</u>



### 非池化堆内存分配总结

<u>当使用Netty非池化进行堆内存分配时：</u>

<u>1.Java8及其以下版本：直接在堆空间分配内存。</u>

<u>2.Java9及其以上版本：如果系统支持Unsafe时（通常都是支持的），对于小于1K（默认）的在堆内存分配，大于1K的分配堆外内存；如果系统不支持Unsafe直接在堆内存分配；默认大小阈值可以通过-Dio.netty.uninitializedArrayAllocationThreshold来指定，默认为1024字节。</u>

<u>3.堆内存数据获取通过数组实现；堆外内存获取通过UNSAFE.getByte这种地址+偏移量的方式获取。</u>





# 非池化堆外内存分配

下面这段代码摘自UnpooledByteBufAllocator#newDirectBuffer方法，通过此方法分析非池化堆外存的分配。

```java
@Override
protected ByteBuf newDirectBuffer(int initialCapacity, int maxCapacity) {
  final ByteBuf buf;
  if (PlatformDependent.hasUnsafe()) {
  	buf = noCleaner ? new InstrumentedUnpooledUnsafeNoCleanerDirectByteBuf(this, initialCapacity, maxCapacity) :
  	new InstrumentedUnpooledUnsafeDirectByteBuf(this, initialCapacity, maxCapacity);
  } else {
  	buf = new InstrumentedUnpooledDirectByteBuf(this, initialCapacity, maxCapacity);
  }
  return disableLeakDetector ? buf : toLeakAwareBuffer(buf);
}
```

**解读：** 平台不支持支持Unsafe，构造InstrumentedUnpooledDirectByteBuf；平台支持Unsafe并且noCleaner=true构造InstrumentedUnpooledUnsafeNoCleanerDirectByteBuf；平台支持Unsafe并且noCleaner=false，构造InstrumentedUnpooledUnsafeDirectByteBuf。那问题来了，这三个有啥区别呢？



### noCleaner

```java
 noCleaner = tryNoCleaner && PlatformDependent.hasUnsafe()
                && PlatformDependent.hasDirectBufferNoCleanerConstructor();
```

**解读** ：三个判断条件一个一个来看：

@1 tryNoCleaner=PlatformDependent.useDirectBufferNoCleaner()该方法在前一篇文章中也分析过，当maxDirectMemory!=0 && 支持Unsafe && DirectByteBuffer的构造函数可用时，tryNoCleaner = true。

@2 PlatformDependent.hasUnsafe() 在上一篇文章中分析过具体 UNSAFE_UNAVAILABILITY_CAUSE == null，平台不支持UNSAFE会将异常封装在UNSAFE_UNAVAILABILITY_CAUSE中，等于null意味着平台支持Unsafe。

@3 PlatformDependent.hasDirectBufferNoCleanerConstructor() 指的是通过反射DirectByteBuffer构造器对象是否可用。



所以看出，通常在实践中（Linux、JDK8）上面这些条件都是支持的，也就是Netty默认noCleaner为true。



### InstrumentedUnpooledUnsafeNoCleanerDirectByteBuf

下面看下堆外内存的分配和回收：

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210208102959.png)



**注解@1**  堆外内存分配底层调用了PlatformDependent0#allocateDirectNoCleaner方法，malloc()返回获得内存空间的首地址，失败返回null，然后根据返回的内存地址调用DirectByteBuffer构造函数分配堆外内存。 

```java
static ByteBuffer allocateDirectNoCleaner(int capacity) {
  // Calling malloc with capacity of 0 may return a null ptr or a memory address that can be used.
  // Just use 1 to make it safe to use in all cases:
  // See: http://pubs.opengroup.org/onlinepubs/009695399/functions/malloc.html

  /**
  * malloc()返回获得内存空间的首地址，失败返回null
  */
	return newDirectBuffer(UNSAFE.allocateMemory(Math.max(1, capacity)), capacity);
}
```

**注解@2** 堆外内存释放底层调用了PlatformDependent0#freeMemory方法，通过UNSAFE.freeMemory释放堆外内存。

```java
static void freeMemory(long address) {
    UNSAFE.freeMemory(address);
}
```



### InstrumentedUnpooledUnsafeDirectByteBuf

下面看下堆外内存的分配与回收

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210209094247.png)



**注解@1** 底层调用ByteBuffer#allocateDirect来分配堆外内存，具体为直接new DirectByteBuffer()

```java
public static ByteBuffer allocateDirect(int capacity) {
    return new DirectByteBuffer(capacity);
}
```

**注解@2** 释放堆外内存调用了PlatformDependent#freeDirectBuffer()底层调用CLEANER.freeDirectBuffer实现

```java
public static void freeDirectBuffer(ByteBuffer buffer) {
   CLEANER.freeDirectBuffer(buffer);
}
```



### nstrumentedUnpooledDirectByteBuf



![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210210074748.png)



**注解@1** 堆外内存分配同InstrumentedUnpooledUnsafeDirectByteBuf，通过父类UnpooledDirectByteBuf#allocateDirect调用ByteBuffer#allocateDirect来分配堆外内存，堆内存直接new DirectByteBuffer

```java
public static ByteBuffer allocateDirect(int capacity) {
    return new DirectByteBuffer(capacity);
}
```



**注解@2** 堆外内存释放同InstrumentedUnpooledUnsafeDirectByteBuf，通过父类UnpooledDirectByteBuf#freeDirect调用底层调用CLEANER.freeDirectBuffer实现。

```java
public static void freeDirectBuffer(ByteBuffer buffer) {
    CLEANER.freeDirectBuffer(buffer);
}
```



**小结：** @1 InstrumentedUnpooledUnsafeNoCleanerDirectByteBuf的内存释放调用UNSAFE.freeMemory(address)实现

​			 @2 InstrumentedUnpooledUnsafeDirectByteBuf和nstrumentedUnpooledDirectByteBuf的内存释放是一样的，使用了JDK提供的CLEANER.freeDirectBuffer(buffer)。

​			@3 Netty默认自行管理堆外内存的分配与释放，并未使用JDK提供的释放方式，而是通过底层API自行释放。



### 堆外内存数据获取

下面是三个buffer获取内存数据的方式：

InstrumentedUnpooledUnsafeNoCleanerDirectByteBuf通过UnpooledUnsafeDirectByteBuf#_getByte()获取数据

InstrumentedUnpooledUnsafeDirectByteBuf通过UnpooledUnsafeDirectByteBuf#_getByte()获取数据

nstrumentedUnpooledDirectByteBuf通过UnpooledDirectByteBuf#_getByte()获取数据



InstrumentedUnpooledUnsafeNoCleanerDirectByteBuf和InstrumentedUnpooledUnsafeDirectByteBuf获取方式一样；下面看下这两类获取方式。



**UnpooledUnsafeDirectByteBuf获取内存数据** 

```java
protected byte _getByte(int index) {
     return UnsafeByteBufUtil.getByte(addr(index)); // 注解@1
}
```

**注解@1** 底层通过UNSAFE.getByte(address)这种“地址+偏移量” 的方式获取内存数据。



**UnpooledDirectByteBuf获取内存数据** 

```java
protected byte _getByte(int index) {
    return buffer.get(index); // 注解@2
}
```

**注解@2** 通过ByteBuffer#get方式获取，底层通过Bits#unsafe.getByte(address)获取内存数据。



### 非池化堆外内存总结



Netty在堆外内存分配上，在系统支持的情况下，默认自己通过UNSAFE.freeMemory去释放内存，也就是noCleaner，没有使用JDK提供的Cleaner释放机制。至于为啥netty选择自己实现，不用JDK提供的方式，主要考虑性能原因，与JDK Bits设计有关系。



















