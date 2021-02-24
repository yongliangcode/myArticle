---
title: Netty12# 池化内存框架流程
categories: Netty
tags: Netty
date: 2021-02-16 11:55:01
---



# 前言



本文简要梳理为什么使用池化内存？Netty使用池化内存从哪些方面提升了效率？梳理了池化内存的核心组件大体含义以及内存分配流程，勾勒池化内存的整体框架。后面文章会详细拆解每个点是如何实现的。



<!--more-->



# 使用池化内存

**为啥要使用池化内存呢？** 主要以下两点：

1.频繁申请释放堆外直接内存耗时严重影响效率

2.减少小而不连续的空闲内存（也就是内存碎片）



**Netty中又是如何体现内存池并提升效率的呢？** 

1.将申请的大块内存划分为不同的尺寸

2.不同尺寸的内存使用不同的分配算法，例如：Netty参考slab内存分配算法和Buddy（伙伴）分配算法

3.将划分的不同尺寸缓存起来，使用的时候先从缓存中获取

4.每个线程绑定了专属逻辑内存区域（PoolArena），减少资源竞争

5.使用对象池减少频繁创建销毁性能损耗（ByteBuf对象池）

6.内存用完后，按照特定算法重新合并到大块内存中，看起来像是内存池



# 内存池核心组件

**内存池尺寸划分**

Netty内存池划分了四种类型尺寸，Netty以Chunk为单位申请内存。 内存池主要指16M（默认）以下的内存，大于16M的内存分配不做缓存。

| 名称   | 范围                                      |
| ------ | ----------------------------------------- |
| tiny   | 0~512Byte，内存分配参考了slab算法         |
| small  | 512Byte~8KB，内存分配同tiny参考了slab算法 |
| normal | 8KB~16M，内存分配参考了                   |
| huge   | 大于16M                                   |



**内存池核心类** 

| 类名                   | 说明                                                         |
| ---------------------- | ------------------------------------------------------------ |
| PooledByteBufAllocator | 内存池门面类，池化内存分配入口                               |
| PoolArena              | 逻辑上的一块内存区域，管理多个PoolChunk                      |
| PoolChunk              | 连续的内存区域，一个Chunk大小为16M。每个Chunk由Page组成，每个Page大小为8KB；包含nomal类型核心内存分配算法（参考了Buddy（伙伴）分配算法） |
| PoolSubpage            | 包含8KB以下tiny和small的核心分配算法（参考了slab分配算法）   |
| PoolThreadCache        | 每个线程都有独立的PoolThreadCache，缓存了tiny类型、small类型、normal类型；分配时先从缓存中获取 |
| Recycler               | 轻量级对象缓存池，避免频繁创建和消费性能损耗                 |
| ResourceLeakDetector   | 负责内存泄漏检测                                             |



# 内存分配流程

下面通过PooledByteBufAllocator#newDirectBuffer()方法，梳理内存分配的整体流程。

```java
 @Override
 protected ByteBuf newDirectBuffer(int initialCapacity, int maxCapacity) {
   PoolThreadCache cache = threadCache.get(); // 注解@1
   PoolArena<ByteBuffer> directArena = cache.directArena;

  final ByteBuf buf;
  if (directArena != null) {
  	buf = directArena.allocate(cache, initialCapacity, maxCapacity); // 注解@2
  } else {
    buf = PlatformDependent.hasUnsafe() ?
    UnsafeByteBufUtil.newUnsafeDirectByteBuf(this, initialCapacity, maxCapacity) :
    new UnpooledDirectByteBuf(this, initialCapacity, maxCapacity);
  }
  return toLeakAwareBuffer(buf); // 注解@3
}
```

**注解@1**  从当前线程中获取PoolThreadCache，也就是每个线程都绑定了PoolThreadCache

**注解@2** 执行内存分配过程

**注解@3** 通过ResourceLeakDetector检测内存泄漏



跟踪第二步，查看内存分配过程。

```java
 PooledByteBuf<T> allocate(PoolThreadCache cache, int reqCapacity, int maxCapacity) {
        PooledByteBuf<T> buf = newByteBuf(maxCapacity); 
        allocate(cache, buf, reqCapacity);
        return buf;
}
```

PooledByteBuf从RECYCLER中获取（对象池）

```java
static PooledUnsafeDirectByteBuf newInstance(int maxCapacity) {
        PooledUnsafeDirectByteBuf buf = RECYCLER.get();
        buf.reuse(maxCapacity);
        return buf;
}
```

下面的内存分配过程，先关注主干代码框架，分配过程整体包含了三个部分： tiny&small、small、huge。

huge：大于16M，直接分配堆外直接内存。

small：先从缓存中分配，缓存没有再从内存池分配（借鉴了buddy伙伴算法）

tiny&small：先从缓存中分配，缓存没有再从内存池分配（借鉴了slab算法）

```java
private void allocate(PoolThreadCache cache, PooledByteBuf<T> buf, final int reqCapacity) {
        final int normCapacity = normalizeCapacity(reqCapacity);
        if (isTinyOrSmall(normCapacity)) { // capacity < pageSize（8KB） tiny或者small粒度
            int tableIdx;
            PoolSubpage<T>[] table;
            boolean tiny = isTiny(normCapacity);
            if (tiny) { // < 512 tiny粒度
                if (cache.allocateTiny(this, buf, reqCapacity, normCapacity)) { // 缓存分配
                    return;
                }
                tableIdx = tinyIdx(normCapacity);
                table = tinySubpagePools;
            } else { // small 粒度
                if (cache.allocateSmall(this, buf, reqCapacity, normCapacity)) {
                    return;
                }
                tableIdx = smallIdx(normCapacity);
                table = smallSubpagePools;
            }
           
            final PoolSubpage<T> head = table[tableIdx];  // 获取对应的节点

            synchronized (head) {
                final PoolSubpage<T> s = head.next;
                if (s != head) {
                    assert s.doNotDestroy && s.elemSize == normCapacity;
                    long handle = s.allocate();
                    assert handle >= 0;
                    s.chunk.initBufWithSubpage(buf, null, handle, reqCapacity);
                    incTinySmallAllocation(tiny);
                    return;
                }
            }
            synchronized (this) {
                allocateNormal(buf, reqCapacity, normCapacity);
            }

            incTinySmallAllocation(tiny);
            return;
        }
        if (normCapacity <= chunkSize) { // Normal粒度（8K~16M） 
            if (cache.allocateNormal(this, buf, reqCapacity, normCapacity)) { // 尝试先从缓存分配
                return;
            }
            synchronized (this) {
                allocateNormal(buf, reqCapacity, normCapacity);
                ++allocationsNormal;
            }
        } else {
            allocateHuge(buf, reqCapacity); // 大于16M的huge内存分配
        }
    }
```



**小结下内存分配的整体过程** 

1.从RECYCLER对象池中复用PooledByteBuf

2.每个线程绑定了缓存PoolThreadCache

3.内存分配时，先从当前线程绑定的PoolThreadCache缓存分配；缓存没有再内存池分配，不同内存尺寸使用不同的分配算法

4.每个分配的Buffer都会由ResourceLeakDetector检测内存泄漏

































