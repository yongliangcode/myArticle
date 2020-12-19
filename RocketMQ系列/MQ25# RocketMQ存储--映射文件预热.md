---
title: MQ25# RocketMQ存储--映射文件预热
categories: RocketMQ
tags: RocketMQ
date: 2020-12-20 20:42:01
---



# 问题描述

1.为什么创建文件（commitLog）时要预热？

2.为什么要写入1G大小的假值（0）呢？

3.为什么要锁定内存？

4.预热流程是怎么样的？



# 调用链

```
@1 AllocateMappedFileService#mmapOperation

// pre write mappedFile

if (mappedFile.getFileSize() >= this.messageStore.getMessageStoreConfig()

.getMapedFileSizeCommitLog() && this.messageStore.getMessageStoreConfig().isWarmMapedFileEnable()) {

//预热

mappedFile.warmMappedFile(

this.messageStore.getMessageStoreConfig().getFlushDiskType(),

this.messageStore.getMessageStoreConfig()

.getFlushLeastPagesWhenWarmMapedFile()

);

}

@2 MappedFilewarmMappedFile
```



# 流程图



![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20201219133918.png)



<!--more-->



# 代码验证

在文件预热时为什么将1G假值（0）写入文件呢？不写这些值会怎么样呢？

将预热代码改造下做个测试：分别映射空文件和将文件写入1G假值0，观察内存映射变化。

**1.文件映射测试代码**

```
public static void main(String [] args) throws Exception {

File file = new File(args[0]);

FileChannel fileChannel = new RandomAccessFile(file, "rw").getChannel();

MappedByteBuffer mappedByteBuffer = fileChannel.map(FileChannel.MapMode.READ_WRITE, 0, 1024 * 1024 * 1024);

if(args.length >1 && args[1]!=null && Boolean.parseBoolean(args[1])){

for (int i = 0, j = 0; i < 1024 * 1024 * 1024; i += 1024 * 4, j++) {

mappedByteBuffer.put(i, (byte) 0);

}

}

final long address = ((DirectBuffer) (mappedByteBuffer)).address();

Pointer pointer = new Pointer(address);

{

int ret = LibC.INSTANCE.mlock(pointer, new NativeLong(1024 * 1024 * 102));

}

{

int ret = LibC.INSTANCE.madvise(pointer, new NativeLong(1024 * 1024 * 102), LibC.MADV_WILLNEED);

}

Thread.sleep(1000000000);

}
```

**2.映射空文件**

新建文件x.tmp，此文件为空，映射到内存会发生什么呢？

运行例子程序

```
java -jar melontst2.jar x.tmp
```



查看虚拟内存映射

```
cat /proc/xxx-pid/maps

7f4c48000000-7f4c88000000 rw-s 00000000 fd:00 395167 /home/baseuser/x.tmp
```



<u>小结：7f4c88000000-7f4c48000000 = 139966675943424 - 139965602201600</u>

<u>= 1024 * 1024 * 1024 = 1G。即：虽然是空文件，内存映射大小依然是1G大小</u>



**3.映射1G文件**

新建文件y.tmp, 写入大小为1G字节0的数据，映射到内存会发生什么呢？

运行例子程序

```
java -jar melontst2.jar y.tmp true
```



查看虚拟内存映射

```
cat /proc/xxx-pid/maps

7f36e4000000-7f3724000000 rw-s 00000000 fd:00 395900 /home/baseuser/y.tmp
```



<u>小结：内存映射大小计算</u>

<u>7f3724000000-7f36e4000000=139874803908608-139873730166784=1073741824</u>

<u>= 1024 * 1024 * 1024 = 1G</u>

<u>内存分配了1G大小。</u>

**4.思考**

既然空文件和写入1G字节虚拟内存映射都是1G大小，写入1G大小的意义呢？

**使用mmap()内存分配时，只是建立了进程虚拟地址空间，并没有分配虚拟内存对应的物理内存。当进程访问这些没有建立映射关系的虚拟内存时，处理器自动触发一个缺页异常，进而进入内核空间分配物理内存、更新进程缓存表，最后返回用户空间，回复进程运行**。

```
小结：写入这些假值的意义在于实际分配物理内存，在消息写入时防止缺页异常
```

**5.内存映射简图**

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20201219134135.png)

**虚拟内存**

计算机系统内存管理的一种技术。它使得应用程序认为它拥有连续的可用的内存（一个连续完整的地址空间），而实际上，它通常是被分隔成多个物理内存碎片，还有部分暂时存储在外部磁盘存储器上，在需要时进行数据交换

虚拟地址空间的内部又被分为内核空间和用户空间两部分，进程在用户态时，只能访问用户空间内存；只有进入内核态后，才可以访问内核空间内存

**MMU**

MMU是Memory Management Unit的缩写，中文名是内存管理单元，它是中央处理器（CPU）中用来管理虚拟存储器、物理存储器的控制线路，同时也负责虚拟地址映射为物理地址

**页表**

是虚拟内存系统用来存储逻辑地址和物理地址之间映射的数据结构

**内存映射mmap**

将虚拟地址映射到物理地址



# Native API解释

**mmap**

映射文件或设备到内存

void mmap(void *start, size_t length, int prot, int flags, int fd, off_t offset);

The mmap() function asks to map length bytes starting at offset offset from the file (or other object) specified by the file descriptor fd into memory, preferably at address start. This latter address is a hint only, and is usually specified as 0. The actual place where the object is mapped is returned by mmap().

**mlock**

锁定内存

int mlock(const void *addr, size_t len);

mlock() locks pages in the address range starting at addr and continuing for len bytes. All pages that contain a part of the specified address range are guaranteed to be resident in RAM when the call returns successfully; the pages are guaranteed to stay in RAM until later unlocked.

**madvise**

提出建议关于使用内存

int madvise(void *start, size_t length, int advice);

The madvise() system call advises the kernel about how to handle paging input/output in the address range beginning at address start and with size length bytes. It allows an application to tell the kernel how it expects to use some mapped or shared memory areas, so that the kernel can choose appropriate read-ahead and caching techniques. This call does not influence the semantics of the application (except in the case ofMADV_DONTNEED), but may influence its performance. The kernel is free to ignore the advice。

MADV_WILLNEED模式（MappedFile预热使用该模式）

MADV_WILLNEED：Expect access in the near future. (Hence, it might be a good idea to read some pages ahead.)



# 总结

1.Broker配置warmMapedFileEnable为false，开启预热需要设置true。

2.写入1G字节假值0是为了让系统分配物理内存空间，如果没有这些假值，系统不会实际分配物理内存，防止在写入消息时发生缺页异常。

3.mlock锁定内存，防止其被交换到swap空间。

4.madvise建议操作系统如何使用内存，MADV_WILLNEED提前预热，预读一些页面，提高性能。

5.文件预热使得内存提前分配，并锁定在内存中，在写入消息时不必再进行内存分配。



# 附录源代码

**1.文件初始化源代码**

```
private void init(final String fileName, final int fileSize) throws IOException {

this.fileName = fileName;

this.fileSize = fileSize;

this.file = new File(fileName);

this.fileFromOffset = Long.parseLong(this.file.getName());//文件名代表该文件的起始偏移量

boolean ok = false;

ensureDirOK(this.file.getParent());

try {

//通过RandomAccessFile创读写文件通道

this.fileChannel = new RandomAccessFile(this.file, "rw").getChannel();

//将文件内容通过NIO的内存映射Buffer,将文件映射到内存

//将磁盘文件读到内存中，每个文件大小为1G

this.mappedByteBuffer = this.fileChannel.map(MapMode.READ_WRITE, 0, fileSize);

TOTAL_MAPPED_VIRTUAL_MEMORY.addAndGet(fileSize);

TOTAL_MAPPED_FILES.incrementAndGet();

ok = true;

} catch (FileNotFoundException e) {

log.error("create file channel " + this.fileName + " Failed. ", e);

throw e;

} catch (IOException e) {

log.error("map file " + this.fileName + " Failed. ", e);

throw e;

} finally {

if (!ok && this.fileChannel != null) {

this.fileChannel.close();

}

}

}
```



**2.预热源代码**

```
public void warmMappedFile(FlushDiskType type, int pages) {

long beginTime = System.currentTimeMillis();

ByteBuffer byteBuffer = this.mappedByteBuffer.slice();

int flush = 0; //记录上一次刷盘的字节数

long time = System.currentTimeMillis();

for (int i = 0, j = 0; i < this.fileSize; i += MappedFile.OS_PAGE_SIZE, j++) {

byteBuffer.put(i, (byte) 0);

// force flush when flush disk type is sync

//当刷盘策略为同步刷盘时，执行强制刷盘

//每修改pages个分页刷一次盘 内存页的大小为4K

if (type == FlushDiskType.SYNC_FLUSH) {

if ((i / OS_PAGE_SIZE) - (flush / OS_PAGE_SIZE) >= pages) {

flush = i;

mappedByteBuffer.force();

}

}

// prevent gc

//Linux CPU调度策略基于时间片 Thread.sleep 当前线程主动放弃CPU资源，立即进入就绪状态

//防止一直抢占CPU资源不释放

if (j % 1000 == 0) {

log.info("j={}, costTime={}", j, System.currentTimeMillis() - time);

time = System.currentTimeMillis();

try {

Thread.sleep(0);

} catch (InterruptedException e) {

e.printStackTrace();

}

}

}

// force flush when prepare load finished

if (type == FlushDiskType.SYNC_FLUSH) {

log.info("mapped file warm-up done, force to disk, mappedFile={}, costTime={}",

this.getFileName(), System.currentTimeMillis() - beginTime);

mappedByteBuffer.force();

}

log.info("mapped file warm-up done. mappedFile={}, costTime={}", this.getFileName(),

System.currentTimeMillis() - beginTime);

this.mlock();

}
```



**3.内存锁定代码**

```
public void mlock() {

final long beginTime = System.currentTimeMillis();

final long address = ((DirectBuffer) (this.mappedByteBuffer)).address();

Pointer pointer = new Pointer(address);

{

//锁死

int ret = LibC.INSTANCE.mlock(pointer, new NativeLong(this.fileSize));

log.info("mlock {} {} {} ret = {} time consuming = {}", address, this.fileName, this.fileSize, ret, System.currentTimeMillis() - beginTime);

}

{

int ret = LibC.INSTANCE.madvise(pointer, new NativeLong(this.fileSize), LibC.MADV_WILLNEED);

log.info("madvise {} {} {} ret = {} time consuming = {}", address, this.fileName, this.fileSize, ret, System.currentTimeMillis() - beginTime);

}

}
```

