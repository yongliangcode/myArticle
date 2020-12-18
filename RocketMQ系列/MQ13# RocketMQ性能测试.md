---
title: MQ13# RocketMQ性能测试
categories: RocketMQ
tags: RocketMQ
date: 2020-12-20 19:14:01
---

# 目录

```
1.概述

2.1个线程测试记录

a.1个线程 消息1K 8个队列

b.1个线程 消息3K 8个队列

c.1个线程 消息1K 16个队列

d.1个线程 消息3K 16个队列

3.10个线程测试记录

a.10个线程 消息1K 8个队列

b.10个线程 消息3K 8个队列

c.10个线程 消息1K 16个队列

d.10个线程 消息3K 16个队列

4.30个线程测试记录

a.30个线程 消息1K 8个队列

b.30个线程 消息3K 8个队列

c.30个线程 消息1K 16个队列

d.30个线程 消息3K 16个队列

5.45个线程测试记录

a.45个线程 消息1K 8个队列

b.45个线程 消息3K 8个队列

c.45个线程 消息1K 16个队列

d.45个线程 消息3K 16个队列

6.60个线程测试记录

a.60个线程 消息1K 8个队列

b.60个线程 消息3K 8个队列

c.60个线程 消息1K 16个队列

d.60个线程 消息3K 16个队列

7.75个线程测试记录

a.75个线程 消息1K 8个队列

b.75个线程 消息3K 8个队列

c.75个线程 消息1K 16个队列

d.75个线程 消息3K 16个队列

8.100个线程测试记录

a.100个线程 消息1K 8个队列

b.100个线程 消息3K 8个队列

c.100个线程 消息1K 16个队列

d.100个线程 消息3K 16个队列

9.150个线程测试记录

a.150个线程 消息1K 8个队列

b.150个线程 消息3K 8个队列

c.150个线程 消息1K 16个队列

d.150个线程 消息3K 16个队列

10.200个线程测试记录

a.200个线程 消息1K 8个队列

b.200个线程 消息3K 8个队列

c.200个线程 消息1K 16个队列

d.200个线程 消息3K 16个队列
```



# 概述



目的：对生产环境RocketMQ集群进行性能测试,该集群4主4从。

过程：线程数1、线程数10、线程数30、线程数60、线程数100、线程数150、线程数200对消息大小为1K、3K；队列为8个、16个分别进行测试。

结果：其中最大TPS为12.6万，最小TPS为3.6万。



# 1个线程测试记录

### 1个线程 消息1K 8个队列

```
sh producer.sh -t zms-clusterB-perf-tst8 -w 1 -s 1024 -n 192.168.x.x:9876

Send TPS: 4281 Max RT: 299 Average RT: 0.233 Send Failed: 0 Response Failed: 0

Send TPS: 4237 Max RT: 299 Average RT: 0.236 Send Failed: 0 Response Failed: 0

Send TPS: 4533 Max RT: 299 Average RT: 0.221 Send Failed: 0 Response Failed: 0

Send TPS: 4404 Max RT: 299 Average RT: 0.227 Send Failed: 0 Response Failed: 0

Send TPS: 4360 Max RT: 299 Average RT: 0.229 Send Failed: 0 Response Failed: 0

Send TPS: 4269 Max RT: 299 Average RT: 0.234 Send Failed: 0 Response Failed: 0

Send TPS: 4319 Max RT: 299 Average RT: 0.231 Send Failed: 0 Response Failed: 0
```



### 1个线程 消息3K 8个队列

```
sh producer.sh -t zms-clusterB-perf-tst8 -w 1 -s 3072 -n 192.168.x.x:9876

Send TPS: 4120 Max RT: 255 Average RT: 0.242 Send Failed: 0 Response Failed: 0

Send TPS: 4054 Max RT: 255 Average RT: 0.246 Send Failed: 0 Response Failed: 0

Send TPS: 4010 Max RT: 255 Average RT: 0.249 Send Failed: 0 Response Failed: 0

Send TPS: 4125 Max RT: 255 Average RT: 0.242 Send Failed: 0 Response Failed: 0

Send TPS: 4093 Max RT: 255 Average RT: 0.244 Send Failed: 0 Response Failed: 0

Send TPS: 4093 Max RT: 255 Average RT: 0.244 Send Failed: 0 Response Failed: 0

Send TPS: 3999 Max RT: 255 Average RT: 0.250 Send Failed: 0 Response Failed: 0

Send TPS: 3957 Max RT: 255 Average RT: 0.253 Send Failed: 0 Response Failed: 0
```

### 1个线程 消息1K 16个队列

```
sh producer.sh -t zms-clusterB-perf-tst16 -w 1 -s 1024 -n 192.168.x.x:9876

Send TPS: 5289 Max RT: 225 Average RT: 0.189 Send Failed: 0 Response Failed: 0

Send TPS: 5252 Max RT: 225 Average RT: 0.190 Send Failed: 0 Response Failed: 0

Send TPS: 5124 Max RT: 225 Average RT: 0.195 Send Failed: 0 Response Failed: 0

Send TPS: 5146 Max RT: 225 Average RT: 0.194 Send Failed: 0 Response Failed: 0

Send TPS: 4861 Max RT: 225 Average RT: 0.206 Send Failed: 0 Response Failed: 0

Send TPS: 4998 Max RT: 225 Average RT: 0.200 Send Failed: 0 Response Failed: 0

Send TPS: 5063 Max RT: 225 Average RT: 0.198 Send Failed: 0 Response Failed: 0

Send TPS: 5039 Max RT: 225 Average RT: 0.198 Send Failed: 0 Response Failed: 0
```

### 1个线程 消息3K 16个队列

```
sh producer.sh -t zms-clusterB-perf-tst16 -w 1 -s 3072 -n 192.168.x.x:9876

Send TPS: 4778 Max RT: 244 Average RT: 0.209 Send Failed: 0 Response Failed: 0

Send TPS: 5011 Max RT: 244 Average RT: 0.199 Send Failed: 0 Response Failed: 0

Send TPS: 4826 Max RT: 244 Average RT: 0.207 Send Failed: 0 Response Failed: 0

Send TPS: 4762 Max RT: 244 Average RT: 0.210 Send Failed: 0 Response Failed: 0

Send TPS: 4663 Max RT: 244 Average RT: 0.214 Send Failed: 0 Response Failed: 0

Send TPS: 4648 Max RT: 244 Average RT: 0.215 Send Failed: 0 Response Failed: 0

Send TPS: 4778 Max RT: 244 Average RT: 0.209 Send Failed: 0 Response Failed: 0

Send TPS: 4737 Max RT: 244 Average RT: 0.211 Send Failed: 0 Response Failed: 0

Send TPS: 4523 Max RT: 244 Average RT: 0.221 Send Failed: 0 Response Failed: 0

Send TPS: 4544 Max RT: 244 Average RT: 0.220 Send Failed: 0 Response Failed: 0

Send TPS: 4683 Max RT: 244 Average RT: 0.213 Send Failed: 0 Response Failed: 0

Send TPS: 4838 Max RT: 244 Average RT: 0.207 Send Failed: 0 Response Failed: 0
```





# 10个线程测试记录

### 10个线程 消息1K 8个队列

```
sh producer.sh -t zms-clusterB-perf-tst16 -w 1 -s 3072 -n 192.168.x.x:9876

Send TPS: 4778 Max RT: 244 Average RT: 0.209 Send Failed: 0 Response Failed: 0

Send TPS: 5011 Max RT: 244 Average RT: 0.199 Send Failed: 0 Response Failed: 0

Send TPS: 4826 Max RT: 244 Average RT: 0.207 Send Failed: 0 Response Failed: 0

Send TPS: 4762 Max RT: 244 Average RT: 0.210 Send Failed: 0 Response Failed: 0

Send TPS: 4663 Max RT: 244 Average RT: 0.214 Send Failed: 0 Response Failed: 0

Send TPS: 4648 Max RT: 244 Average RT: 0.215 Send Failed: 0 Response Failed: 0

Send TPS: 4778 Max RT: 244 Average RT: 0.209 Send Failed: 0 Response Failed: 0

Send TPS: 4737 Max RT: 244 Average RT: 0.211 Send Failed: 0 Response Failed: 0

Send TPS: 4523 Max RT: 244 Average RT: 0.221 Send Failed: 0 Response Failed: 0

Send TPS: 4544 Max RT: 244 Average RT: 0.220 Send Failed: 0 Response Failed: 0

Send TPS: 4683 Max RT: 244 Average RT: 0.213 Send Failed: 0 Response Failed: 0

Send TPS: 4838 Max RT: 244 Average RT: 0.207 Send Failed: 0 Response Failed: 0
```



### 10个线程 消息3K 8个队列

```
sh producer.sh -t zms-clusterB-perf-tst8 -w 10 -s 3072 -n 192.168.x.x:9876

Send TPS: 40085 Max RT: 265 Average RT: 0.249 Send Failed: 0 Response Failed: 0

Send TPS: 37710 Max RT: 265 Average RT: 0.265 Send Failed: 0 Response Failed: 0

Send TPS: 39305 Max RT: 265 Average RT: 0.254 Send Failed: 0 Response Failed: 0

Send TPS: 39881 Max RT: 265 Average RT: 0.251 Send Failed: 0 Response Failed: 0

Send TPS: 38428 Max RT: 265 Average RT: 0.260 Send Failed: 0 Response Failed: 0

Send TPS: 39280 Max RT: 265 Average RT: 0.255 Send Failed: 0 Response Failed: 0

Send TPS: 38539 Max RT: 265 Average RT: 0.259 Send Failed: 0 Response Failed: 0

Send TPS: 40927 Max RT: 265 Average RT: 0.244 Send Failed: 0 Response Failed: 0
```



### 10个线程 消息1K 16个队列

```
sh producer.sh -t zms-clusterB-perf-tst16 -w 10 -s 1024 -n 192.168.x.x:9876

Send TPS: 41301 Max RT: 243 Average RT: 0.242 Send Failed: 0 Response Failed: 0

Send TPS: 42365 Max RT: 243 Average RT: 0.236 Send Failed: 0 Response Failed: 0

Send TPS: 42181 Max RT: 243 Average RT: 0.237 Send Failed: 0 Response Failed: 0

Send TPS: 42261 Max RT: 243 Average RT: 0.237 Send Failed: 0 Response Failed: 0

Send TPS: 40831 Max RT: 243 Average RT: 0.245 Send Failed: 0 Response Failed: 0

Send TPS: 43010 Max RT: 243 Average RT: 0.232 Send Failed: 0 Response Failed: 0

Send TPS: 41871 Max RT: 243 Average RT: 0.239 Send Failed: 0 Response Failed: 0

Send TPS: 40970 Max RT: 243 Average RT: 0.244 Send Failed: 0 Response Failed: 0
```



### 10个线程 消息3K 16个队列

```
sh producer.sh -t zms-clusterB-perf-tst16 -w 10 -s 3072 -n 192.168.x.x:9876

Send TPS: 36245 Max RT: 237 Average RT: 0.276 Send Failed: 0 Response Failed: 0

Send TPS: 38713 Max RT: 237 Average RT: 0.258 Send Failed: 0 Response Failed: 0

Send TPS: 36327 Max RT: 237 Average RT: 0.275 Send Failed: 0 Response Failed: 0

Send TPS: 39005 Max RT: 237 Average RT: 0.256 Send Failed: 0 Response Failed: 0

Send TPS: 37926 Max RT: 237 Average RT: 0.264 Send Failed: 0 Response Failed: 0

Send TPS: 38804 Max RT: 237 Average RT: 0.258 Send Failed: 0 Response Failed: 0

Send TPS: 39976 Max RT: 237 Average RT: 0.250 Send Failed: 0 Response Failed: 0
```



# 30个线程测试记录

### 30个线程 消息1K 8个队列

```
sh producer.sh -t zms-clusterB-perf-tst8 -w 30 -s 1024 -n 192.168.x.x:9876

Send TPS: 86259 Max RT: 309 Average RT: 0.348 Send Failed: 0 Response Failed: 0

Send TPS: 85335 Max RT: 309 Average RT: 0.351 Send Failed: 0 Response Failed: 0

Send TPS: 81850 Max RT: 309 Average RT: 0.366 Send Failed: 0 Response Failed: 0

Send TPS: 87712 Max RT: 309 Average RT: 0.342 Send Failed: 0 Response Failed: 0

Send TPS: 89288 Max RT: 309 Average RT: 0.336 Send Failed: 0 Response Failed: 0

Send TPS: 86732 Max RT: 309 Average RT: 0.346 Send Failed: 0 Response Failed: 0
```



### 30个线程 消息3K 8个队列

```
sh producer.sh -t zms-clusterB-perf-tst8 -w 30 -s 3072 -n 192.168.x.x:9876

Send TPS: 74085 Max RT: 334 Average RT: 0.405 Send Failed: 0 Response Failed: 0

Send TPS: 71014 Max RT: 334 Average RT: 0.422 Send Failed: 0 Response Failed: 0

Send TPS: 77792 Max RT: 334 Average RT: 0.386 Send Failed: 0 Response Failed: 0

Send TPS: 73913 Max RT: 334 Average RT: 0.406 Send Failed: 0 Response Failed: 0

Send TPS: 77337 Max RT: 334 Average RT: 0.392 Send Failed: 0 Response Failed: 0

Send TPS: 72184 Max RT: 334 Average RT: 0.416 Send Failed: 0 Response Failed: 0

Send TPS: 77271 Max RT: 334 Average RT: 0.388 Send Failed: 0 Response Failed: 0

Send TPS: 75016 Max RT: 334 Average RT: 0.400 Send Failed: 0 Response Failed: 0
```



### 30个线程 消息1K 16个队列

```
sh producer.sh -t zms-clusterB-perf-tst16 -w 30 -s 1024 -n 192.168.x.x:9876

Send TPS: 82946 Max RT: 306 Average RT: 0.362 Send Failed: 0 Response Failed: 0

Send TPS: 86902 Max RT: 306 Average RT: 0.345 Send Failed: 0 Response Failed: 0

Send TPS: 83157 Max RT: 306 Average RT: 0.365 Send Failed: 0 Response Failed: 0

Send TPS: 86804 Max RT: 306 Average RT: 0.345 Send Failed: 0 Response Failed: 0

Send TPS: 87009 Max RT: 306 Average RT: 0.345 Send Failed: 0 Response Failed: 0

Send TPS: 80219 Max RT: 306 Average RT: 0.374 Send Failed: 0 Response Failed: 0
```



### 30个线程 消息3K 16个队列

```
sh producer.sh -t zms-clusterB-perf-tst16 -w 30 -s 3072 -n 192.168.x.x:9876

Send TPS: 73864 Max RT: 329 Average RT: 0.403 Send Failed: 0 Response Failed: 0

Send TPS: 78555 Max RT: 329 Average RT: 0.382 Send Failed: 0 Response Failed: 0

Send TPS: 75200 Max RT: 329 Average RT: 0.406 Send Failed: 0 Response Failed: 0

Send TPS: 73925 Max RT: 329 Average RT: 0.406 Send Failed: 0 Response Failed: 0

Send TPS: 69955 Max RT: 329 Average RT: 0.429 Send Failed: 0 Response Failed: 0
```



# 45个线程测试记录

### 45个线程 消息1K 8个队列

```
sh producer.sh -t zms-clusterB-perf-tst8 -w 45 -s 1024 -n 192.168.x.x:9876

Send TPS: 91266 Max RT: 2063 Average RT: 0.493 Send Failed: 0 Response Failed: 0

Send TPS: 87279 Max RT: 2063 Average RT: 0.515 Send Failed: 0 Response Failed: 0

Send TPS: 92130 Max RT: 2063 Average RT: 0.487 Send Failed: 0 Response Failed: 1

Send TPS: 95227 Max RT: 2063 Average RT: 0.472 Send Failed: 0 Response Failed: 1

Send TPS: 96340 Max RT: 2063 Average RT: 0.467 Send Failed: 0 Response Failed: 1

Send TPS: 84272 Max RT: 2063 Average RT: 0.534 Send Failed: 0 Response Failed: 1
```



### 45个线程 消息3K 8个队列

```
sh producer.sh -t zms-clusterB-perf-tst8 -w 45 -s 3072 -n 192.168.x.x:9876

Send TPS: 89334 Max RT: 462 Average RT: 0.503 Send Failed: 0 Response Failed: 0

Send TPS: 84237 Max RT: 462 Average RT: 0.534 Send Failed: 0 Response Failed: 0

Send TPS: 86051 Max RT: 462 Average RT: 0.523 Send Failed: 0 Response Failed: 0

Send TPS: 86475 Max RT: 462 Average RT: 0.520 Send Failed: 0 Response Failed: 0

Send TPS: 86088 Max RT: 462 Average RT: 0.523 Send Failed: 0 Response Failed: 0

Send TPS: 90403 Max RT: 462 Average RT: 0.498 Send Failed: 0 Response Failed: 0

Send TPS: 84229 Max RT: 462 Average RT: 0.534 Send Failed: 0 Response Failed: 0
```



### 45个线程 消息1K 16个队列

```
sh producer.sh -t zms-clusterB-perf-tst16 -w 45 -s 1024 -n 192.168.x.:9876

Send TPS: 91724 Max RT: 604 Average RT: 0.490 Send Failed: 0 Response Failed: 0

Send TPS: 90414 Max RT: 604 Average RT: 0.498 Send Failed: 0 Response Failed: 0

Send TPS: 89904 Max RT: 604 Average RT: 0.500 Send Failed: 0 Response Failed: 0

Send TPS: 100158 Max RT: 604 Average RT: 0.449 Send Failed: 0 Response Failed: 0

Send TPS: 99658 Max RT: 604 Average RT: 0.451 Send Failed: 0 Response Failed: 0

Send TPS: 92440 Max RT: 604 Average RT: 0.489 Send Failed: 0 Response Failed: 0
```



### 45个线程 消息3K 16个队列

```
sh producer.sh -t zms-clusterB-perf-tst16 -w 30 -s 3072 -n 192.168.x.x:9876

Send TPS: 75159 Max RT: 436 Average RT: 0.399 Send Failed: 0 Response Failed: 0

Send TPS: 75315 Max RT: 436 Average RT: 0.398 Send Failed: 0 Response Failed: 0

Send TPS: 77297 Max RT: 436 Average RT: 0.388 Send Failed: 0 Response Failed: 0

Send TPS: 72188 Max RT: 436 Average RT: 0.415 Send Failed: 0 Response Failed: 0

Send TPS: 77525 Max RT: 436 Average RT: 0.387 Send Failed: 0 Response Failed: 0

Send TPS: 71535 Max RT: 436 Average RT: 0.422 Send Failed: 0 Response Failed: 0
```



<!--more-->



# 60个线程测试记录

### 60个线程 消息1K 8个队列

```
sh producer.sh -t zms-clusterB-perf-tst8 -w 60 -s 1024 -n 192.168.x.x:9876

Send TPS: 110067 Max RT: 369 Average RT: 0.545 Send Failed: 0 Response Failed: 0

Send TPS: 111395 Max RT: 369 Average RT: 0.538 Send Failed: 0 Response Failed: 0

Send TPS: 103114 Max RT: 369 Average RT: 0.582 Send Failed: 0 Response Failed: 0

Send TPS: 107466 Max RT: 369 Average RT: 0.558 Send Failed: 0 Response Failed: 0

Send TPS: 106655 Max RT: 369 Average RT: 0.562 Send Failed: 0 Response Failed: 0

Send TPS: 107241 Max RT: 369 Average RT: 0.559 Send Failed: 0 Response Failed: 1

Send TPS: 110672 Max RT: 369 Average RT: 0.540 Send Failed: 0 Response Failed: 1

Send TPS: 109037 Max RT: 369 Average RT: 0.552 Send Failed: 0 Response Failed: 1
```



### 60个线程 消息3K 8个队列

```
sh producer.sh -t zms-clusterB-perf-tst8 -w 60 -s 3072 -n 192.168.x.x:9876

Send TPS: 92572 Max RT: 583 Average RT: 0.648 Send Failed: 0 Response Failed: 0

Send TPS: 95163 Max RT: 583 Average RT: 0.640 Send Failed: 0 Response Failed: 1

Send TPS: 93823 Max RT: 583 Average RT: 0.654 Send Failed: 0 Response Failed: 1

Send TPS: 97091 Max RT: 583 Average RT: 0.628 Send Failed: 0 Response Failed: 1

Send TPS: 98205 Max RT: 583 Average RT: 0.628 Send Failed: 0 Response Failed: 1

Send TPS: 99535 Max RT: 583 Average RT: 0.596 Send Failed: 0 Response Failed: 3
```



### 60个线程 消息1K 16个队列

```
sh producer.sh -t zms-clusterB-perf-tst16 -w 60 -s 1024 -n 192.168.x.x:9876

Send TPS: 105229 Max RT: 358 Average RT: 0.578 Send Failed: 0 Response Failed: 0

Send TPS: 103003 Max RT: 358 Average RT: 0.582 Send Failed: 0 Response Failed: 0

Send TPS: 95497 Max RT: 358 Average RT: 0.628 Send Failed: 0 Response Failed: 0

Send TPS: 108878 Max RT: 358 Average RT: 0.551 Send Failed: 0 Response Failed: 0

Send TPS: 109265 Max RT: 358 Average RT: 0.549 Send Failed: 0 Response Failed: 0

Send TPS: 105545 Max RT: 358 Average RT: 0.568 Send Failed: 0 Response Failed: 0

Send TPS: 111667 Max RT: 358 Average RT: 0.537 Send Failed: 0 Response Failed: 0
```



### 60个线程 消息3K 16个队列

```
sh producer.sh -t zms-clusterB-perf-tst16 -w 60 -s 3072 -n 192.168.x.x:9876

Send TPS: 98899 Max RT: 358 Average RT: 0.606 Send Failed: 0 Response Failed: 0

Send TPS: 101073 Max RT: 358 Average RT: 0.594 Send Failed: 0 Response Failed: 0

Send TPS: 97295 Max RT: 358 Average RT: 0.617 Send Failed: 0 Response Failed: 0

Send TPS: 97923 Max RT: 358 Average RT: 0.609 Send Failed: 0 Response Failed: 1

Send TPS: 96111 Max RT: 358 Average RT: 0.620 Send Failed: 0 Response Failed: 2

Send TPS: 93873 Max RT: 358 Average RT: 0.639 Send Failed: 0 Response Failed: 2

Send TPS: 96466 Max RT: 358 Average RT: 0.622 Send Failed: 0 Response Failed: 2

Send TPS: 96579 Max RT: 358 Average RT: 0.621 Send Failed: 0 Response Failed: 2
```



# 75个线程测试记录

### 75个线程 消息1K 8个队列

```
sh producer.sh -t zms-clusterB-perf-tst8 -w 75 -s 1024 -n 192.168.x.x:9876

Send TPS: 108367 Max RT: 384 Average RT: 0.692 Send Failed: 0 Response Failed: 0

Send TPS: 107516 Max RT: 384 Average RT: 0.701 Send Failed: 0 Response Failed: 0

Send TPS: 110974 Max RT: 384 Average RT: 0.680 Send Failed: 0 Response Failed: 0

Send TPS: 109754 Max RT: 384 Average RT: 0.683 Send Failed: 0 Response Failed: 0

Send TPS: 111917 Max RT: 384 Average RT: 0.670 Send Failed: 0 Response Failed: 0

Send TPS: 104764 Max RT: 384 Average RT: 0.712 Send Failed: 0 Response Failed: 1

Send TPS: 112208 Max RT: 384 Average RT: 0.668 Send Failed: 0 Response Failed: 1

Send TPS: 112707 Max RT: 384 Average RT: 0.665 Send Failed: 0 Response Failed: 1
```



### 75个线程 消息3K 8个队列

```
sh producer.sh -t zms-clusterB-perf-tst8 -w 75 -s 3072 -n 192.168.x.x:9876

Send TPS: 102311 Max RT: 370 Average RT: 0.733 Send Failed: 0 Response Failed: 0

Send TPS: 93722 Max RT: 370 Average RT: 0.800 Send Failed: 0 Response Failed: 0

Send TPS: 101091 Max RT: 370 Average RT: 0.742 Send Failed: 0 Response Failed: 0

Send TPS: 100404 Max RT: 370 Average RT: 0.747 Send Failed: 0 Response Failed: 0

Send TPS: 102328 Max RT: 370 Average RT: 0.733 Send Failed: 0 Response Failed: 0

Send TPS: 103953 Max RT: 370 Average RT: 0.722 Send Failed: 0 Response Failed: 0

Send TPS: 103454 Max RT: 370 Average RT: 0.725 Send Failed: 0 Response Failed: 0
```



### 75个线程 消息1K 16个队列

```
sh producer.sh -t zms-clusterB-perf-tst16 -w 75 -s 1024 -n 192.168.x.x:9876

Send TPS: 106813 Max RT: 605 Average RT: 0.687 Send Failed: 0 Response Failed: 0

Send TPS: 110828 Max RT: 605 Average RT: 0.673 Send Failed: 0 Response Failed: 1

Send TPS: 109855 Max RT: 605 Average RT: 0.676 Send Failed: 0 Response Failed: 3

Send TPS: 102741 Max RT: 605 Average RT: 0.730 Send Failed: 0 Response Failed: 3

Send TPS: 110123 Max RT: 605 Average RT: 0.681 Send Failed: 0 Response Failed: 3

Send TPS: 115659 Max RT: 605 Average RT: 0.648 Send Failed: 0 Response Failed: 3

Send TPS: 108157 Max RT: 605 Average RT: 0.693 Send Failed: 0 Response Failed: 3
```



### 75个线程 消息3K 16个队列

```
sh producer.sh -t zms-clusterB-perf-tst16 -w 75 -s 3072 -n 192.168.x.x:9876

Send TPS: 90459 Max RT: 499 Average RT: 0.829 Send Failed: 0 Response Failed: 0

Send TPS: 96838 Max RT: 499 Average RT: 0.770 Send Failed: 0 Response Failed: 1

Send TPS: 96590 Max RT: 499 Average RT: 0.776 Send Failed: 0 Response Failed: 1

Send TPS: 95137 Max RT: 499 Average RT: 0.788 Send Failed: 0 Response Failed: 1

Send TPS: 89502 Max RT: 499 Average RT: 0.834 Send Failed: 0 Response Failed: 2

Send TPS: 90255 Max RT: 499 Average RT: 0.831 Send Failed: 0 Response Failed: 2

Send TPS: 99871 Max RT: 499 Average RT: 0.725 Send Failed: 0 Response Failed: 9
```



# 100个线程测试记录

### 100个线程 消息1K 8个队列

```
sh producer.sh -t zms-clusterB-perf-tst8 -w 100 -s 1024 -n 192.168.x.x:9876

Send TPS: 113204 Max RT: 402 Average RT: 0.883 Send Failed: 0 Response Failed: 0

Send TPS: 114872 Max RT: 402 Average RT: 0.868 Send Failed: 0 Response Failed: 1

Send TPS: 116261 Max RT: 402 Average RT: 0.860 Send Failed: 0 Response Failed: 1

Send TPS: 118116 Max RT: 402 Average RT: 0.847 Send Failed: 0 Response Failed: 1

Send TPS: 112594 Max RT: 402 Average RT: 0.888 Send Failed: 0 Response Failed: 1

Send TPS: 124407 Max RT: 402 Average RT: 0.801 Send Failed: 0 Response Failed: 2

Send TPS: 126590 Max RT: 402 Average RT: 0.790 Send Failed: 0 Response Failed: 2
```



### 100个线程 消息3K 8个队列

```
sh producer.sh -t zms-clusterB-perf-tst8 -w 100 -s 3072 -n 192.168.x.x:9876

Send TPS: 106723 Max RT: 426 Average RT: 0.937 Send Failed: 0 Response Failed: 0

Send TPS: 104768 Max RT: 426 Average RT: 0.943 Send Failed: 0 Response Failed: 1

Send TPS: 106697 Max RT: 426 Average RT: 0.935 Send Failed: 0 Response Failed: 2

Send TPS: 105147 Max RT: 426 Average RT: 0.951 Send Failed: 0 Response Failed: 2

Send TPS: 105814 Max RT: 426 Average RT: 0.935 Send Failed: 0 Response Failed: 5

Send TPS: 108616 Max RT: 426 Average RT: 0.916 Send Failed: 0 Response Failed: 6

Send TPS: 101429 Max RT: 426 Average RT: 0.986 Send Failed: 0 Response Failed: 6
```



### 100个线程 消息1K 16个队列

```
sh producer.sh -t zms-clusterB-perf-tst16 -w 100 -s 1024 -n 192.168.x.x:9876

Send TPS: 123424 Max RT: 438 Average RT: 0.805 Send Failed: 0 Response Failed: 0

Send TPS: 111418 Max RT: 438 Average RT: 0.897 Send Failed: 0 Response Failed: 0

Send TPS: 110360 Max RT: 438 Average RT: 0.905 Send Failed: 0 Response Failed: 0

Send TPS: 118734 Max RT: 438 Average RT: 0.842 Send Failed: 0 Response Failed: 0

Send TPS: 120725 Max RT: 438 Average RT: 0.816 Send Failed: 0 Response Failed: 4

Send TPS: 113823 Max RT: 438 Average RT: 0.878 Send Failed: 0 Response Failed: 4

Send TPS: 115639 Max RT: 438 Average RT: 0.865 Send Failed: 0 Response Failed: 4

Send TPS: 112787 Max RT: 438 Average RT: 0.889 Send Failed: 0 Response Failed: 4

Send TPS: 106677 Max RT: 438 Average RT: 0.937 Send Failed: 0 Response Failed: 4

Send TPS: 112635 Max RT: 438 Average RT: 0.888 Send Failed: 0 Response Failed: 4

Send TPS: 108470 Max RT: 438 Average RT: 0.922 Send Failed: 0 Response Failed: 4
```



### 100个线程 消息3K 16个队列

```
sh producer.sh -t zms-clusterB-perf-tst16 -w 100 -s 3072 -n 192.168.x.x:9876

Send TPS: 93374 Max RT: 441 Average RT: 1.071 Send Failed: 0 Response Failed: 3

Send TPS: 98421 Max RT: 441 Average RT: 1.017 Send Failed: 0 Response Failed: 3

Send TPS: 103664 Max RT: 441 Average RT: 0.964 Send Failed: 0 Response Failed: 4

Send TPS: 98234 Max RT: 441 Average RT: 0.995 Send Failed: 0 Response Failed: 6

Send TPS: 103563 Max RT: 441 Average RT: 0.960 Send Failed: 0 Response Failed: 7

Send TPS: 103807 Max RT: 441 Average RT: 0.962 Send Failed: 0 Response Failed: 7

Send TPS: 102715 Max RT: 441 Average RT: 0.973 Send Failed: 0 Response Failed: 7
```



# 150个线程测试记录

### 150个线程 消息1K 8个队列

```
sh producer.sh -t zms-clusterB-perf-tst8 -w 150 -s 1024 -n 192.168.x.x:9876

Send TPS: 124458 Max RT: 633 Average RT: 1.205 Send Failed: 0 Response Failed: 0

Send TPS: 124567 Max RT: 633 Average RT: 1.204 Send Failed: 0 Response Failed: 0

Send TPS: 121324 Max RT: 633 Average RT: 1.236 Send Failed: 0 Response Failed: 0

Send TPS: 124928 Max RT: 633 Average RT: 1.201 Send Failed: 0 Response Failed: 0

Send TPS: 122830 Max RT: 633 Average RT: 1.242 Send Failed: 0 Response Failed: 0

Send TPS: 118825 Max RT: 633 Average RT: 1.262 Send Failed: 0 Response Failed: 0

Send TPS: 124085 Max RT: 633 Average RT: 1.209 Send Failed: 0 Response Failed: 0
```



### 150个线程 消息3K 8个队列

```
sh producer.sh -t zms-clusterB-perf-tst8 -w 150 -s 3072 -n 192.168.x.x:9876

Send TPS: 106575 Max RT: 582 Average RT: 1.404 Send Failed: 0 Response Failed: 1

Send TPS: 101830 Max RT: 582 Average RT: 1.477 Send Failed: 0 Response Failed: 1

Send TPS: 99666 Max RT: 582 Average RT: 1.505 Send Failed: 0 Response Failed: 1

Send TPS: 102139 Max RT: 582 Average RT: 1.465 Send Failed: 0 Response Failed: 2

Send TPS: 105405 Max RT: 582 Average RT: 1.419 Send Failed: 0 Response Failed: 3

Send TPS: 107032 Max RT: 582 Average RT: 1.399 Send Failed: 0 Response Failed: 4

Send TPS: 103416 Max RT: 582 Average RT: 1.448 Send Failed: 0 Response Failed: 5
```



### 150个线程 消息1K 16个队列

```
sh producer.sh -t zms-clusterB-perf-tst16 -w 150 -s 1024 -n 192.168.x.x:9876

Send TPS: 115151 Max RT: 574 Average RT: 1.299 Send Failed: 0 Response Failed: 1

Send TPS: 106960 Max RT: 574 Average RT: 1.402 Send Failed: 0 Response Failed: 1

Send TPS: 116382 Max RT: 574 Average RT: 1.289 Send Failed: 0 Response Failed: 1

Send TPS: 110587 Max RT: 574 Average RT: 1.349 Send Failed: 0 Response Failed: 4

Send TPS: 122832 Max RT: 574 Average RT: 1.220 Send Failed: 0 Response Failed: 4

Send TPS: 124474 Max RT: 574 Average RT: 1.213 Send Failed: 0 Response Failed: 4

Send TPS: 112153 Max RT: 574 Average RT: 1.337 Send Failed: 0 Response Failed: 4

Send TPS: 120450 Max RT: 574 Average RT: 1.261 Send Failed: 0 Response Failed: 4
```



### 150个线程 消息3K 16个队列

```
sh producer.sh -t zms-clusterB-perf-tst16 -w 150 -s 3072 -n 192.168.x.x:9876

Send TPS: 105061 Max RT: 535 Average RT: 1.428 Send Failed: 0 Response Failed: 0

Send TPS: 102117 Max RT: 535 Average RT: 1.465 Send Failed: 0 Response Failed: 1

Send TPS: 105569 Max RT: 535 Average RT: 1.421 Send Failed: 0 Response Failed: 1

Send TPS: 100689 Max RT: 535 Average RT: 1.489 Send Failed: 0 Response Failed: 2

Send TPS: 108464 Max RT: 535 Average RT: 1.381 Send Failed: 0 Response Failed: 2

Send TPS: 111285 Max RT: 535 Average RT: 1.348 Send Failed: 0 Response Failed: 2

Send TPS: 103406 Max RT: 535 Average RT: 1.451 Send Failed: 0 Response Failed: 2

Send TPS: 109203 Max RT: 535 Average RT: 1.388 Send Failed: 0 Response Failed: 2
```



# 200个线程测试记录

### 200个线程 消息1K 8个队列

```
sh producer.sh -t zms-clusterB-perf-tst8 -w 200 -s 1024 -n 192.168.x.x:9876

Send TPS: 117965 Max RT: 628 Average RT: 1.674 Send Failed: 0 Response Failed: 7

Send TPS: 115583 Max RT: 628 Average RT: 1.715 Send Failed: 0 Response Failed: 12

Send TPS: 118732 Max RT: 628 Average RT: 1.672 Send Failed: 0 Response Failed: 16

Send TPS: 126170 Max RT: 628 Average RT: 1.582 Send Failed: 0 Response Failed: 17

Send TPS: 116203 Max RT: 628 Average RT: 1.719 Send Failed: 0 Response Failed: 18

Send TPS: 114793 Max RT: 628 Average RT: 1.739 Send Failed: 0 Response Failed: 19
```



### 200个线程 消息3K 8个队列

```
sh producer.sh -t zms-clusterB-perf-tst8 -w 200 -s 3072 -n 192.168.x.x:9876

Send TPS: 107240 Max RT: 761 Average RT: 1.865 Send Failed: 0 Response Failed: 0

Send TPS: 104585 Max RT: 761 Average RT: 1.906 Send Failed: 0 Response Failed: 2

Send TPS: 110892 Max RT: 761 Average RT: 1.803 Send Failed: 0 Response Failed: 2

Send TPS: 105414 Max RT: 761 Average RT: 1.898 Send Failed: 0 Response Failed: 2

Send TPS: 105904 Max RT: 761 Average RT: 1.885 Send Failed: 0 Response Failed: 3

Send TPS: 110748 Max RT: 761 Average RT: 1.806 Send Failed: 0 Response Failed: 3
```



### 200个线程 消息1K 16个队列

```
sh producer.sh -t zms-clusterB-perf-tst16 -w 200 -s 1024 -n 192.168.x.x:9876

Send TPS: 118892 Max RT: 601 Average RT: 1.679 Send Failed: 0 Response Failed: 4

Send TPS: 118839 Max RT: 601 Average RT: 1.668 Send Failed: 0 Response Failed: 12

Send TPS: 117122 Max RT: 601 Average RT: 1.704 Send Failed: 0 Response Failed: 12

Send TPS: 122670 Max RT: 601 Average RT: 1.630 Send Failed: 0 Response Failed: 12

Send TPS: 119592 Max RT: 601 Average RT: 1.672 Send Failed: 0 Response Failed: 12

Send TPS: 121243 Max RT: 601 Average RT: 1.649 Send Failed: 0 Response Failed: 12

Send TPS: 124760 Max RT: 601 Average RT: 1.603 Send Failed: 0 Response Failed: 12

Send TPS: 124354 Max RT: 601 Average RT: 1.608 Send Failed: 0 Response Failed: 12

Send TPS: 119272 Max RT: 601 Average RT: 1.677 Send Failed: 0 Response Failed: 12
```



### 200个线程 消息3K 16个队列

```
sh producer.sh -t zms-clusterB-perf-tst16 -w 200 -s 3072 -n 192.168.x.x:9876

Send TPS: 105091 Max RT: 963 Average RT: 1.896 Send Failed: 0 Response Failed: 4

Send TPS: 106243 Max RT: 963 Average RT: 1.882 Send Failed: 0 Response Failed: 4

Send TPS: 103994 Max RT: 963 Average RT: 1.958 Send Failed: 0 Response Failed: 5

Send TPS: 109741 Max RT: 963 Average RT: 1.822 Send Failed: 0 Response Failed: 5

Send TPS: 103788 Max RT: 963 Average RT: 1.927 Send Failed: 0 Response Failed: 5

Send TPS: 110597 Max RT: 963 Average RT: 1.805 Send Failed: 0 Response Failed: 6

Send TPS: 111201 Max RT: 963 Average RT: 1.798 Send Failed: 0 Response Failed: 6
```

