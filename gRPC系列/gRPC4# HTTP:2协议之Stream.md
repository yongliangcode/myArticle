---
title: gRPC4# HTTP/2协议之Stream
categories: gRPC
tags: gRPC
date: 2020-12-05 11:55:01
---



# 前言

前面三篇介绍了HPPT/2的“连接前言”、“二进制桢”、“头部压缩”。本文从“流及多路复用”、“流状态”、“流量控制”、“流优先级”、“HTTP/2扩展”介绍HTTP/2协议流相关知识。 



<!--more-->



# 流与多路复用



**流**

前面介绍桢格式时，每个桢都有一个流标示，标记自己属于哪个流。通过将相同流标识的桢组装，桢之间时有严格顺序的，即形成了“流”。

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210225100227.png)



**多路复用**

一个HTTP/2连接可以并非很多个流，流ID顺序递增且互相独立，形成多路复用。由客户端发起的流ID为奇数，服务端发起的为偶数。



# 流状态

**idle**

流空闲状态，可以发送接收HEADERS帧。



**open**

流开启状态，idle发送或者接受HEADERS帧后，状态变更为开启



**half closed**

发送包含END_STREAM桢的一端流转为本地半关闭half closed(local)，表示客户端发送请求数据完毕，等待服务端响应数据，接受到服务端发送的END_STREAM进入close关闭状态。接受END_STREAM桢的另一端称为远程半关闭状态half closed(remote)，表示服务端知道客户端请求已经发送完毕，处理结束后可以发送响应数据，并发送END_STREAM到客户端，进入close关闭状态。



**close**

流的关闭状态。除了half closed数据发送结束关闭外，发送RST_STREAM(发生错误或取消)也可关闭流。



**流状态交互示意图**

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210225100341.png)



# 流量控制

流量控制是保护接收方的机制，通过配额机制实现。发送端每发送数据后window窗口大小相应的减少。当发送端收到接收端WINDOW_UPDATE桢后window窗口增加。window等于0则不可以进行发送，窗口初始值为65535字节。 



![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210225100506.png)

# 流优先级

通过发送端向接收端发送优先级权重期待接收端给予资源分配支持，接受端不保证一定遵守，默认权重为16。优先级表达可以通过HEADERS或者单独发送PRIORITY帧实现。



![image-20210225100647921](/Users/yongliang/Library/Application Support/typora-user-images/image-20210225100647921.png)





# 流依赖

客户端通过PRIORITY帧可以告诉服务端当前流所依赖的流，形成流依赖树。同一父级的各个字节点通过权重分配资源；父级先分配资源传输结束后，再分配子级资源。



![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210225100804.png)





# 总结



通HTTP/2的四篇文章，对HTTP2工作原理有了全局的认识，相信再阅读HTTP/2相关文献不再困难。



