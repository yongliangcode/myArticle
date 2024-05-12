---
title: No.179# IM即时通信知识点梳理
categories: 方案设计
tags: 方案设计
date: 2022-12-11 11:55:01
---



# 一、资源限制与隔离



### 资源限制

Linux Cgroups，Linux Control Group制造约束主要手段、限制一个进程组使用的资源上限（CPU、内存、磁盘、网络带宽）。

| 名称           | 说明                                            |
| -------------- | ----------------------------------------------- |
| cpu子系统      | 限制进程的 cpu 使用率                           |
| cpuacct 子系统 | 统计 cgroups中的进程的cpu使用报告               |
| cpuset 子系统  | cgroups中的进程分配单独的cpu节点或者内存节点    |
| memory 子系统  | 限制进程的 memory 使用量                        |
| blkio 子系统   | 限制进程的块设备 IO                             |
| devices 子系统 | 控制进程能够访问某些设备                        |
| net_cls 子系统 | 可使用tc模块（traffic control）对数据包进行控制 |
| freezer 子系统 | 挂起或者恢复 cgroups 中的进程                   |
| ns 子系统      | 不同 cgroups 下面的进程使用不同的 namespace     |

例如：每100ms的时间里控制组只能使用30ms

docker run -it --cpu-period=100000 --cpu-quota=30000 ubuntu /bin/bash



### 资源隔离

资源隔离，Linux Namespace修改进程视图、将部分系统资源隔离、不同命名空间不可见。

例如：进程 ID 命名空间（PID namespaces），创建进程（clone）时使用CLONE_NEWPID方式（PID namespaces）、隔离进程的 PID 空间，不同的 PID 命名空间中的 PID 可以重复。

```
clone(main_function, stack_size, CLONE_NEWPID | SIGCHLD, NULL)
```



Linux内核共支持八种命名空间：

| 名称               | 说明                                                     |
| ------------------ | -------------------------------------------------------- |
| 挂载命名空间       | 隔离挂载点等信息、对容器视图的改变需重新挂载才能生效     |
| Unix 主机命名空间  | 隔离主机名与域名等信息                                   |
| 进程间通信命名空间 | 隔离 System V IPC                                        |
| 进程 ID 命名空间   | 隔离进程的 PID 空间                                      |
| 网络命名空间       | 虚拟化一个完整的网络栈，每个网络栈拥有一套完整的网络资源 |
| 用户命名空间       | 隔离用户与组信息                                         |
| Cgroup 命名空间    | 隔离 cgroup 层次结构                                     |
| 系统时间命名空间   | 允许不同的进程看到不同的系统时间                         |



### 问题

容器单进程模型，下面的问题在流控平台基于CPU做自适应限流时，发现获取的CPU不是容器而是宿主机的。

/prof 文件，记录当前内核运行状态（CPU、内存等）。容器执行TOP命令显示的是宿主机的内核信息，而不是当前容器的。

原因：/prof/stats 文件系统并不知道用户通过 Cgroups 做了限制、生产环境需要重新修正。

解决：通过lxcf来自动维护宿主机cgroup中容器的真实资源信息与容器内`/proc`下文件的映射关系

| 容器内目录                     | 宿主机lxcfs目录                                             |
| ------------------------------ | ----------------------------------------------------------- |
| /proc/cpuinfo                  | /var/lib/lxcfs/{container_id}/proc/cpuinfo                  |
| /proc/meminfo                  | /var/lib/lxcfs/{container_id}/proc/meminfo                  |
| /proc/diskstats                | /var/lib/lxcfs/{container_id}/proc/diskstats                |
| /proc/stat                     | /var/lib/lxcfs/{container_id}/proc/stat                     |
| /proc/swaps                    | /var/lib/lxcfs/{container_id}/proc/swaps                    |
| /proc/uptime                   | /var/lib/lxcfs/{container_id}/proc/uptime                   |
| /proc/loadavg                  | /var/lib/lxcfs/{container_id}/proc/loadavg                  |
| /sys/devices/system/cpu/online | /var/lib/lxcfs/{container_id}/sys/devices/system/cpu/online |





容器镜像/根文件系统（rootfs）

文件的重新挂载与隔离

挂载命名空间：Mount Namespace隔离挂载点等信息、对容器视图的改变需重新挂载才能生效

Linux Chroot：改变当前环境的根目录到一个文件夹、这个文件夹之外的东西、对于当前环境是不可见的

通常在一个容器的根目录挂载一个完整操作系统文件目录，这个文件系统即容器镜像，另名根文件系统rootfs

联合文件系统UnionFS（Union File System)：将多个位置的目录联合挂载到同一个目录下，即层（layer）。

只读层（ro+wh->readonly+whiteout）： 挂载的方式是只读的

可读写层（rw->read write）：对容器的修改、在该层出现

AuFS通过在可读写层创建一个without文件，通过将只读层的两个文件联合挂载，将只读层遮挡起来。

Init层：位于只读层与可读写层之间，专门存放/etc/hots、/etc/resolv.conf等，用于容器启动时初始化参数env、hostname等。



命名空间切换与数据管理



不同的 Namespace 独立于操作系统的其他部分，可以拥有自己的网络、挂载点、进程、IPC 等资源，从而实现进程之间的隔离。



通过加入进程所在的Namesapce当中，实现进入进程所在的容器，setns(fd, 0) 方法允许在进程之间切换 Linux 命名空间。



setns(fd, 0) 方法将当前进程切换到由文件描述符 fd 指定的 Namespace 中，fd指向尚未关闭的 Namespace 的文件描述符。



Volume机制： Docker 中用来实现数据管理，容器与主机或其他容器之间共享文件或者数据块。允许将宿主机上指定的目录或文件，挂载到容器里面进行读取和修改操作。



Docker 内置了多种类型的 Volume 存储驱动：

| Volume 存储驱动  | 说明                                                         |
| ---------------- | ------------------------------------------------------------ |
| Host Volume      | 直接绑定主机文件系统上的某个文件或目录，挂载到容器中使用     |
| Anonymous Volume | 为容器内的某个目录创建一个匿名卷，匿名卷仅用于容器内部数据存储，在主机上也无法看到这个目录 |
| Named Volume     | 可以为某个特定容器命名一个永久性的卷，并将其挂载到该容器的指定目录，<br />使得数据不会随着容器的销毁而丢失，而且可以在独立的主机或容器之间共享使用 |



Volume 机制的主要特点：

| Volume 特点 | 说明                                                         |
| ----------- | ------------------------------------------------------------ |
| 隔离性      | Volume 可以为每个容器提供一个本地文件系统空间，具有容器隔离的效果 |
| 持久化      | 任何被写入 Volume 中的数据都会持久保存在本地主机磁盘上,即使容器被删除 |
| 数据共享    | 可以在独立的主机或容器之间轻松地共享数据                     |
| 管理灵活    | 可以通过命令行或者 Compose 文件管理 Volume，方便实现增、删、改、查等功能 |



Docker中通过「docker run -v」申明Volume。



静态视图：联合挂载在 /var/lib/docker/aufs/mnt 上的 rootfs

动态视图：由 Namespace+Cgroups 构成的隔离环境







![img](https://ask.qcloudimg.com/http-save/yehe-3911980/0e46d55a13e243101edc43c65da015a2.png)



由控制节点Master节点和计算节点node两种计算节点组成。



控制节点Master由kube-apiserver、kube-scheduler、kube-controller-manager组成。

**kube-apiserver：** 负责API服务与Kubernetes资源交互的入口，可以通过kubectl或者client-go其他语言类库进行访问

**kube-scheduler:**   负责资源调度与计算，将Pod按照特定策略分发到计算节点

**kube-controller-manager:**  负责资源编排，运行一系列控制器的组件，比如：节点控制器、任务控制器、端点控制器等



计算节点的最核心组件kubelet。



**kubelete：** 

* 负责通过CRI（Container Runtime Interface）同容器运行时打交道，定义了容器运行时的核心操作
* 通过监听控制面接受指令，运行在计算节点中，在节点内执行操作
* 通过OCI（Open Container Initiative）把CRI转换成Linux的底层调用、操作Linux Namespace和Cgroup
* 通过gRPC协议与Device Plugin插件交互，管理 GPU 、FPGA等宿主机物理设备
* 通过调用网络插件CNI（Container Networking Interface）和存储插件CSI（Container Storage Interface）为容器配置网络和存储

**kube-proxy:** 运行在计算节点的网络代理，负责Pod内外的网络通信代理



**etcd：** 键值存储数据库，保存Kubernetes整个集群相关数据





k8s operator



K8s将docker作为容器运行时的一种实现，而大规模集群任务的关系、编排和管理才是最困难的地方。



Pod里的容器共享同一个Network Namespace、同一组数据卷、达到高效率交换



K8s给Pod绑定Service服务，Service服务声明的IP等信息不变，作为Pod代理的入口，替代Pod对外暴露一个固定网络地址。

k8s负责Service 后端真正代理Pod的IP地址、端口等信息的自动更新、维护









Google Borg 论文 原文

https://static.googleusercontent.com/media/research.google.com/zh-CN//pubs/archive/43438.pdf



Google Borg 论文译文

https://blog.opskumu.com/borg.html





集群搭建



# kubernetes 部署工具



Kubernetes部署kubeadm

https://github.com/yongliangcode/kubeadm



容器里运行 kubelet，依然没有很好的解决办法，我也不推荐你用容器去部署 Kubernetes 项目



把 kubelet 直接运行在宿主机上，然后使用容器部署其他的 Kubernetes 组件



要让 kubelet 隔着容器的 Mount Namespace 和文件系统，操作宿主机的文件系统，就有点儿困难了



apt-get install kubeadm



kubelet直接运行在宿主机上



kubeadm init --config kubeadm.yaml



kubeadm init流程

kubeadm join流程



Kubeadm 为集群生成一个bootstrap token，安装了kubelet和kubeadm的节点都可以通过kubeadm join加入到集群当中。





答案是：Pod，其实是一组共享了某些资源的容器。



具体的说：Pod 里的所有容器，共享的是同一个 Network Namespace，并且可以声明共享同一个 Volume。



Pod 里的所有容器，共享的是同一个 Network Namespace，并且可以声明共享同一个 Volume





在这个 Pod 中，Infra 容器永远都是第一个被创建的容器，而其他用户定义的容器，则通过 Join Network Namespace 的方式，与 Infra 容器关联在一起



它们可以直接使用 localhost 进行通信



它们看到的网络设备跟 Infra 容器看到的完全一样





一个 Pod 只有一个 IP 地址，也就是这个 Pod 的 Network Namespace 对应的 IP 地址



Pod，实际上是在扮演传统基础设施里“虚拟机”的角色；而容器，则是这个虚拟机里运行的用户程序。





![image-20231128101623191](/Users/admin/Library/Application Support/typora-user-images/image-20231128101623191.png)

























































