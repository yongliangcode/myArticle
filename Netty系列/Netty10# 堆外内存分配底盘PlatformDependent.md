---
title: Netty10# 堆外内存底盘PlatformDependent
categories: Netty
tags: Netty
date: 2021-01-30 22:47:01
---



# 前言

非池化/池化内存如何分配的？该撸这块了，奈何到处都在调用PlatformDependent类的方法，要不各种判断，要不分配堆外内存。反正到处都能看到它，得，索性先把这个撸一把。PlatformDependent又依赖了PlatformDependent0，那就一层一层剥好了。



嗯，有点碎，大伙随便看看。



# PlatformDependent0



### 成员变量

| 名称                                      | 含义                                                         |
| ----------------------------------------- | :----------------------------------------------------------- |
| ADDRESS_FIELD_OFFSET                      | Buffer#address字段的内存偏移地址                             |
| BYTE_ARRAY_BASE_OFFSET                    | 获取内存中第一个元素的内存偏移量                             |
| DIRECT_BUFFER_CONSTRUCTOR                 | DirectByteBuffer构造器对象，用于反射实例化                   |
| EXPLICIT_NO_UNSAFE_CAUSE                  | 平台不支持UNSAFE时，不支持异常封装在EXPLICIT_NO_UNSAFE_CAUSE中 |
| ALLOCATE_ARRAY_METHOD                     | Unsafe#allocateUninitializedArray Method对象，用于反射调用   |
| JAVA_VERSION                              | 获取Java版本                                                 |
| IS_ANDROID                                | 系统是否为Android                                            |
| UNSAFE_UNAVAILABILITY_CAUSE               | 系统不支持UNSAFE会将异常封装在此变量中                       |
| INTERNAL_UNSAFE                           | Java9中Unsafe实例对象                                        |
| IS_EXPLICIT_TRY_REFLECTION_SET_ACCESSIBLE | 启用反射访问，在Java9版本之前是禁止的；Java9以及之后版本需要开启（这点要注意，Netty到处都有对Java9之后的兼容判断） |
| UNSAFE                                    | Unsafe实例对象，操作堆外内存                                 |
| UNSAFE_COPY_THRESHOLD                     | Java8以及以下版本内存拷贝时的最大阈值，最大为1M（1024L * 1024L） |
| UNALIGNED                                 | java.nio.Bits#unaligned方法在当前系统是否可用                |



### 赋值流程

这部分主要针对当前运行的系统平台是否支持UNSAFE等众多涉及到堆外内存分配的属性是否支持，以及赋值给成员变量，在static静态块中判断。



![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210131171437.png)



**@1 检查平台是否支持Unsafe，不支持将异常错误封装在UNSAFE_UNAVAILABILITY_CAUSE中**

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210131102931.png)

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210131103148.png)



**@2 如果系统支持Unsafe，检查Unsafe类中是否包含copyMemory方法，不支持禁用Unsaf**

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210131103401.png)



**@3 检查Buffer类的内存地址address功能，不支持禁用Unsafe**

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210131103802.png)

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210131172146.png)



**@4 检查Unsafe类中的arrayIndexScale方法是否支持，不支持禁用Unsafe**

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210131104315.png)



**@5 检查是否支持反射获取DirectBuffer的构造器，方便堆外内存分配**

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210131110146.png)

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210131110242.png)



**@6 检查系统是否支持java.nio.Bits#unaligned方法**

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210131172239.png)

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210131172305.png)



**@7 Java9以及以上版本检查jdk.internal.misc.Unsafe以及其方法allocateUninitializedArray，并赋值给INTERNAL_UNSAFE和ALLOCATE_ARRAY_METHOD**

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210131111631.png)

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210131172450.png)

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210131111747.png)



### 重要方法走查

PlatformDependent0提供的方法，主要判断Unsafe是否可用、Unsafe分配堆外内存、Unsafe从堆外内存获取数据等。下面挑几个走查下。

**@1 检查UNSAFE是否可用**

```java
static boolean hasUnsafe() {
	return UNSAFE != null;
}
```

**@2 DirectByteBuffer的构造函数是否可用**

```java
static boolean hasDirectBufferNoCleanerConstructor() {
  return DIRECT_BUFFER_CONSTRUCTOR != null;
}
```

**@3 堆外内存分配**

```java
/**
*  malloc()返回获得内存空间的首地址，失败返回null
*  根据返回的内存地址构造DirectBuffer
* @param capacity
* @return
*/
static ByteBuffer allocateDirectNoCleaner(int capacity) {
  /**
  * malloc()返回获得内存空间的首地址，失败返回null
  */
	return newDirectBuffer(UNSAFE.allocateMemory(Math.max(1, capacity)), capacity);
}
```

**@4 调用DirectByteBuffer构造函数分配堆外内存**

```java
static ByteBuffer newDirectBuffer(long address, int capacity) {
        ObjectUtil.checkPositiveOrZero(capacity, "capacity");

        try {
            return (ByteBuffer) DIRECT_BUFFER_CONSTRUCTOR.newInstance(address, capacity);
        } catch (Throwable cause) {
            // Not expected to ever throw!
            if (cause instanceof Error) {
                throw (Error) cause;
            }
            throw new Error(cause);
        }
    }
```

**@5 获取堆外内存数据**

```java
static byte getByte(long address) {
	return UNSAFE.getByte(address);
}
```



# PlatformDependent



### 成员变量

| 名称                                     | 含义                                                         |
| ---------------------------------------- | ------------------------------------------------------------ |
| MAX_DIRECT_MEMORY_SIZE_ARG_PATTERN       | 允许最大堆外内存的正则表达式，可以通过MaxDirectMemorySize参数在应用启动时指定 |
| IS_WINDOWS                               | 判断系统是否为windows系统                                    |
| IS_OSX                                   | 判断是否为MacOS系统                                          |
| IS_J9_JVM                                | 是否为jdk9版本                                               |
| IS_IVKVM_DOT_NET                         | 判断是否为IKVM.NET                                           |
| MAYBE_SUPER_USER                         | 判断是否为root超级用户                                       |
| CAN_ENABLE_TCP_NODELAY_BY_DEFAULT        | Linux系统可以开启TCP_NODELAY参数，当开启时数据会以最快的速度发出去同时也就禁用了纳格算法（Nagle algorithm）；如果不设置（开启纳格算法），数据会缓存满足阈值后发出 |
| UNSAFE_UNAVAILABILITY_CAUSE              | 平台不支持UNSAFE会将异常封装在UNSAFE_UNAVAILABILITY_CAUSE中  |
| DIRECT_BUFFER_PREFERRED                  | 默认优先使用堆外内存分配                                     |
| MAX_DIRECT_MEMORY                        | 获取最大堆外内存通过sun.misc.VM#maxDirectMemory方法获取      |
| MPSC_CHUNK_SIZE                          | 使用JCTools提供的无锁队列，初始化队列容量，默认1024          |
| MIN_MAX_MPSC_CAPACITY                    | 使用JCTools提供的无锁队列，最大队列容量；MIN_MAX_MPSC_CAPACITY =  MPSC_CHUNK_SIZE * 2 |
| MAX_ALLOWED_MPSC_CAPACITY                | 使用JCTools提供的无锁队列，允许队列最大容量，默认为1073741824（1 << 30）；MAX_ALLOWED_MPSC_CAPACITY = Pow2.MAX_POW2 |
| BYTE_ARRAY_BASE_OFFSET                   | 获取内存中第一个元素的内存偏移量；UNSAFE.arrayBaseOffset     |
| TMPDIR                                   | Netty临时目录                                                |
| BIT_MODE                                 | 操作系统是32位还是64位                                       |
| NORMALIZED_ARCH                          | 获取操作系统CPU architecture，例如：x86_64                   |
| NORMALIZED_OS                            | 获取操作系统名称，例如：Linux                                |
| ALLOWED_LINUX_OS_CLASSIFIERS             | Linux系统演化了众多版本，下面是允许的的Linux版本             |
| LINUX_OS_CLASSIFIERS                     | 通过系统识别文件将支持的Linux版本填充到该集合中，是ALLOWED_LINUX_OS_CLASSIFIERS子集 |
| ADDRESS_SIZE                             | 返回系统指针的大小，32位系统返回4；64位系统返回8             |
| USE_DIRECT_BUFFER_NO_CLEANER             | 堆外内存是否能分配（系统是否支持unsafe、DirectByteBuffer是否可用） |
| DIRECT_MEMORY_COUNTER                    | 堆外内存使用限制                                             |
| ThreadLocalRandomProvider                | 随机数字生成器                                               |
| CLEANER                                  | 可以用于堆外内存回收                                         |
| UNINITIALIZED_ARRAY_ALLOCATION_THRESHOLD | 堆外内存分配阈值，默认为1024（字节）；小于该阈值分配堆内存，大于分配堆外内存；可以通过-Dio.netty.uninitializedArrayAllocationThreshold指定 |
| OS_RELEASE_FILES                         | 包含了操作系统识别数据 /etc/os-release与/usr/lib/os-release；String[] OS_RELEASE_FILES = {"/etc/os-release", "/usr/lib/os-release"} |
| BIG_ENDIAN_NATIVE_ORDER                  | 是否为大端排序（默认大端排序）；boolean BIG_ENDIAN_NATIVE_ORDER = ByteOrder.nativeOrder() == ByteOrder.BIG_ENDIAN |



### 重要方法走查

挑几个比较重要的方法走查下，



**@1 内存分配**

小于阈值默认1024（字节），使用堆内存；大于阈值使用堆外内存

```java
 public static byte[] allocateUninitializedArray(int size) {
        return UNINITIALIZED_ARRAY_ALLOCATION_THRESHOLD < 0 || UNINITIALIZED_ARRAY_ALLOCATION_THRESHOLD > size ?
                new byte[size] : PlatformDependent0.allocateUninitializedArray(size);
}
```



**@2释放堆外内存**

```java
public static void freeDirectBuffer(ByteBuffer buffer) {
        CLEANER.freeDirectBuffer(buffer);
}
```



**@3 构造Queue**

使用了使用JCTools提供的无锁队列

```java
public static <T> Queue<T> newMpscQueue() {
        return Mpsc.newMpscQueue();
}
static <T> Queue<T> newMpscQueue() {
  return USE_MPSC_CHUNKED_ARRAY_QUEUE ? new MpscUnboundedArrayQueue<T>(MPSC_CHUNK_SIZE)
    : new MpscUnboundedAtomicArrayQueue<T>(MPSC_CHUNK_SIZE);
}
```



其他的方法基本在判断成员变量或者调用PlatformDependent0的方法分配堆外内存、获取堆外内存数据、释放堆外内存等。



# 小结

PlatformDependent与PlatformDependent0主要针对操作系统、JDK版本等环境因素是否支持堆外内存Unsafe以及一些关联类进行判断；通过封装Unsafe申请堆外内存、释放、获取数据等操作。另外Netty为了提高性能使用了JCTools提供的无锁队列、可以通过-XX:MaxDirectMemorySize参数调整Netty允许使用的最大堆外内存，超过最大限制将使堆外内存分配失败，抛出 “failed to allocate byte(s) of direct memory” 异常。





