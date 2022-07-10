---
title: Netty15# 池化内存Normal类型内存分配
categories: Netty
tags: Netty
date: 2021-03-13 18:25:01
---



# 前言 

Netty所谓的池化就是先申请了一块大内存，后面需要分配的时候就来我这里分就完了。以堆外直接内存分配为例，Netty以Chunk为单位16M申请了一块连续内存，这么一大块内存是以平衡二叉树的形式组织起来的。分配的时候就从这颗树上找合适的节点。池化内存的分配是Netty的最为核心部分，这块的代码很多位运算，不太容易看懂，读的时候需要边调试边分析。



# 平衡二叉树



Netty使用平衡二叉树将申请到的Chunk块组织起来，如下图所示，并使用数组将整个树映射进去，见下文构造函数中memoryMap。



![](https://raw.githubusercontent.com/laoliangcode/md-picture/raw/master/img/Netty%E5%B9%B3%E8%A1%A1%E4%BA%8C%E5%8F%89%E6%A0%91.png)





<!--more-->



### 构造函数

```java
PoolChunk(PoolArena<T> arena, T memory, int pageSize, int maxOrder, int pageShifts, int chunkSize, int offset) {
        unpooled = false;
        this.arena = arena;
        this.memory = memory;
        this.pageSize = pageSize;
        this.pageShifts = pageShifts;
        this.maxOrder = maxOrder;
        this.chunkSize = chunkSize;
        this.offset = offset;
        unusable = (byte) (maxOrder + 1);
        log2ChunkSize = log2(chunkSize);
        subpageOverflowMask = ~(pageSize - 1);
        freeBytes = chunkSize;
        assert maxOrder < 30 : "maxOrder should be < 30, but is: " + maxOrder;
        maxSubpageAllocs = 1 << maxOrder;
        memoryMap = new byte[maxSubpageAllocs << 1];
        depthMap = new byte[memoryMap.length];
        int memoryMapIndex = 1;
        for (int d = 0; d <= maxOrder; ++ d) {
            int depth = 1 << d;
            for (int p = 0; p < depth; ++ p) {
                memoryMap[memoryMapIndex] = (byte) d;
                depthMap[memoryMapIndex] = (byte) d;
                memoryMapIndex ++;
            }
        }
        subpages = newSubpageArray(maxSubpageAllocs);
        cachedNioBuffers = new ArrayDeque<ByteBuffer>(8);
    }
```

**关键参数** 

| 参数                  | 说明                                    |
| --------------------- | --------------------------------------- |
| memory                | 申请的一块内存大小为16M，以字节数组表示 |
| pageSize              | 8KB                                     |
| pageShifts            | 13                                      |
| maxOrder              | 11                                      |
| chunkSize             | 16M                                     |
| log2ChunkSize         | 14                                      |
| subpageOverflowMask   | -8192                                   |
| maxSubpageAllocs      | 2048                                    |
| maxSubpageAllocs << 1 | memoryMap数组的长度，4096               |
| memoryMap             | 将二叉树平铺在该数组，格式见下文。      |
| depthMap              | 同memoryMap将二叉树平铺在该数组         |



上面这些有些是直接传入的，有的一眼看不出结果，代入公式算算。



**log2ChunkSize** 这个是chunkSize的对数，chunkSize大小是16M

```java
@Test
public void testlog2ChunkSize(){

	System.out.println("16M的对数：" + log2(16384));

	System.out.println("2的14次方：" + Math.pow(2, 14));
	
}

private static int log2(int val) {
	int INTEGER_SIZE_MINUS_ONE = Integer.SIZE - 1;
	// compute the (0-based, with lsb = 0) position of highest set bit i.e, log2
	return INTEGER_SIZE_MINUS_ONE - Integer.numberOfLeadingZeros(val);
}

输出：
16M的对数：14
2的14次方：16384.0
```

**subpageOverflowMask** 主要用于位运算的“&”操作判断。

```java
@Test
public void testSubpageOverflowMask(){
  int pageSize = 8192;
  int subpageOverflowMask = ~(pageSize - 1);
  System.out.println("subpageOverflowMask:" + subpageOverflowMask);
}

输出：
subpageOverflowMask:-8192
```

**maxSubpageAllocs** 用于限制数组的长度

```java
@Test
public void testMaxSubpageAllocs(){
   int maxOrder = 11;
   int maxSubpageAllocs = 1 << maxOrder;
   System.out.println("maxSubpageAllocs:" + maxSubpageAllocs);
   int length = maxSubpageAllocs << 1;
   System.out.println("memoryMap数组的长度:" + length);
}
输出：
maxSubpageAllocs:2048
memoryMap数组的长度:4096
```



**memoryMap&depthMap** 

这两个数组主要把平衡二叉树装到了数组里，外层循环控制层高，内层循环控制每层的节点数。直接看还是看不出到底啥格式，不要紧代入输出看看。

```java
 @Test
 public void testMemoryMap(){
 	int maxOrder = 11;
  int memoryMapIndex = 1;
  int maxSubpageAllocs = 2048;
  byte[] memoryMap = new byte[maxSubpageAllocs << 1];
  byte[] depthMap = new byte[memoryMap.length];
  for (int d = 0; d <= maxOrder; ++ d) {
            int depth = 1 << d;
            for (int p = 0; p < depth; ++ p) {
                memoryMap[memoryMapIndex] = (byte) d;
                depthMap[memoryMapIndex] = (byte) d;
                memoryMapIndex ++;
            }
   }
   System.out.println("memoryMap的长度：" + memoryMap.length);
   System.out.println("memoryMap数组内容:" + Arrays.toString(memoryMap));
}

输出：
memoryMap的长度：4096
memoryMap数组内容:[0, 0, 1, 1, 2, 2, 2, 2, 3, 3, 3, 3, 3, 3, 3, 3, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 5, ..... ]
```



### 小结

由输出可以看出memoryMap的长度为4096，内容格式如上，值表示所在的层，例如：数字1表示该节点在第树的第1层；有两个1表示该层有两个叶子节点。也就是从2的0次方，一直到2的11次方，平铺到了数组中。



# 平衡二叉树查找更新过程

### 三次分配示例

Normal类型的内存分配，主要是如何在二叉树中找到匹配的节点的过程，以及该节点的被分配后整个树的状态更新变化。

源码部分在PoolChunk#allocateRun(int normCapacity) 部分，为了方便debug和调试，把值代入后单独拎出来跑一跑。

下面代码可以直接运行，以执行三次分配，每次分配8KB的过程来看其对平衡二叉树的查找过程。

```java
public class PoolChunkTest {

    byte[] memoryMap = null;

    private final byte unusable = (byte) (11 + 1);

    public static void main(String [] args){
        PoolChunkTest poolChunkTest = new PoolChunkTest();
        poolChunkTest.initMemoryMap();
        int normCapacity = 8 * 1024;
        long id01 = poolChunkTest.allocateRun(normCapacity);
        System.out.println("第一次分配：" + id01);

        long id02 = poolChunkTest.allocateRun(normCapacity);
        System.out.println("第二次分配：" + id02);

        long id03 = poolChunkTest.allocateRun(normCapacity);
        System.out.println("第三次分配：" + id03);

    }

    private long allocateRun(int normCapacity) {
        int d = 11 - (log2(normCapacity) - 13);
        int id = allocateNode(d);

        return id;
    }

    private int allocateNode(int d) {
        int id = 1;
        int initial = - (1 << d); // has last d bits = 0 and rest all = 1
        byte val = value(id);
        if (val > d) { // unusable
            return -1;
        }
        while (val < d || (id & initial) == 0) { // id & initial == 1 << d for all ids at depth d, for < d it is 0
            id <<= 1;
            val = value(id);
            if (val > d) {
                id ^= 1;
                val = value(id);
                System.out.println("val:" + val);
            }
        }
        byte value = value(id);
        assert value == d && (id & initial) == 1 << d : String.format("val = %d, id & initial = %d, d = %d",
                value, id & initial, d);
        setValue(id, unusable); // mark as unusable
        updateParentsAlloc(id);
        return id;
    }

    private void updateParentsAlloc(int id) {
        while (id > 1) {
            int parentId = id >>> 1;
            byte val1 = value(id);
            byte val2 = value(id ^ 1);
            byte val = val1 < val2 ? val1 : val2;
            setValue(parentId, val);
            id = parentId;
        }
    }

    private void setValue(int id, byte val) {
        memoryMap[id] = val;
    }


    private static int log2(int val) {
        int INTEGER_SIZE_MINUS_ONE = Integer.SIZE - 1;
        // compute the (0-based, with lsb = 0) position of highest set bit i.e, log2
        return INTEGER_SIZE_MINUS_ONE - Integer.numberOfLeadingZeros(val);
    }

    private byte value(int id) {
        return memoryMap[id];
    }


    public void initMemoryMap(){
        int maxOrder = 11;
        int memoryMapIndex = 1;
        int maxSubpageAllocs = 2048;
        memoryMap = new byte[maxSubpageAllocs << 1];
        byte[] depthMap = new byte[memoryMap.length];
        for (int d = 0; d <= maxOrder; ++ d) {
            int depth = 1 << d;
            for (int p = 0; p < depth; ++ p) {
                memoryMap[memoryMapIndex] = (byte) d;
                depthMap[memoryMapIndex] = (byte) d;
                memoryMapIndex ++;
            }
        }
    }

}
```

输出：

```
第一次分配：2048
第二次分配：2049
第三次分配：2050
```



例子中分配的8KB，根据公式 int d = 11 - (log2(normCapacity) - 13)算出其在11层，所以下文中三次分配时入参d=11。



### 第一分配8KB



第一次分配找到了数组memoryMap的下标为2048，此时对应的值为memoryMap[2048]=11。

![](https://raw.githubusercontent.com/laoliangcode/md-picture/raw/master/img/20210314192404.png)



当分配后将该节点标记为不可用，也就是更新为12（总共才11层，所以12为不可用），此时memoryMap[2048]=12。

![](https://raw.githubusercontent.com/laoliangcode/md-picture/raw/master/img/20210314174107.png)



递归整棵树，从下往上更新直到根节点，将父节点更新为其子节点的最小值。例如：memoryMap[1024]的值原来为10，被更新成了11.

![](https://raw.githubusercontent.com/laoliangcode/md-picture/raw/master/img/20210314195243.png)





### 第二次分配8KB

先找到了id=2048，这个节点发现其值为12，也就是不可用了。此时memoryMap[2049]=11。

![](https://raw.githubusercontent.com/laoliangcode/md-picture/raw/master/img/20210314185849.png)



通过id ^= 1找到其兄弟节点id=2049，其对应的值为11可用。返回该节点。

![image-20210314185941227](/Users/yongliang/Library/Application Support/typora-user-images/image-20210314185941227.png)



将节点id=2049设置为不可用，即：memoryMap[2049]=12。

![](https://raw.githubusercontent.com/laoliangcode/md-picture/raw/master/img/20210314200044.png)



递归更新整棵树，由于其父节点的两个子节点都被分配出去了，所以1024被标记为不可用。memoryMap[1024]=12。

![](https://raw.githubusercontent.com/laoliangcode/md-picture/raw/master/img/20210314200415.png)



### 第三次分配8KB

第三次分配8KB时，当循环到了节点1024，发现其不可用，也就是其子节点也不可用了。

![](https://raw.githubusercontent.com/laoliangcode/md-picture/raw/master/img/20210314192117.png)



通过id <<= 1找到1024的兄弟节点1025，接着向下查找。

![](https://raw.githubusercontent.com/laoliangcode/md-picture/raw/master/img/20210314191633.png)



找到节点1025的子节点2050发现其可用，即：memoryMap[2050]=11。找到后最后过程同上，标记其不可用表示已分配了，并更新整个树把其父节点更新为子节点的最小值。

![](https://raw.githubusercontent.com/laoliangcode/md-picture/raw/master/img/20210314190313.png)





# 平衡二叉树查找更新图示



### 第一次分配8KB前

整个树都没有被占用，8KB会被选在第11层分配。



![](https://raw.githubusercontent.com/laoliangcode/md-picture/raw/master/img/%E5%B9%B3%E8%A1%A1%E4%BA%8C%E5%8F%89%E6%A0%91%E7%AC%AC%E4%B8%80%E6%AC%A1%E5%88%86%E9%85%8D.png)



### 第一次分配8KB后



红色标记为变化部分，节点红色表示被占用。第11层的第一个节点memoryMap[2048]被标记为不可用。第10层的第一个节点memoryMap[1024]值被从10更新为11，整个树会继续向上递归变化，将整个树父节点更新为子节点最小的值。



![](https://raw.githubusercontent.com/laoliangcode/md-picture/raw/master/img/%E5%B9%B3%E8%A1%A1%E4%BA%8C%E5%8F%89%E6%A0%91%E7%AC%AC%E4%B8%80%E6%AC%A1%E5%88%86%E9%85%8D%E5%90%8E.png)



### 第二次分配8KB后



第二次分配8KB后，第11层的第二个节点memoryMap[2049]被标记为不可用，其父节点memoryMap[1024]由于其子节点都被分配完毕，也被标记为不可用。整个树会继续向上递归变化，将整个树父节点更新为子节点最小的值。



![](https://raw.githubusercontent.com/laoliangcode/md-picture/raw/master/img/%E7%AC%AC%E4%BA%8C%E6%AC%A18KB%E5%88%86%E9%85%8D%E5%90%8E.png)



### 第三次分配8KB后



第三次分配8KB后，将第11层的第三个节点memoryMap[2050]标记为不可用，同时更新其父节点memoryMap[1025]为子节点最小值11。整个树会继续向上递归变化，将整个树父节点更新为子节点最小的值。

![](https://raw.githubusercontent.com/laoliangcode/md-picture/raw/master/img/%E7%AC%AC%E4%B8%89%E6%AC%A18KB%E5%88%86%E9%85%8D%E5%90%8E.png)

