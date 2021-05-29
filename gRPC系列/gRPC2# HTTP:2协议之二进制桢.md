---
title: gRPC2# HTTP/2协议之二进制桢
categories: gRPC
tags: gRPC
date: 2020-12-02 11:55:01
---



# 前言

HTTP/2的报文是以二进制桢发送的。那桢格式、桢大小、桢类型是怎么样的？本文会整理桢的格式以及十种桢类型。



<!--more-->



# 桢格式

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210225094347.png)

##### 1.桢格式说明

桢的格式由9个字节的桢头和桢数据Payload构成；桢头由3个字节的桢长度、1个字节的桢类型、1个字节的标志位、4个字节的流标识符（含1位R保留位）构成。

**桢长度**
桢长度由24位3个字节大小表示。取值在2^14(16,384)与2^24-1(16,777,215)之间；可在接收方SETTINGS_MAX_FRAME_SIZE设置。

**桢类型**
桢类型用8位1个字节表示，说明桢的格式和语义。具体桢的类型详见下文介绍。

**标志位**
标志位用8位1个字节表示。例如：END_HEADERS标志表示头数据传输结束；END_STREAM表示单方向数据传输结束。

**R**
R即1位保留字段，未定义，以0x0结尾。

**流标识符**
流标识符用31位表示，上限为2^31。接收方可以根据流标识ID进行组装，同一个Stream中内Frame必须是有序的，所以接受方根据流ID可以拼接成有序的流。另外：客户端发起的流用奇数表识；服务器发起的流用偶数标识。正因为使用了流标识，接收端可以将并发的Stream进行有序拼接，实现多路复用。

**桢数据**
传输的数据内容Payload由桢类型决定。



##### 2.Wireshark抓包截图

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210225094502.png)



# 桢类型



###### 1.DATA桢

数据桢主要存储HTTP/2数据报文，具体格式如下图：

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210225094537.png)

**字段含义**
Padding: 8位填充字节，填充字节可以改变DATA桢的大小，可以启到安全性功能
Pad Length: 填充字节的长度；PADDED标记为true时表明有填充字节
Data: 具体传输的数据

**Wireshark抓包截图**

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210225094605.png)



###### 2.Header桢

Header桢的结构如下图：

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210225094628.png)

**字段含义**
Pad Length：填充字节的长度，填充字节含义同上述Data桢
E：表识流是否为独占的。设置PRIORITY时才有值
Stream Dependency：该流的依赖流。设置PRIORITY时才有值
Weight：流优先级权重。设置PRIORITY时才有值
Header Block Fragment：Header块片段
Padding：填充的字节长度



**Wireshark抓包截图**

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210225094649.png)



###### 3.PRIORITY帧

发送流的优先级，格式如下，各字段含义与抓包截图见Header桢。

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210225094710.png)



###### 4.RST_STREAM帧

当发生错误或者取消时，用于终止一个流。

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210225094726.png)

**字段含义**
Error Code: 32位错误代码，指发生错误的原因。



**Wireshark抓包截图**

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210225094750.png)



###### 5.SETTINGS帧

用于传达连接端点之间的配置参数。
SETTINGS帧的标记ACK为0表示被对等的SETTINGS桢使用；ACK不为0时表示FRAME_SIZE_ERROR的连接错误。



**桢格式**

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210225094817.png)



**Identifier参数含义**
SETTINGS_HEADER_TABLE_SIZE：通知接收方header解码表（解码header块）的最大尺寸
SETTINGS_ENABLE_PUSH：初始值1表示允许服务端推送，0表示不允许服务端推送
SETTINGS_MAX_CONCURRENT_STREAMS：最大的并发流数（发送者）
SETTINGS_INITIAL_WINDOW_SIZE：stream窗口大小，默认为65535
SETTINGS_MAX_FRAME_SIZE：桢负载大小
SETTINGS_MAX_HEADER_LIST_SIZE：Header列表的最大值



**Wireshark抓包截图**

![image-20210225094839960](/Users/yongliang/Library/Application Support/typora-user-images/image-20210225094839960.png)



**6.PUSH_PROMISE帧**

服务端向客户端推送的桢，客户端可以返回RST_STREAM拒绝。
图中R为保留位。

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210225094901.png)

###### 7.PING帧

心跳检测，测量发送往还时间，确定连接是否正常。
标记ACK为0即false表示为PING桢的响应（response）；1即True表示PING桢。



**桢格式**

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210225094930.png)



**Wireshark抓包截图**

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210225094951.png)



###### 8.GOAWAY帧

用于关闭连接或者发出错误，允许停止接受新的流并完成前面的流处理。



**桢格式**

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210225095017.png)



**Wireshark抓包截图**

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210225095034.png)



**9.WINDOW_UPDATE帧**

用于连接和流的流量控制。

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210225095054.png)

**Wireshark抓包截图**

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210225095111.png)

##### 10.CONTINUATION

CONTINUATION一种持续桢用于继续传输Header头块片段。通常在Header块比较大，在HEADERS、PUSH_PROMISE、CONTINUATION桢之后继续传输。

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210225095128.png)

# 小结

通过对二进制桢内容的整理和走查，对HTTP/2通信的各种桢不再陌生，根据桢的类型可以知道通信双方在做什么操作。欢迎跟作者互动、共同探讨。