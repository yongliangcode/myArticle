---
title: MQ4# 查看RocketMQ Tps命令
categories: RocketMQ
tags: RocketMQ
date: 2020-12-17 11:13:01
---

# 查看所有Topic吞吐Tps

```
bin/mqadmin statsAll -n localhost:9876

Topic Consumer Group InTPS OutTPS InMsg24Hour OutMsg24Hour

T_SCANRECORD_NEW_groy 0.00 0 NO_CONSUMER

T_SCANRECORD_NEW internationalScanRecordAll 5480.10 0.00 310165682 0

T_SCANRECORD_NEW realnameConsumer 5480.10 79.98 310165682 29896917

T_SCANRECORD_NEW AdpMqCluster_consumer2 5480.10 0.00 310165682 0

```





# 查看特定Topic的吞吐Tps

```
bin/mqadmin statsAll -t SCANRECORD -n 192.168.1.x:9876

Java HotSpot(TM) 64-Bit Server VM warning: ignoring option PermSize=128m; support was removed in 8.0

Java HotSpot(TM) 64-Bit Server VM warning: ignoring option MaxPermSize=128m; support was removed in 8.0

Topic Consumer Group Accumulation InTPS OutTPS InMsg24Hour OutMsg24Hour

SCANRECORD ZtoSignGroup 5826 2086.33 44.00 246409321 29988099

SCANRECORD newOpenPartnerDeadlineJob 5447 2086.33 44.00 246409321 29988099

SCANRECORD smartidivision-scanrecord-dis 3523 2086.33 14.32 246409321 31212557
```