---
title: Netty9# Netty抽象内存分配器实现原理
categories: Netty
tags: Netty
date: 2021-01-16 11:55:01

---



# 前言

 本文通过分析抽象内存分配器API梳理其基于堆内存、堆外内存分配的实现原理。最后走查了CompositeByteBuf这种类似数据库视图的实现原理。



# 内存分配器概览

**堆外内存&堆内存** 

| 分配方式 | 优点                                              | 缺点                                                         |
| -------- | ------------------------------------------------- | ------------------------------------------------------------ |
| 堆内存   | JVM负责内存的分配与回收                           | 数据过多会引起频繁GC和停顿；<br />多一次拷贝，在用户态分配、I/O通信需要数据拷贝到内核态 |
| 堆外内存 | I/O性能高，直接在内核态分配<br />降低GC频率和停顿 | 内存分配和收回比较慢、需要手动处理                           |



**内存分配器类图** 

字节缓存的分配出自ByteBufAllocator，其实现类AbstractByteBufAllocator（抽象类）、PooledByteBufAllocator（池化内存分配器）、UnpooledByteBufAllocator（非池化内存分配器）、PreferHeapByteBufAllocator（堆内存分配器）、PreferredDirectByteBufAllocator（堆外内存分配器）。





![](https://gitee.com/laoliangcode/md-picture/raw/master/img/%E5%86%85%E5%AD%98%E5%88%86%E9%85%8D%E5%99%A8%E7%B1%BB%E5%9B%BE%20(1).png)





**主要接口** 

| 接口                               | 说明                                                         |
| ---------------------------------- | ------------------------------------------------------------ |
| ByteBuf buffer()                   | 分配一块字节缓存，由其实现类决定堆外内存或者堆内存           |
| ByteBuf ioBuffer()                 | 系统支持UNSAFE和CLEANER则优先分配堆外内存；否则分配堆内存。  |
| ByteBuf heapBuffer()               | 分配堆内存字节缓存区                                         |
| ByteBuf directBuffer()             | 分配堆外内存字节缓存区                                       |
| CompositeByteBuf compositeBuffer() | 分配一个CompositeByteBuf（将多个buffers组合成一个buffer）<br />由实现类决定堆内存或者堆外内存 |



# 内存分配器API解读

下面走查下抽象内存分配器AbstractByteBufAllocator的API。

**构造函数**

```java
private final boolean directByDefault;
private final ByteBuf emptyBuf;
protected AbstractByteBufAllocator(boolean preferDirect) {
  directByDefault = preferDirect && PlatformDependent.hasUnsafe(); // 注解@1
  emptyBuf = new EmptyByteBuf(this);
}
```

**注解@1** directByDefault是否使用堆外内存分配，满足两个条件。preferDirect布尔型用户传入；PlatformDependent.hasUnsafe() 系统是否支持UNSAFE（通过内存指针进行堆外内存分配）；即：用户传入preferDirect=true并且系统支持UNSAFE则使用堆外内存。



**buffer()**

```java
@Override
public ByteBuf buffer() { // 注解@2
  if (directByDefault) {
    return directBuffer();
  }
  return heapBuffer();
}
```

**注解@2** directByDefault如果为true使用堆外内存分配DirectByteBuffer，底层使用unsafe.allocateMemory分配。

directByDefault如果为false使用堆内存分配 new byte[initialCapacity]。



**ioBuffer** 

```java
public ByteBuf ioBuffer() { // 注解@3
  if (PlatformDependent.hasUnsafe() || isDirectBufferPooled()) {
  return directBuffer(DEFAULT_INITIAL_CAPACITY);
  }
  return heapBuffer(DEFAULT_INITIAL_CAPACITY);
}
```

**注解@3** 如果系统支持UNSAFE或者使用池化内存，优先分配堆外内存，否则分配堆内存。



**heapBuffer**

```java
@Override
public ByteBuf heapBuffer() { // 注解@4
	return heapBuffer(DEFAULT_INITIAL_CAPACITY, DEFAULT_MAX_CAPACITY);
}
```

**注解@4** 分配堆外内存new一个byte数组（new byte[]）。



**directBuffer**

```java
@Override
public ByteBuf directBuffer() { // 注解@5
	return directBuffer(DEFAULT_INITIAL_CAPACITY, DEFAULT_MAX_CAPACITY);
}
```

**注解@5** 分配堆外内存。



**compositeBuffer**

```java
@Override
public CompositeByteBuf compositeBuffer() { // 注解@6
  if (directByDefault) {
  	return compositeDirectBuffer();
  }
	return compositeHeapBuffer();
}
```

**注解@6** 跟上面一样的，只是分配的CompositeByteBuf。下面看下这种将多个个buffer组合成一个buffer是如何实现的。

```java
@Override
public CompositeByteBuf compositeHeapBuffer() {
	return compositeHeapBuffer(DEFAULT_MAX_COMPONENTS);
}
@Override
public CompositeByteBuf compositeHeapBuffer(int maxNumComponents) {
	return toLeakAwareBuffer(new CompositeByteBuf(this, false, maxNumComponents));
}
private CompositeByteBuf(ByteBufAllocator alloc, boolean direct, int maxNumComponents, int initSize) {
  super(AbstractByteBufAllocator.DEFAULT_MAX_CAPACITY);

  this.alloc = ObjectUtil.checkNotNull(alloc, "alloc");
  if (maxNumComponents < 1) {
  throw new IllegalArgumentException(
  "maxNumComponents: " + maxNumComponents + " (expected: >= 1)");
  }

  this.direct = direct;
  this.maxNumComponents = maxNumComponents;
  components = newCompArray(initSize, maxNumComponents); // 注解@6.1
}
```

**注解@6.1** 在CompositeByteBuf的构造方法中初始化了一个components，这个默认initSize=0；maxNumComponents默认为16。

```java
private static Component[] newCompArray(int initComponents, int maxNumComponents) {
	int capacityGuess = Math.min(AbstractByteBufAllocator.DEFAULT_MAX_COMPONENTS, maxNumComponents);
	return new Component[Math.max(initComponents, capacityGuess)]; // 注解@6.2
}
```

**注解@6.2** components是Component的对象数组，数组大小默认为16. 



# CompositeByteBuf实现原理

下面通过例子来体验一把CompositeByteBuf，先直观感受下。

**示例代码** 

```java
@Test
public void testCompositeByteBuf(){
  String str1 = "瓜农";
  String str2 = "老梁";
  ByteBuf buf1 = Unpooled.buffer(10);
  buf1.writeBytes(str1.getBytes(CharsetUtil.UTF_8));
  System.out.println("buf1's readerIndex:" + buf1.readerIndex());
  System.out.println("buf1's writeIndex" + buf1.writerIndex());

  ByteBuf buf2 = Unpooled.buffer(10);
  buf2.writeBytes(str2.getBytes(CharsetUtil.UTF_8));
  System.out.println("buf2's readerIndex:" + buf2.readerIndex());
  System.out.println("buf2's writeIndex" + buf2.writerIndex());

  ByteBuf compositeByteBuf = Unpooled.wrappedBuffer(buf1,buf2);
  System.out.println("compositeByteBuf's readerIndex:" + compositeByteBuf.readerIndex());
  System.out.println("compositeByteBuf's writeIndex" + compositeByteBuf.writerIndex());
  System.out.print(compositeByteBuf.toString(CharsetUtil.UTF_8));
}
```

**结果输出**

```
buf1's readerIndex:0
buf1's writeIndex6
buf2's readerIndex:0
buf2's writeIndex6
compositeByteBuf's readerIndex:0
compositeByteBuf's writeIndex12
瓜农老梁
```

**小结**  Unpooled.wrappedBuffer(buf1,buf2)将两个ByteBuf进行了合并一个ByteBuf；对外提供统一的读写指针供使用。



接下来看下他是如何合并的。

```java
public static ByteBuf wrappedBuffer(int maxNumComponents, ByteBuf... buffers) {
        switch (buffers.length) {
        case 0:
            break;
        case 1:
            ByteBuf buffer = buffers[0];
            if (buffer.isReadable()) {
                return wrappedBuffer(buffer.order(BIG_ENDIAN));
            } else {
                buffer.release();
            }
            break;
        default:
            for (int i = 0; i < buffers.length; i++) {
                ByteBuf buf = buffers[i];
                if (buf.isReadable()) {
                    return new CompositeByteBuf(ALLOC, false, maxNumComponents, buffers, i); // 注解@7
                }
                buf.release();
            }
            break;
        }
        return EMPTY_BUFFER;
    }
```

**注解@7** 通过创建一个CompositeByteBuf，将ByteBuf数组传入构造函数。

```java
CompositeByteBuf(ByteBufAllocator alloc, boolean direct, int maxNumComponents,
            ByteBuf[] buffers, int offset) {
            this(alloc, direct, maxNumComponents, buffers.length - offset);

	addComponents0(false, 0, buffers, offset); // 注解@8
	consolidateIfNeeded();
	setIndex0(0, capacity()); // 注解@9
}
```

**注解@8**  填充Component[]数据，每个Component元素包含了传入的ByteBuf。

```java
private CompositeByteBuf addComponents0(boolean increaseWriterIndex,
            final int cIndex, ByteBuf[] buffers, int arrOffset) {
  			//  buffers数组的长度；本例中arrOffset=0；count=len
        final int len = buffers.length, count = len - arrOffset;
        int ci = Integer.MAX_VALUE;
        try {
            checkComponentIndex(cIndex); // 合法性校验
            shiftComps(cIndex, count); // 注解@8.1
            int nextOffset = cIndex > 0 ? components[cIndex - 1].endOffset : 0;
            for (ci = cIndex; arrOffset < len; arrOffset++, ci++) { // 注解@8.2
                // 从数组中拿出传入的ByteBuf
              	ByteBuf b = buffers[arrOffset];
                if (b == null) {
                    break;
                }
                // 构建Component
                Component c = newComponent(ensureAccessible(b), nextOffset); 
              	// 加入components数组
                components[ci] = c;
                // 递增endOffset
                nextOffset = c.endOffset;
            }
            return this;
        } finally {
            // ...
        }
}
```

**注解@8.1** 扩容Component数组，默认的数量为16个，当添加的buffer的数量超过16时就需要扩容了，下面看下其如何扩容的。

```java
private void shiftComps(int i, int count) {
        final int size = componentCount, newSize = size + count;
        assert i >= 0 && i <= size && count > 0;
        if (newSize > components.length) {
            // grow the array 
          	int newArrSize = Math.max(size + (size >> 1), newSize); // 注解@8.1.1
            Component[] newArr;
            // 注解@8.1.2
            if (i == size) {
                newArr = Arrays.copyOf(components, newArrSize, Component[].class);
            } else {
                newArr = new Component[newArrSize];
                if (i > 0) {
                    System.arraycopy(components, 0, newArr, 0, i);
                }
                if (i < size) {
                    System.arraycopy(components, i, newArr, i + count, size - i);
                }
            }
            components = newArr;
        } else if (i < size) {
            System.arraycopy(components, i, components, i + count, size - i);
        }
        componentCount = newSize;
    }
```

**注解@8.1.1 ** 默认size=16，size >> 1 = 8 也就是扩容会以原大小一半的容量进行扩容。

**注解@8.1.2** 下面判断根据场景通过Arrays.copyOf、System.arraycopy将Component数组扩容；插入尾部、中部、头部等情况。本示例中没有超过16，所以不会扩容，componentCount=2。

**注解@8.2** 循环拿出传入的ByteBuf数组构建Component，并将其加入Component数组中；最后移动nextOffset。关于各个参数的含义，源码给出了注释。构造函数中

第一个参数：传入的ByteBuf
第二个参数：源ByteBuf的readerIndex
第三个参数：unwrapped的buffer
第四个参数：unwrappedIndex
第五个参数：offset = components[cIndex - 1].endOffset
第六个参数：len = buf.readableBytes()  buf为源buffer
第七个参数：slice = null （示例）

以endOffset为例，等于插入数组中上一个Conponent的endOffset + 当前ByteBuf的可读长度，从而维护了其在整个CompositeByteBuf的写索引情况。

一个buffer对应一个Component，每个Component持有源buffer并维护了其在整个CompositeByteBuf的索引情况。

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210117101614.png)



**注解@9** 设置整个CompositeByteBuf的读索引和写索引，读索引初始值为0；写索引为components[size - 1].endOffset，也就是整个Conponent数组中其每个元素维护的ByteBuf可读字节（writerIndex - readerIndex）大小的总和。

```java
final void setIndex0(int readerIndex, int writerIndex) {
	this.readerIndex = readerIndex;
	this.writerIndex = writerIndex;
}
public int capacity() {
  int size = componentCount;
  return size > 0 ? components[size - 1].endOffset : 0;
}
```



**小结：** CompositeByteBuf通过将多个ByteBuf装入component数组中，对其统一维护读写索引，在外面看起来是一个统一的buffer；类似数据库中的视图。

