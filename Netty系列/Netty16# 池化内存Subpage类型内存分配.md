---
title: Netty16# 池化内存Subpage类型内存分配
categories: Netty
tags: Netty
date: 2021-04-10 18:25:01
---



# 引言 

前面聊了大于8KB的内存分配，那小于8KB的呢？上一篇的平衡二叉树第十一层的叶子节点最小也是8KB，那比如要分配128B的缓存，直接分给8KB显然是不合适的，Tiny是小于512Byte，Small介于512B~8KB，Tiny和Small统称Subpage，本文就聊聊他们的内存分配情况，这块应该是整个netty最为复杂的部分了。



# 内容提要

下面是以分配128B为例的整体流程架构图，下面大体叙述下其流程。

* 先从平衡二叉树的第11层选一个未分配的叶子节点大小为8KB的一个Page
  备注：本例中为memoryMap[2048]

* 对该Page进行切割，假如要分配128B，整体会切割为64块
  备注：8192/128=64

* 通过long类型二进制64位来标记分割成各个块的分配状态

  备注：0:未分配，1:已分配

* 一个bitmap数组长度为8，每个元素都能对64块内存进行标记

* 建立了二叉树节点与切分块之间的映射关系
  备注：memoryMapIdx ^ maxSubpageAllocs

* 分配后建立二叉树叶子节点与标记位之间的关系，可以指向内存一块区域
  备注：0x4000000000000000L | (long) bitmapIdx << 32 | memoryMapIdx





![](https://gitee.com/laoliangcode/md-picture/raw/master/img/subpages%E6%95%B4%E4%BD%93%E6%9E%B6%E6%9E%84.png)



<!--more-->



# 源码分析

**示例代码**

```java
@Test
public void testAllocateSubpage() {
    ByteBufAllocator allocator = new PooledByteBufAllocator();
    allocator.directBuffer(128); 
}
```

备注：以分配128B的内存为例，分析其分配过程。



**源码分析**

```java
private long allocateSubpage(int normCapacity) {
    PoolSubpage<T> head = arena.findSubpagePoolHead(normCapacity); // 注解@1
  	int d = maxOrder; 
    synchronized (head) {
        int id = allocateNode(d); // 注解@2
			  if (id < 0) {
            return id;
        }
        final PoolSubpage<T>[] subpages = this.subpages; // 注解@3
        final int pageSize = this.pageSize;
        freeBytes -= pageSize;
        int subpageIdx = subpageIdx(id); // 注解@4

        PoolSubpage<T> subpage = subpages[subpageIdx]; 
        if (subpage == null) { // 注解@5
            subpage = new PoolSubpage<T>(head, this, id, runOffset(id), pageSize, normCapacity);
            subpages[subpageIdx] = subpage;
        } else {
            subpage.init(head, normCapacity);
        }
        return subpage.allocate(); // 注解@6
    }
}
```

**注解@1** 从tinySubpagePools中获取PoolSubpage。获取过程为elemSize >>> 4（除以16）来获取。

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210403195729.png)



**tinySubpagePools结构**

tinySubpagePools被初始化成长度为32的数组，元素之间差额为16B。

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/tinySubpagePools%E5%AD%98%E5%82%A8%E7%BB%93%E6%9E%84%E6%96%B0.png)

**注解@2** allocateNode 在上一篇文章分析过，d = maxOrder = 1。表示在平衡二叉树的第11层找到可分配的节点，具体为memoryMap数组中的下标。如果整个树都没有内存可分配了，返回的id=-1。



**注解@3** 先看下subpages的初始化，maxSubpageAllocs = 1 << maxOrder= 2048。也就是PoolSubpage<T>[] subpages的长度为平衡二叉树第11层所有的节点数（2^11）。

```java
subpages = newSubpageArray(maxSubpageAllocs);
```



**注解@4** 将平衡二叉树第11层的下标memoryMap[]的下标转换为subpages[]数组的下标。转换关系为memoryMapIdx ^ maxSubpageAllocs。

例如：平衡二叉树第11层第1个节点数组下标为2048，转换为subpages的下标为0，平衡二叉树第11层第2个节点数组下标为2049，转换为subpages的下标为1，平衡二叉树第11层第2个节点数组下标为2050，转换为subpages的下标为2。

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/%E5%B9%B3%E8%A1%A1%E4%BA%8C%E5%8F%89%E6%A0%91%E5%88%86%E5%B8%83%20(1).png)



**注解@5** 初始化PoolSubpage

```java
PoolSubpage(PoolSubpage<T> head, PoolChunk<T> chunk, int memoryMapIdx, int runOffset, int pageSize, int elemSize) {
  this.chunk = chunk;
  this.memoryMapIdx = memoryMapIdx;
  this.runOffset = runOffset;
  this.pageSize = pageSize;
  bitmap = new long[pageSize >>> 10]; // pageSize / 16 / 64
  init(head, elemSize);
}
```

**参数说明** 

```
head: PoolSubpage数组中的一个元素，本例中为第4个元素

chunk: 当前PoolChunk实例

memoryMapIdx: 平衡二叉树第11层用于分配的节点，具体为memoryMap数组下标

elemSize: 待分配的内存，本例中为128KB

bitmap: long数组长度为8「8192无符号右移10位=8」
```

**初始化说明**

```java
void init(PoolSubpage<T> head, int elemSize) {
     doNotDestroy = true;
  	 // 待分配内存
     this.elemSize = elemSize; 
     if (elemSize != 0) {
            // maxNumElems表示可以被切割成几份（8192除以待分配内存）例如：64=8192/128被切成了64份
            maxNumElems = numAvail = pageSize / elemSize;
            nextAvail = 0;
            // 无符号右移6位，高位补零（相当于除以64）例如：64的二进制右移6位为1，128的二进制右移6位为2
            bitmapLength = maxNumElems >>> 6;
            if ((maxNumElems & 63) != 0) { // 相当于是否能被64整除
                bitmapLength ++; // 不能被整除递增bitmapLength
            }

            for (int i = 0; i < bitmapLength; i ++) {
                bitmap[i] = 0; // 等于零表示未被分配
            }
     }
     addToPool(head);
}
```

**过程说明** 

@1 先计算一个Page被切成了几份 maxNumElems（ pageSize / elemSize）

@2 计算bitmap数组长度bitmapLength（maxNumElems无符号右移6位相当于除以64）

**备注：** 此处不太好理解为什么要maxNumElems要除以64来计算bitmap的长度呢？也就是bitmap数组中的每个元素可以标记64个被切的内存块。bitmap是long数组，每个long类型是64位，他用每个二进制位来标记被切内存块的分配情况。

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/bitmap%E4%BA%8C%E8%BF%9B%E5%88%B6%E4%BD%8D%E5%88%9D%E5%A7%8B%E5%8C%96%E7%8A%B6%E6%80%81%20(1).png)

**加入链表** 

新构建的PoolSubpage与tinySubpagePools中的PoolSubpage建成链表关系。

```java
private void addToPool(PoolSubpage<T> head) {
    assert prev == null && next == null;
    prev = head;
    next = head.next;
    next.prev = this;
    head.next = this;
}
```

**小结：** 构造的PoolSubpage中持有了一个bitmap[]数组，数组长度与待分配的内存有关。待分配内存大小为elemSize，数组长度=PageSize/elemSize，并将bitmap数组的元素标记为未分配。



![](https://gitee.com/laoliangcode/md-picture/raw/master/img/subpages%E5%AD%98%E5%82%A8%E7%BB%93%E6%9E%84%20(2).png)



**注解@6** 分配内存

内存的分配以两次分配128B内存为例观察期分配过程。

```java
@Test
public void testAllocateSubpage() {
   ByteBufAllocator allocator = new PooledByteBufAllocator();
   allocator.directBuffer(128); // 第一次分配
   allocator.directBuffer(128); // 第二次分配
}
```

**第一次分配**

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210410194035.png)

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210410195035.png)



**第二次分配**

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210410200757.png)

第一次轮询第一位已被占用，需要向右移位。

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210410201041.png)

第二次轮询第二位未被占用。

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210410201407.png)

第二次分配过程

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210410202216.png)



# 两次内存分配图示

**第一次分配128B图示**

此时64位第一位被标记为1，bitmap[0] = 1

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/%E7%AC%AC%E4%B8%80%E6%AC%A1%E5%86%85%E5%AD%98%E5%88%86%E9%85%8D.png)

**第二次分配128B图示**

此时64位第二位也被标记为1，bitmap[0] = 3

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/%E7%AC%AC%E4%BA%8C%E6%AC%A1%E5%86%85%E5%AD%98%E5%88%86%E9%85%8D%20(1).png)