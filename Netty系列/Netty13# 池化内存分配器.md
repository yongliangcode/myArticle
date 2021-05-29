---
title: Netty13# 池化内存分配器
categories: Netty
tags: Netty
date: 2021-02-19 11:55:01
---

# 前言 

PooledByteBufAllocator作为池化内存分配的入口，提供了众多的配置参数和便捷方法。这篇主要撸下他们大体都啥含义、干啥用的。为后面池化内存其他组件做铺垫。



# 成员变量说明

下面的成员变量基本都提供了默认值，可以通过参数去自定义，下面表格给出具体说明。

| 成员变量                                 | 说明                                                         |
| ---------------------------------------- | ------------------------------------------------------------ |
| DEFAULT_NUM_HEAP_ARENA                   | PoolAreana（堆内存）个数，默认为核数的2倍，可以由参数-Dio.netty.allocator.numHeapArenas指定 |
| DEFAULT_NUM_DIRECT_ARENA                 | PoolAreana（堆外内存）个数默认为核数的2倍，堆外内存，可以通过-Dio.netty.allocator.numDirectArenas指定 |
| DEFAULT_PAGE_SIZE                        | 默认pageSize=8K，可以通过-Dio.netty.allocator.pageSize，需大于4096且为2的倍数 |
| DEFAULT_MAX_ORDER                        | 二叉树最高层数，取值范围为0~14，默认为11，可以通过-Dio.netty.allocator.maxOrder参数指定 |
| DEFAULT_TINY_CACHE_SIZE                  | 默认tiny类型缓存池大小512，可以通过-Dio.netty.allocator.tinyCacheSize指定 |
| DEFAULT_SMALL_CACHE_SIZE                 | 默认small类型缓存池大小为256，可以通过-Dio.netty.allocator.smallCacheSize指定 |
| DEFAULT_NORMAL_CACHE_SIZE                | 默认normal类型缓存池大小为64，可以通过-Dio.netty.allocator.normalCacheSize指定 |
| DEFAULT_MAX_CACHED_BUFFER_CAPACITY       | 默认为32KB，用于限制normal缓存数组的长度，可以通过-Dio.netty.allocator.maxCachedBufferCapacity指定 |
| DEFAULT_CACHE_TRIM_INTERVAL              | 默认8192，分配次数阈值，超过后释放内存池，可以通过-Dio.netty.allocator.cacheTrimInterval指定 |
| DEFAULT_CACHE_TRIM_INTERVAL_MILLIS       | 默认0不开启，定时释放内存池，可以通过-Dio.netty.allocator.cacheTrimIntervalMillis指定 |
| DEFAULT_USE_CACHE_FOR_ALL_THREADS        | 默认true，使用线程缓存，可以通过-Dio.netty.allocator.useCacheForAllThread制定 |
| DEFAULT_DIRECT_MEMORY_CACHE_ALIGNMENT    | 直接内存的校准对齐参数，分配内存时按位与（&）校准。默认0不校准，可以通过-Dio.netty.allocator.directMemoryCacheAlignment指定 |
| DEFAULT_MAX_CACHED_BYTEBUFFERS_PER_CHUNK | 默认1023，指定PoolChunk缓存ByteBuffer对象的最大数量，可以通过-Dio.netty.allocator.maxCachedByteBuffersPerChunk指定 |
| MIN_PAGE_SIZE                            | 校验用的，PageSize不能小于4KB                                |
| MAX_CHUNK_SIZE                           | 校验用的，Chunk的边界值，(((long) Integer.MAX_VALUE + 1) / 2) |
| heapArenas                               | Arena数组，元素为HeapArena                                   |
| directArenas                             | Arena数组，元素为DirectArena                                 |
| PooledByteBufAllocatorMetric metric      | 暴露统计指标，例如：用了多少堆内存、用了多少堆外直接内存等   |





# 静态块赋值 



**DEFAULT_PAGE_SIZE** 

下面是通过static{}静态块赋值PageSize，默认为8KB，可以通过-Dio.netty.allocator.pageSize自定义。

```java
static {
        int defaultPageSize = SystemPropertyUtil.getInt("io.netty.allocator.pageSize", 8192);
        Throwable pageSizeFallbackCause = null;
        try {
            validateAndCalculatePageShifts(defaultPageSize);
        } catch (Throwable t) {
            pageSizeFallbackCause = t;
            defaultPageSize = 8192;
        }
        DEFAULT_PAGE_SIZE = defaultPageSize;
```

PageSize校验过程，不能小于MIN_PAGE_SIZE（4KB），需要2的倍数。

```java
private static int validateAndCalculatePageShifts(int pageSize) {
        if (pageSize < MIN_PAGE_SIZE) {
            throw new IllegalArgumentException("pageSize: " + pageSize + " (expected: " + MIN_PAGE_SIZE + ")");
        }

        if ((pageSize & pageSize - 1) != 0) {
            throw new IllegalArgumentException("pageSize: " + pageSize + " (expected: power of 2)");
        }

        // Logarithm base 2. At this point we know that pageSize is a power of two.
        return Integer.SIZE - 1 - Integer.numberOfLeadingZeros(pageSize);
    }
```



**DEFAULT_MAX_ORDER** 

maxOrder默认大小为11，可以通过-Dio.netty.allocator.maxOrder自定义。

```java
pageSizeint defaultMaxOrder = SystemPropertyUtil.getInt("io.netty.allocator.maxOrder", 11);
Throwable maxOrderFallbackCause = null;
try {
  validateAndCalculateChunkSize(DEFAULT_PAGE_SIZE, defaultMaxOrder);
} catch (Throwable t) {
  maxOrderFallbackCause = t;
  defaultMaxOrder = 11;
}
DEFAULT_MAX_ORDER = defaultMaxOrder;
```

maxOrder校验过程，最大值不能超过14，同时计算了ChunkSize <<=1 ，即：8192<<=1 为 16777216（16M），也就是默认ChunkSize大小为16M。

```java
private static int validateAndCalculateChunkSize(int pageSize, int maxOrder) {
  if (maxOrder > 14) {
    throw new IllegalArgumentException("maxOrder: " + maxOrder + " (expected: 0-14)");
  }
  int chunkSize = pageSize;
  for (int i = maxOrder; i > 0; i --) {
    if (chunkSize > MAX_CHUNK_SIZE / 2) {
      throw new IllegalArgumentException(String.format(
        "pageSize (%d) << maxOrder (%d) must not exceed %d", pageSize, maxOrder, MAX_CHUNK_SIZE));
    }
    chunkSize <<= 1;
  }
  return chunkSize;
}
```



**DEFAULT_NUM_HEAP_ARENA和DEFAULT_NUM_DIRECT_ARENA**

```java
final Runtime runtime = Runtime.getRuntime();
final int defaultMinNumArena = NettyRuntime.availableProcessors() * 2;
final int defaultChunkSize = DEFAULT_PAGE_SIZE << DEFAULT_MAX_ORDER;
DEFAULT_NUM_HEAP_ARENA = Math.max(0,
                SystemPropertyUtil.getInt(
                        "io.netty.allocator.numHeapArenas",
                        (int) Math.min(
                                defaultMinNumArena,
                                runtime.maxMemory() / defaultChunkSize / 2 / 3)));


 DEFAULT_NUM_DIRECT_ARENA = Math.max(0,
                SystemPropertyUtil.getInt(
                        "io.netty.allocator.numDirectArenas",
                        (int) Math.min(
                                defaultMinNumArena,
                                PlatformDependent.maxDirectMemory() / defaultChunkSize / 2 / 3)));
```

**解读：** DEFAULT_NUM_HEAP_ARENA与DEFAULT_NUM_DIRECT_ARENA赋值结构相同，默认值也相同。DEFAULT_NUM_HEAP_ARENA通过-Dio.netty.allocator.numHeapArenas自定义；DEFAULT_NUM_DIRECT_ARENA通过-Dio.netty.allocator.numDirectArenas自定义。

| 参数                                           | 含义                                                         |
| ---------------------------------------------- | ------------------------------------------------------------ |
| defaultMinNumArena                             | 默认为CPU核数的2倍                                           |
| defaultChunkSize                               | DEFAULT_PAGE_SIZE << DEFAULT_MAX_ORDER 也就是 8192 << 11 = 16777216（16M） |
| runtime.maxMemory()                            | Jvm从操作系统获取的最大内存由参数-Xmx指定                    |
| runtime.maxMemory() / defaultChunkSize / 2 / 3 | defaultChunkSize=16M，可以简化为 runtime.maxMemory()/96M。也就是Jvm最大内存/96M |

所以默认DEFAULT_NUM_HEAP_ARENA=DEFAULT_NUM_DIRECT_ARENA，CPU核数2倍与runtime.maxMemory()/96M取最小值，在高配的环境下，通常为核数的两倍。



**其他** 

下面这几个，都是可以通过参数指定，没啥特别逻辑，具体含义见上面成员变量。

```java
DEFAULT_TINY_CACHE_SIZE = SystemPropertyUtil.getInt("io.netty.allocator.tinyCacheSize", 512);
DEFAULT_SMALL_CACHE_SIZE = SystemPropertyUtil.getInt("io.netty.allocator.smallCacheSize", 256);
DEFAULT_NORMAL_CACHE_SIZE = SystemPropertyUtil.getInt("io.netty.allocator.normalCacheSize", 64);
DEFAULT_MAX_CACHED_BUFFER_CAPACITY = SystemPropertyUtil.getInt(
                "io.netty.allocator.maxCachedBufferCapacity", 32 * 1024);
DEFAULT_CACHE_TRIM_INTERVAL = SystemPropertyUtil.getInt(
                "io.netty.allocator.cacheTrimInterval", 8192);
DEFAULT_CACHE_TRIM_INTERVAL_MILLIS = SystemPropertyUtil.getLong(
                        "io.netty.allocator.cacheTrimIntervalMillis", 0);
DEFAULT_USE_CACHE_FOR_ALL_THREADS = SystemPropertyUtil.getBoolean(
                "io.netty.allocator.useCacheForAllThreads", true);
DEFAULT_DIRECT_MEMORY_CACHE_ALIGNMENT = SystemPropertyUtil.getInt(
                "io.netty.allocator.directMemoryCacheAlignment", 0);
DEFAULT_MAX_CACHED_BYTEBUFFERS_PER_CHUNK = SystemPropertyUtil.getInt(
                "io.netty.allocator.maxCachedByteBuffersPerChunk", 1023);
```



# 构造函数

下面对构造函数的赋值和校验进行走查。

```java
public PooledByteBufAllocator(boolean preferDirect, int nHeapArena, int nDirectArena, int pageSize, int maxOrder,
                                  int tinyCacheSize, int smallCacheSize, int normalCacheSize,
                                  boolean useCacheForAllThreads, int directMemoryCacheAlignment) {
        // 指定是否使用直接内存
        super(preferDirect);
        // 创建PoolThreadLocalCache，useCacheForAllThreads是否允许使用对象池
        threadCache = new PoolThreadLocalCache(useCacheForAllThreads);
        // tiny类型缓存大小
        this.tinyCacheSize = tinyCacheSize;
        // small类型缓存大小
        this.smallCacheSize = smallCacheSize;
        // normal类型缓存大小
        this.normalCacheSize = normalCacheSize;
        // chunk大小默认16M
        chunkSize = validateAndCalculateChunkSize(pageSize, maxOrder);
				// 需正数
        checkPositiveOrZero(nHeapArena, "nHeapArena");
        checkPositiveOrZero(nDirectArena, "nDirectArena");

        checkPositiveOrZero(directMemoryCacheAlignment, "directMemoryCacheAlignment");
        if (directMemoryCacheAlignment > 0 && !isDirectMemoryCacheAlignmentSupported()) {
            throw new IllegalArgumentException("directMemoryCacheAlignment is not supported");
        }

        if ((directMemoryCacheAlignment & -directMemoryCacheAlignment) != directMemoryCacheAlignment) {
            throw new IllegalArgumentException("directMemoryCacheAlignment: "
                    + directMemoryCacheAlignment + " (expected: power of two)");
        }
        // 页偏移默认为13，pageShift = Integer.SIZE - 1 - Integer.numberOfLeadingZeros(8192) = 32 - 1 - 18 = 13
        int pageShifts = validateAndCalculatePageShifts(pageSize);

        if (nHeapArena > 0) {
            // 堆内存，创建PoolArena[]数组
            heapArenas = newArenaArray(nHeapArena);
            List<PoolArenaMetric> metrics = new ArrayList<PoolArenaMetric>(heapArenas.length);
            // 给数组元素赋值HeapArena
            for (int i = 0; i < heapArenas.length; i ++) {
                PoolArena.HeapArena arena = new PoolArena.HeapArena(this,
                        pageSize, maxOrder, pageShifts, chunkSize,
                        directMemoryCacheAlignment);
                heapArenas[i] = arena;
                metrics.add(arena);
            }
            heapArenaMetrics = Collections.unmodifiableList(metrics);
        } else {
            heapArenas = null;
            heapArenaMetrics = Collections.emptyList();
        }

        if (nDirectArena > 0) {
            // 堆外直接内存，创建PoolArena[]数组
            directArenas = newArenaArray(nDirectArena);
            List<PoolArenaMetric> metrics = new ArrayList<PoolArenaMetric>(directArenas.length);
            // 直接内存数组赋值DirectArena
            for (int i = 0; i < directArenas.length; i ++) {
                PoolArena.DirectArena arena = new PoolArena.DirectArena(
                        this, pageSize, maxOrder, pageShifts, chunkSize, directMemoryCacheAlignment);
                directArenas[i] = arena;
                metrics.add(arena);
            }
            directArenaMetrics = Collections.unmodifiableList(metrics);
        } else {
            directArenas = null;
            directArenaMetrics = Collections.emptyList();
        }
        metric = new PooledByteBufAllocatorMetric(this);
    }
```



# 重要方法走查

比较重要的方法主要是newHeapBuffer&newDirectBuffer，这两个的流程和代码结构一致，大体流程见上一篇文末有梳理。

另外，metric这个方法可以观察到内存相关情况。

```java
public PooledByteBufAllocatorMetric metric() {
        return metric;
}
```

**usedHeapMemory** ，查看Netty内存池分配堆空间大小。

```java
final long usedHeapMemory() {
  return usedMemory(heapArenas);
}
```

**usedDirectMemory**, 查看Netty内存池分配堆外直接空间大小。

```java
final long usedDirectMemory() {
	return usedMemory(directArenas);
}
```

通过usedMemory累加PoolArena的内存分配。

```java
 private static long usedMemory(PoolArena<?>[] arenas) {
   if (arenas == null) {
     return -1;
   }
   long used = 0;
   for (PoolArena<?> arena : arenas) {
     used += arena.numActiveBytes();
     if (used < 0) {
       return Long.MAX_VALUE;
     }
   }
   return used;
 }
```









