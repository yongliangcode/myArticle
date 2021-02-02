---
'
title: Netty10 Netty
categories: Netty
tags: Netty
date: 2021-01-16 11:55:01


---



# 非池化内存管理

下面从示例代码Unpooled.buffer(1)跟踪看下非池化内存分配器UnpooledByteBufAllocator分配，先看其构造函数情况。

```java
@Test
public void testWriteUtf81() {
        String str1 = "瓜农";
        ByteBuf buf = Unpooled.buffer(1); // 调用链@1 入口
        buf.writeBytes(str1.getBytes(CharsetUtil.UTF_8));
        ByteBuf readByteBuf = ByteBufUtil.readBytes(UnpooledByteBufAllocator.DEFAULT,buf,str1.getBytes(CharsetUtil.UTF_8).length);
        System.out.print(readByteBuf.toString(CharsetUtil.UTF_8));
}
```

```java
// 调用链@2
public static ByteBuf buffer(int initialCapacity) {
        return ALLOC.heapBuffer(initialCapacity);
}
// 调用链@3 成员变量ALLOC
private static final ByteBufAllocator ALLOC = UnpooledByteBufAllocator.DEFAULT; // 注解@1

// 调用链@4
public static final UnpooledByteBufAllocator DEFAULT = 
            new UnpooledByteBufAllocator(PlatformDependent.directBufferPreferred()); 

// 调用链@5 PlatformDependent.directBufferPreferred
public static boolean directBufferPreferred() {
        return DIRECT_BUFFER_PREFERRED;
}

// 调用链@6 PlatformDependent中DIRECT_BUFFER_PREFERRED成员变量赋值
DIRECT_BUFFER_PREFERRED = CLEANER != NOOP
                                  && !SystemPropertyUtil.getBoolean("io.netty.noPreferDirect", false); // 注解@2
// 调用链@7
public UnpooledByteBufAllocator(boolean preferDirect) {
      this(preferDirect, false);
}
// 调用链@8
public UnpooledByteBufAllocator(boolean preferDirect, boolean disableLeakDetector, boolean tryNoCleaner) {
  super(preferDirect);
  this.disableLeakDetector = disableLeakDetector;
  noCleaner = tryNoCleaner && PlatformDependent.hasUnsafe()
    && PlatformDependent.hasDirectBufferNoCleanerConstructor();
}
// 调用链@9
protected AbstractByteBufAllocator(boolean preferDirect) {
  directByDefault = preferDirect && PlatformDependent.hasUnsafe();
  emptyBuf = new EmptyByteBuf(this);
}
```

**注解@1**  UnpooledByteBufAllocator提供了一个默认的实例DEFAULT，传入preferDirect的参数。preferDirect=true则优先使用堆外内存DirectByteBuffer分配内存。preferDirect=false则分配堆内内存。preferDirect的取值取决于DIRECT_BUFFER_PREFERRED。



**注解@2** DIRECT_BUFFER_PREFERRED判断条件是CLEANER != NOOP，CLEANER是什么？用于跟踪DirectByteBuffer对象的垃圾回收，当其被回收掉同时清理掉已分配的堆外内存。

下面是CLEANER的赋值，只要系统支持CLEANER堆外内存回收，默认使用分配堆外内存。

```java
if (!isAndroid()) {
  // only direct to method if we are not running on android.
  // See https://github.com/netty/netty/issues/2604
  if (javaVersion() >= 9) {
    CLEANER = CleanerJava9.isSupported() ? new CleanerJava9() : NOOP;
  } else {
    CLEANER = CleanerJava6.isSupported() ? new CleanerJava6() : NOOP;
  }
} else {
  CLEANER = NOOP;
}
```

![image-20210121101857330](/Users/yongliang/Library/Application Support/typora-user-images/image-20210121101857330.png)

**小结：** 系统支持UNSAFE和CLEANER则directByDefault=true，调用AbstractByteBufAllocator#buffer()时默认使用堆外内存。CLEANER用于堆外内存回收。另外可以通过构造函数直接preferDirect设置偏好是否使用堆外内存。













