

```
title: No.181# 点直播简要架构梳理走查
categories: 架构设计
tags: 架构设计
date: 2023-01-02 11:55:01
```



# 引言

直播带货、潮流电商、短视频不断融合，本文走查下音视频直播的简要架构和角色。选择UDP，注重传输实时性，在线教育、音视频会议等。选择TCP，注重画面质量、是否卡顿等，娱乐直播、直播带货等。

本文主要内容有：

* 音视频直播架构
* 点直播服务器搭建
* CDN内容分发网络



# 一、音视频直播架构



下图为音视频直播架构简图。



![](https://raw.githubusercontent.com/yongliangcode/md-picture/master/img2/20230102121159.png)



### 1、涉及的角色

**直播客户端（主播）** 

* 功能主要包括音视频数据的采集、编码、推流
* 从摄像头、麦克风采集数据，并对数据进行编码后通过RTMP协议发送给CDN源节点

**直播客户端（观众）** 

* 功能主要包括拉流、解码、播放
* 从直播系统获取房间流媒体地址
* 通过RTMP协议从CDN边缘节点获取数据、解码、渲染

**信令服务器** 

* 接受指令并处理业务逻辑，创建房间、加入房间、送礼物等

**CND网络** 

* 内容分发网络（Content Delivery Network）

* 利用最靠近每位用户的服务器，更快、更可靠地将音乐、图片、视频、应用程序等发送给用户来提供高性能
* 提供异地备援，100%的高可用性



### 2、传输协议

**RTMP协议**

* 实时消息传输协议，Real Time Messaging Protocol的缩写
* 最初由Macromedia为通过互联网在Flash播放器与一个服务器之间传输流媒体音频、视频和数据而开发的一个专有协议
* 基于TCP，默认使用1935端口的“明文”协议

**HLS协议**

* 苹果公司提出基于HTTP的流媒体网络传输协议，HTTP Live Streaming的缩写
* 工作原理是把整个流分成一个个小的基于HTTP的文件来下载，每次只下载一些
* HLS只请求基本的HTTP报文，与实时传输协议（RTP）不同，HLS可以穿过任何允许HTTP数据通过的防火墙或者代理服务器
* 根据客户端带宽情况自适应调整码率，例如使用FFmpeg可以将视屏文件转换为HLS切片



### 3、整体流程

* 直播客户端（主播）向信令服务器发起信令创建直播间
* 信令服务器收到指令后返回CDN源站推流地址
* 直播客户端（主播）通过音/视频采集设备采集数据后编码、通过RTMP协议发送给CDN网络
* 直播客户端（观众）向信令服务器发起信令加入直播间
* 信令服务器收到指令后向客户端（观众）推送其附近的CND边缘节点地址
* 直播客户端（观众）收到地址后使用RTMP/HLS协议拉取直播数据





# 二、点直播服务器搭建



下面两种方式比较快速搭建点直播服务器。



### 方式一

* 使用Nginx+RTMP 推拉流插件
* Nginx RTMP Module支持RTMP/HLS/MPEG-DASH 协议

```
https://github.com/arut/nginx-rtmp-module
https://nginx.org/download/
```



### 方式二

* 使用开源SRS服务器
* SRS是一个简单高效的实时视频服务器，支持RTMP/WebRTC/HLS/HTTP-FLV/SRT/GB28181

```
https://ossrs.net/lts/zh-cn/docs/v4/doc/introduction
```





# 三、CDN内容分发网络

CDN内容分发网络（Content Distribution Network）是指一种透过互联网互相连接的电脑网络系统，利用最靠近每位用户的服务器，更快、更可靠地将音乐、图片、视频、应用程序及其他文件发送给用户，来提供高性能、可扩展性及低成本的网络内容传递给用户。

* 提高网页加载速度
* 提高文件下载速度
* 提高视频播放速度



![](https://raw.githubusercontent.com/yongliangcode/md-picture/master/img2/20230102155635.png)



云厂商提供的CDN服务

```
阿里云：https://www.aliyun.com/product/cdn
腾讯云：https://cloud.tencent.com/product/cdn
华为云：https://www.huaweicloud.com/product/cdn.html
七牛云：https://www.qiniu.com/products/qcdn#scene
```

