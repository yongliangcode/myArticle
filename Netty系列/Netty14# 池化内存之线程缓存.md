---
title: Netty14# 池化内存之线程缓存
categories: Netty
tags: Netty
date: 2021-03-06 11:55:01
---



# 前言 

在前面文章『Netty12# 池化内存框架流程』Netty会将不同的内存尺寸缓存起来，每个线程绑定了专属逻辑内存区域（PoolArena），减少资源竞争。每个线程绑定了缓存PoolThreadCache，内存分配时，先从当前线程绑定的PoolThreadCache缓存分配。下图为涉及到相关类的关系图：



![](https://gitee.com/laoliangcode/md-picture/raw/master/img/%E6%9C%AA%E5%91%BD%E5%90%8D%E6%96%87%E4%BB%B6%20(3).png)



工作过程：

@1 通过引导类传入NioEventLoopGroup，线程工厂创建的线程均为FastThreadLocalThread

@2 FastThreadLocalThread持有InternalThreadLocalMap（内部维护一个对象数组）

@3 当通过PooledByteBufAllocator#newDirectBuffer分配内存时，通过调用PoolThreadLocalCache#get()完成对InternalThreadLocalMap的第一次填充，对象数组下标为线程索引号，其对应的值为PoolThreadCache。

@4 PoolThreadCache是被当前线程缓存的对象



<!--more-->



# 线程缓存梳理

PoolThreadLocalCache继承了线程类FastThreadLocal，FastThreadLocal的作用类似ThreadLocal，传递线程上下文变量。本小节梳理PoolThreadLocalCache工作流程。



### 构造函数

```java
 final class PoolThreadLocalCache extends FastThreadLocal<PoolThreadCache> {
        private final boolean useCacheForAllThreads;

        PoolThreadLocalCache(boolean useCacheForAllThreads) {
            this.useCacheForAllThreads = useCacheForAllThreads;
        }
   // ...
 }
```

小结：构造函数就一个变量useCacheForAllThreads，默认true，使用线程缓存，可以通过-Dio.netty.allocator.useCacheForAllThread制定。



### 初始化方法赋值

```java
@Override
protected synchronized PoolThreadCache initialValue() {
    final PoolArena<byte[]> heapArena = leastUsedArena(heapArenas);
    final PoolArena<ByteBuffer> directArena = leastUsedArena(directArenas); // 注解@1

    final Thread current = Thread.currentThread();
    if (useCacheForAllThreads || current instanceof FastThreadLocalThread) { // 注解@2
      final PoolThreadCache cache = new PoolThreadCache(
        heapArena, directArena, tinyCacheSize, smallCacheSize, normalCacheSize,
        DEFAULT_MAX_CACHED_BUFFER_CAPACITY, DEFAULT_CACHE_TRIM_INTERVAL);

      if (DEFAULT_CACHE_TRIM_INTERVAL_MILLIS > 0) {
        final EventExecutor executor = ThreadExecutorMap.currentExecutor();
        if (executor != null) {
          executor.scheduleAtFixedRate(trimTask, DEFAULT_CACHE_TRIM_INTERVAL_MILLIS,
          DEFAULT_CACHE_TRIM_INTERVAL_MILLIS, TimeUnit.MILLISECONDS);
        }
      }
      return cache;
    }
    // No caching so just use 0 as sizes.
    return new PoolThreadCache(heapArena, directArena, 0, 0, 0, 0, 0); // 注解@3
}
```

注解@1：heapArenas/directArenas：Arena数组，元素为HeapArena/DirectArena。调用了同一个方法leastUsedArena()。

```java
private <T> PoolArena<T> leastUsedArena(PoolArena<T>[] arenas) {
  if (arenas == null || arenas.length == 0) {
    return null;
  }

  PoolArena<T> minArena = arenas[0];
  for (int i = 1; i < arenas.length; i++) {
    PoolArena<T> arena = arenas[i];
    if (arena.numThreadCaches.get() < minArena.numThreadCaches.get()) {
      minArena = arena;
    }
  }

 return minArena;
}
```

每个线程都会绑定PoolArena，在leastUsedArena()轮询一遍，获取当前绑定线程数最少的PoolArena。



注解@2：当useCacheForAllThreads=true（默认true）和当前thread属于FastThreadLocalThread才构造PoolThreadCache进行缓存。

DEFAULT_CACHE_TRIM_INTERVAL_MILLIS：定时释放缓存。默认为0表示关闭，可以通过-Dio.netty.allocator.cacheTrimIntervalMillis指定。

```java
private final Runnable trimTask = new Runnable() {
        @Override
        public void run() {
            PooledByteBufAllocator.this.trimCurrentThreadCache();
        }
};
```

```java
public boolean trimCurrentThreadCache() {
        PoolThreadCache cache = threadCache.getIfExists();
        if (cache != null) {
            cache.trim();
            return true;
        }
        return false;
}
```

通过定时调度调用PoolThreadCache的trim()方法将线程缓存释放。



注解@3：禁用线程缓存依然是构造PoolThreadCache，只是传入的参数为0.



小结：初始化赋值过程实际是为了创建一个PoolThreadCache对象。



### 初始化方法调用

初始化方法PoolThreadLocalCache#initialValue()什么时候调用的呢？在第一次调用FastThreadLocal#get()时进行的初始化。例如：在PooledByteBufAllocator#newDirectBuffer()方法中PoolThreadCache cache = threadCache.get();

```java
public final V get() {
        InternalThreadLocalMap threadLocalMap = InternalThreadLocalMap.get();
        Object v = threadLocalMap.indexedVariable(index);
        if (v != InternalThreadLocalMap.UNSET) {
            return (V) v;
        }

        return initialize(threadLocalMap);
}
```

初始化后，会将放入InternalThreadLocalMap, 其中维护了一个对象数组Object[]，下标即为index，每创建一个线程FastThreadLocal，都会递增一个index。

```java
private V initialize(InternalThreadLocalMap threadLocalMap) {
        V v = null;
        try {
            v = initialValue();
        } catch (Exception e) {
            PlatformDependent.throwException(e);
        }
				// 放入InternalThreadLocalMap中实际为数组
        threadLocalMap.setIndexedVariable(index, v); 
        addToVariablesToRemove(threadLocalMap, this);
        return v;
    }
```

```java
private final int index;

public FastThreadLocal() {
  // 每创建一个fast线程都会分配一个index
	index = InternalThreadLocalMap.nextVariableIndex();
}
```



小结：初始化方法initialValue()，在第一次调用threadCache.get()的时候执行。并将初始化的结果PoolThreadCache放入InternalThreadLocalMap（实际为对象数组）。



### FastThreadLocalThread的调用 

在初始化赋值注解@2中，只有满足两个条件才会缓存，if (useCacheForAllThreads || current instanceof FastThreadLocalThread) 。其中一个是当前线程属于FastThreadLocalThread。那问题是我们有用FastThreadLocalThread吗？

在通过引导类构建Netty客户端和服务端时会传入EventLoopGroup，我们以NioEventLoopGroup看下它创建的是什么线程。

```
EventLoopGroup group = new NioEventLoopGroup();
```

通过NioEventLoopGroup的构造函数可以跟到下面内容：

```java
protected MultithreadEventExecutorGroup(int nThreads, Executor executor,
                                            EventExecutorChooserFactory chooserFactory, Object... args) {
        if (nThreads <= 0) {
            throw new IllegalArgumentException(String.format("nThreads: %d (expected: > 0)", nThreads));
        }

        if (executor == null) {
            executor = new ThreadPerTaskExecutor(newDefaultThreadFactory());
        }
        // ... 
}
```

通过newDefaultThreadFactory()看下线程工厂类DefaultThreadFactory中如何创建线程的。

```java
	@Override
  public Thread newThread(Runnable r) {
        Thread t = newThread(FastThreadLocalRunnable.wrap(r), prefix + nextId.incrementAndGet());
        try {
            if (t.isDaemon() != daemon) {
                t.setDaemon(daemon);
            }

            if (t.getPriority() != priority) {
                t.setPriority(priority);
            }
        } catch (Exception ignored) {
            // Doesn't matter even if failed to set.
        }
        return t;
	}

  protected Thread newThread(Runnable r, String name) {
        return new FastThreadLocalThread(threadGroup, r, name); // 实际为FastThreadLocalThread实例。
	}
```

通过newThread创建的实际为FastThreadLocalThread实例。



小结：我们通过Bootstrap引导类传入的NioEventLoopGroup，使用的线程为FastThreadLocalThread。



# 构造缓存数组

PoolThreadCache 缓存了三个级别的缓存类型，分别为tiny、small、normal。

### 构造函数 

```java
PoolThreadCache(PoolArena<byte[]> heapArena, PoolArena<ByteBuffer> directArena,
                    int tinyCacheSize, int smallCacheSize, int normalCacheSize,
                    int maxCachedBufferCapacity, int freeSweepAllocationThreshold) {
  
      if (directArena != null) {
        
        tinySubPageDirectCaches = createSubPageCaches(
          tinyCacheSize, PoolArena.numTinySubpagePools, SizeClass.Tiny);
        
        smallSubPageDirectCaches = createSubPageCaches(
          smallCacheSize, directArena.numSmallSubpagePools, SizeClass.Small);

        numShiftsNormalDirect = log2(directArena.pageSize);
        normalDirectCaches = createNormalCaches(
          normalCacheSize, maxCachedBufferCapacity, directArena);

        directArena.numThreadCaches.getAndIncrement();
      }
  // ...
}
```

**参数说明** 

heapArena：最少持有线程数(使用率最少)的逻辑堆内存PoolArena，PoolArena[]数组长度默认为核数的2倍

directArena：最少持有线程数(使用率最少)的逻辑堆外直接内存PoolArena，PoolArena[]数组长度默认为核数的2倍

tinyCacheSize：默认tiny类型缓存池大小512

smallCacheSize：默认small类型缓存池大小为256

normalCacheSize：默认normal类型缓存池大小为64

maxCachedBufferCapacity：默认为32KB，用于限制normal缓存数组的长度

freeSweepAllocationThreshold：默认8192，分配次数阈值，超过后释放内存池



构造函数中，主要给三种类型的缓存数组赋值，包括堆内存和堆外直接内存，结构一致，只走查堆外直接内存。

```java
// tiny类型缓存数组
private final MemoryRegionCache<ByteBuffer>[] tinySubPageDirectCaches;
// small类型缓存数组
private final MemoryRegionCache<ByteBuffer>[] smallSubPageDirectCaches;
// normal类型缓存数组
private final MemoryRegionCache<ByteBuffer>[] normalDirectCaches;
```



### createSubPageCaches

tiny类型缓存数组与small类型缓存数组调用调用相同的createSubPageCaches()方法。

```java
private static <T> MemoryRegionCache<T>[] createSubPageCaches(
            int cacheSize, int numCaches, SizeClass sizeClass) {
        if (cacheSize > 0 && numCaches > 0) {
            @SuppressWarnings("unchecked")
            MemoryRegionCache<T>[] cache = new MemoryRegionCache[numCaches];
            for (int i = 0; i < cache.length; i++) {
                cache[i] = new SubPageMemoryRegionCache<T>(cacheSize, sizeClass);
            }
            return cache;
        } else {
            return null;
        }
    }
```

**方法入参**

cacheSize：MemoryRegionCache包含队列Queue的大小，tiny类型512，small类型256

numCaches：不同缓存类型的规格数量。

tiny类型规格数量为32，计算方式 PoolArena.numTinySubpagePools=512 >>> 4=32

small类型规格数量为4，计算方式 heapArena.numSmallSubpagePools=pageShifts - 9=13 - 9 = 4



小结：tiny类型会构建MemoryRegionCache的数组长度为32，每个数组元素为SubPageMemoryRegionCache（包含Queue的大小为512）；

small类型会构建MemoryRegionCache的数组长度为4，每个数组元素为SubPageMemoryRegionCache（包含Queue的大小为256）



### createNormalCaches

Normal类型缓存数组调用createNormalCaches()方法。

```java
private static <T> MemoryRegionCache<T>[] createNormalCaches(
            int cacheSize, int maxCachedBufferCapacity, PoolArena<T> area) {
        if (cacheSize > 0 && maxCachedBufferCapacity > 0) {
            int max = Math.min(area.chunkSize, maxCachedBufferCapacity);
            int arraySize = Math.max(1, log2(max / area.pageSize) + 1);
            @SuppressWarnings("unchecked")
            MemoryRegionCache<T>[] cache = new MemoryRegionCache[arraySize];
            for (int i = 0; i < cache.length; i++) {
                cache[i] = new NormalMemoryRegionCache<T>(cacheSize);
            }
            return cache;
        } else {
            return null;
        }
}
```

**方法入参** 

cacheSize：Normal类型64

maxCachedBufferCapacity：32K

**数组大小计算** 

int arraySize = Math.max(1, log2(max / area.pageSize) + 1);

int max：maxCachedBufferCapacity=32KB；area.chunkSize = 16M，Max.min(32KB，16M) = 32K

pageSize：area.pageSize=8K

log2(max / area.pageSize)，代入log2(4)公式 

```java
 private static int log2(int val) {
        int res = 0;
        while (val > 1) {
            val >>= 1;
            res++;
        }
        return res;
    }
```

经过计算数组大小arraySize= 3



小结：Normal类型会构建MemoryRegionCache的数组长度为3，每个数组元素为SubPageMemoryRegionCache（包含Queue的大小为64）。



# 缓存数组结构



### 缓存数组结构

上面tiny、small、normal无论哪种类型都在构建MemoryRegionCache数组，通过看下MemoryRegionCache的结构看下缓存的不同点。

```java
private abstract static class MemoryRegionCache<T> {
        private final int size;
        private final Queue<Entry<T>> queue;
        private final SizeClass sizeClass;
        private int allocations;

        MemoryRegionCache(int size, SizeClass sizeClass) {
            this.size = MathUtil.safeFindNextPositivePowerOfTwo(size);
            queue = PlatformDependent.newFixedMpscQueue(this.size);
            this.sizeClass = sizeClass;
        }
		// ...
}
```

以对外直接内存Queue<Entry<T>> queue封装的均为ByteBuffer。下面看下不同类型缓存的ByteBuffer是如何分布的。

```java
 boolean add(PoolArena<?> area, PoolChunk chunk, ByteBuffer nioBuffer,
                long handle, int normCapacity, SizeClass sizeClass) {
        MemoryRegionCache<?> cache = cache(area, normCapacity, sizeClass);
        if (cache == null) {
            return false;
        }
        return cache.add(chunk, nioBuffer, handle);
    }
```

通过cache()方法来判断缓存的三种类型判断

```java
private MemoryRegionCache<?> cache(PoolArena<?> area, int normCapacity, SizeClass sizeClass) {
        switch (sizeClass) {
        case Normal:
            return cacheForNormal(area, normCapacity);
        case Small:
            return cacheForSmall(area, normCapacity);
        case Tiny:
            return cacheForTiny(area, normCapacity);
        default:
            throw new Error();
        }
    }
```

下面逐个看看每个里面的结构，先看Tiny类型。

```java
private MemoryRegionCache<?> cacheForTiny(PoolArena<?> area, int normCapacity) {
        // idx = normCapacity 除以 16
        int idx = PoolArena.tinyIdx(normCapacity);
        if (area.isDirect()) {
            // tiny有32个规格类型即32个MemoryRegionCache实例
          	// 例如：normCapacity=32 则返回第2个数组元素MemoryRegionCache
            return cache(tinySubPageDirectCaches, idx);
        }
        return cache(tinySubPageHeapCaches, idx);
}
```

```java
 static int tinyIdx(int normCapacity) {
        return normCapacity >>> 4; // 相当于直接将normCapacity除以16
 }
```

过程：Tiny类型中根据需要分配的大小除以16
示例1：normCapacity=0，idx=0，返回 tinySubPageDirectCaches[0]，也就是 tinySubPageDirectCaches[0]没有缓存。

示例2：normCapacity=16，idx=1，返回 tinySubPageDirectCaches[1]，也就是 tinySubPageDirectCaches[1]中的Queue的buffer大小均为16字节。

示例3：normCapacity=32，idx=2，返回 tinySubPageDirectCaches[2]，也就是 tinySubPageDirectCaches[2]中的Queue的buffer大小均为32字节。

...

示例4：   normCapacity=496，idx=31，返回 tinySubPageDirectCaches[31]，也就是 tinySubPageDirectCaches[31]中的Queue的buffer大小均为496字节。



接着看Small类型的存储格式

```java
 private MemoryRegionCache<?> cacheForSmall(PoolArena<?> area, int normCapacity) {
    int idx = PoolArena.smallIdx(normCapacity);
    if (area.isDirect()) {
         return cache(smallSubPageDirectCaches, idx);
    }
    return cache(smallSubPageHeapCaches, idx);
 }
```

```java
static int smallIdx(int normCapacity) {
        int tableIdx = 0;
        int i = normCapacity >>> 10;
        while (i != 0) {
            i >>>= 1;
            tableIdx ++;
        }
        return tableIdx;
    }
```

过程：Small类型的分配normCapacity >>> 10，代入计算看看System.out.println(smallIdx(normCapacity))。

示例1：normCapacity=512，idx = 0，返回smallSubPageDirectCaches[0]，也就是smallSubPageDirectCaches[0]中Queue的Buffer大小均为512字节。

示例2：normCapacity=1024，idx = 1，返回smallSubPageDirectCaches[1]，也就是smallSubPageDirectCaches[1]中Queue的Buffer大小均为1024字节。

示例3：normCapacity=2048，idx = 2，返回smallSubPageDirectCaches[2]，也就是smallSubPageDirectCaches[2]中Queue的Buffer大小均为2048字节。

示例3：normCapacity=4096，idx = 3，返回smallSubPageDirectCaches[3]，也就是smallSubPageDirectCaches[3]中Queue的Buffer大小均为4096字节。



最后看下Normal类型

```java
private MemoryRegionCache<?> cacheForNormal(PoolArena<?> area, int normCapacity) {
        if (area.isDirect()) {
            int idx = log2(normCapacity >> numShiftsNormalDirect);
            return cache(normalDirectCaches, idx);
        }
        int idx = log2(normCapacity >> numShiftsNormalHeap);
        return cache(normalHeapCaches, idx);
    }
```

过程：先把numShiftsNormalDirect算下

numShiftsNormalDirect = log2(directArena.pageSize) = log2(8192) = 13.

代入公式计算下 int idx = log2(normCapacity >> 13)

示例1：normCapacity=8192（8K），idx = 0，返回normalDirectCaches[0]，也就是normalDirectCaches[0]中Queue的Buffer大小均为8KB。

示例2：normCapacity=16384（16K），idx = 1，返回normalDirectCaches[1]，也就是normalDirectCaches[0]中Queue的Buffer大小均为16KB。

示例3：normCapacity=32768（32K），idx = 2，返回normalDirectCaches[2]，也就是normalDirectCaches[0]中Queue的Buffer大小均为32KB。



小结：通过上面的过程分析，能够得出MemoryRegionCache的缓存结构如下，其中每个数组元素的队列中缓存的大小都是相同的，也就是Queue<Entry<T>> queue中的T即ByteBuffer。

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/%E7%BC%93%E5%AD%98%E7%BB%93%E6%9E%84%E5%9B%BE%E7%A4%BA.png)



### 缓存归队

再回到添加方法中，上面通过cache()方法分析了缓存数组结构，返回不同类型的MemoryRegionCache。

```java
boolean add(PoolArena<?> area, PoolChunk chunk, ByteBuffer nioBuffer,
                long handle, int normCapacity, SizeClass sizeClass) {
        MemoryRegionCache<?> cache = cache(area, normCapacity, sizeClass);
        if (cache == null) {
            return false;
        }
        return cache.add(chunk, nioBuffer, handle); // 注解@1
 }
```

注解@1：下面是将chunk（真正一块连续内存）, nioBuffer, handle（指向内存的指针）放入队列的过程。

```java
 public final boolean add(PoolChunk<T> chunk, ByteBuffer nioBuffer, long handle) {
     Entry<T> entry = newEntry(chunk, nioBuffer, handle); // 注解@2 
     boolean queued = queue.offer(entry); // 注解@3
     if (!queued) {
       // If it was not possible to cache the chunk, immediately recycle the entry
       entry.recycle();
     }

     return queued;
}
```

注解@2：构造Entry对象

注解@3：将Entry放入所在规格的队列Queue中。



小结：还有allocate()方法留在下节梳理，就内存数组结构简单做个小结：

@1 Netty以chunk为单位（16M）向系统申请物理内存，Netty池化内存分成了4种内存类型。Tiny（0~512Byte），Small（512Byte~8KB），Normal（8KB~16MB），Huge（>16M）

@2 Netty对Tiny、Small、Normal做了缓存，针对不同的类型通过”数组+队列“继续切成不同的尺寸，每个尺寸内的缓存ByteBuffer大小相同，不同尺寸之间缓存的Buffer大小以2的N次增长。

@3 Tiny类型从0到496被划分为32个尺寸（数组）

@4 Small类型从512到4096（4K）被划分4个尺寸

@5 Normal类型从8192（8K）到32768（32K）被划分为3个尺寸

@6 在内存分配时，先根据需要分配的内存大小判断属于那种内存类型；进而计算出属于该内存类型的哪个尺寸。

@7 每个尺寸都维护有队列Queue，定位到尺寸规格也就拿到Queue中的实际缓存（PoolChunk）和指针（handle）并完成所需分配内存buffer的初始化。













