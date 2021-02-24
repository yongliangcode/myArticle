---
title: Kafka2# 堆内存不能正常回收问题分析（0.11.0.2版本）
categories: Kafka
tags: Kafka
date: 2019-05-16 11:55:01
---



# 问题描述

短信报警堆内存GC后依然超过4G内存，跟上篇文章所说情况相同。只是上次情况告警短信没发出来。这次介入前，dump了该节点的堆照，方便定位引起的问题。

告警GC日志，回收后依然在4G内存，回收前后只减少了几百M。



```
2019-05-26T20:41:52.086+0800: 12768164.084: [GC pause (G1 Evacuation Pause) (young), 0.0296753 secs]
   [Parallel Time: 27.0 ms, GC Workers: 28]
      [GC Worker Start (ms): Min: 12768164084.0, Avg: 12768164084.2, Max: 12768164084.5, Diff: 0.5]
      [Ext Root Scanning (ms): Min: 18.1, Avg: 19.0, Max: 19.9, Diff: 1.8, Sum: 532.0]
      [Update RS (ms): Min: 1.2, Avg: 1.7, Max: 2.3, Diff: 1.1, Sum: 47.6]
         [Processed Buffers: Min: 1, Avg: 2.4, Max: 8, Diff: 7, Sum: 68]
      [Scan RS (ms): Min: 1.2, Avg: 1.8, Max: 2.1, Diff: 0.9, Sum: 49.6]
      [Code Root Scanning (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.0]
      [Object Copy (ms): Min: 3.1, Avg: 3.9, Max: 4.6, Diff: 1.4, Sum: 110.6]
      [Termination (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.4]
         [Termination Attempts: Min: 1, Avg: 1.0, Max: 1, Diff: 0, Sum: 28]
      [GC Worker Other (ms): Min: 0.0, Avg: 0.1, Max: 0.1, Diff: 0.1, Sum: 1.6]
      [GC Worker Total (ms): Min: 26.2, Avg: 26.5, Max: 26.7, Diff: 0.5, Sum: 742.2]
      [GC Worker End (ms): Min: 12768164110.7, Avg: 12768164110.7, Max: 12768164110.8, Diff: 0.1]
   [Code Root Fixup: 0.1 ms]
   [Code Root Purge: 0.0 ms]
   [Clear CT: 0.7 ms]
   [Other: 1.9 ms]
      [Choose CSet: 0.0 ms]
      [Ref Proc: 0.6 ms]
      [Ref Enq: 0.0 ms]
      [Redirty Cards: 0.3 ms]
      [Humongous Register: 0.1 ms]
      [Humongous Reclaim: 0.0 ms]
      [Free CSet: 0.4 ms]
   [Eden: 400.0M(400.0M)->0.0B(400.0M) Survivors: 8192.0K->8192.0K Heap: 4426.3M(8192.0M)->4024.2M(8192.0M)]
 [Times: user=0.75 sys=0.00, real=0.03 secs]
```



<!--more-->



# 堆内存分析

有个对象的内存占用高达2.5G

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210224100318.png)



通过图示可以看到该类：

org.apache.kafka.common.metrics.JmxReporter有个Map，持有高达近330万个的子Map对象。这些子Map中的结构都类似，只是clientId数值不同。

问题：为何消费者注册到该Reporter不删除呢？

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210224100448.png)



# 代码追踪



**JmxReport类分析** 

下面贴出JmxReporter完整的类，成员变量

```
private final Map<String, KafkaMbean> mbeans = new HashMap<String, KafkaMbean>();即该mbeans持有330万个子对象。

KafkaMbean中的两个成员变量：

private final ObjectName objectName;

private final Map<String, KafkaMetric> metrics;

metrics中的key即为：堆分析中的kafka.server:type=Request,client-id=admin-3685211
```



```
package org.apache.kafka.common.metrics;

import java.lang.management.ManagementFactory;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

import javax.management.Attribute;
import javax.management.AttributeList;
import javax.management.AttributeNotFoundException;
import javax.management.DynamicMBean;
import javax.management.InvalidAttributeValueException;
import javax.management.JMException;
import javax.management.MBeanAttributeInfo;
import javax.management.MBeanException;
import javax.management.MBeanInfo;
import javax.management.MBeanServer;
import javax.management.MalformedObjectNameException;
import javax.management.ObjectName;
import javax.management.ReflectionException;

import org.apache.kafka.common.KafkaException;
import org.apache.kafka.common.MetricName;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

/**
 * Register metrics in JMX as dynamic mbeans based on the metric names
 */
public class JmxReporter implements MetricsReporter {

    private static final Logger log = LoggerFactory.getLogger(JmxReporter.class);
    private static final Object LOCK = new Object();
    private String prefix;
    private final Map<String, KafkaMbean> mbeans = new HashMap<String, KafkaMbean>();

    public JmxReporter() {
        this("");
    }

    /**
     * Create a JMX reporter that prefixes all metrics with the given string.
     */
    public JmxReporter(String prefix) {
        this.prefix = prefix;
    }

    @Override
    public void configure(Map<String, ?> configs) {}

    @Override
    public void init(List<KafkaMetric> metrics) {
        synchronized (LOCK) {
            for (KafkaMetric metric : metrics)
                addAttribute(metric);
            for (KafkaMbean mbean : mbeans.values())
                reregister(mbean);
        }
    }

    boolean containsMbean(String mbeanName) {
        return mbeans.containsKey(mbeanName);
    }
    @Override
    public void metricChange(KafkaMetric metric) {
        synchronized (LOCK) {
            KafkaMbean mbean = addAttribute(metric);
            reregister(mbean);
        }
    }

    @Override
    public void metricRemoval(KafkaMetric metric) {
        synchronized (LOCK) {
            MetricName metricName = metric.metricName();
            String mBeanName = getMBeanName(prefix, metricName);
            KafkaMbean mbean = removeAttribute(metric, mBeanName);
            if (mbean != null) {
                if (mbean.metrics.isEmpty()) {
                    unregister(mbean);
                    mbeans.remove(mBeanName);
                } else
                    reregister(mbean);
            }
        }
    }

    private KafkaMbean removeAttribute(KafkaMetric metric, String mBeanName) {
        MetricName metricName = metric.metricName();
        KafkaMbean mbean = this.mbeans.get(mBeanName);
        if (mbean != null)
            mbean.removeAttribute(metricName.name());
        return mbean;
    }

    private KafkaMbean addAttribute(KafkaMetric metric) {
        try {
            MetricName metricName = metric.metricName();
            String mBeanName = getMBeanName(prefix, metricName);
            if (!this.mbeans.containsKey(mBeanName))
                mbeans.put(mBeanName, new KafkaMbean(mBeanName));
            KafkaMbean mbean = this.mbeans.get(mBeanName);
            mbean.setAttribute(metricName.name(), metric);
            return mbean;
        } catch (JMException e) {
            throw new KafkaException("Error creating mbean attribute for metricName :" + metric.metricName(), e);
        }
    }

    /**
     * @param metricName
     * @return standard JMX MBean name in the following format domainName:type=metricType,key1=val1,key2=val2
     */
    static String getMBeanName(String prefix, MetricName metricName) {
        StringBuilder mBeanName = new StringBuilder();
        mBeanName.append(prefix);
        mBeanName.append(":type=");
        mBeanName.append(metricName.group());
        for (Map.Entry<String, String> entry : metricName.tags().entrySet()) {
            if (entry.getKey().length() <= 0 || entry.getValue().length() <= 0)
                continue;
            mBeanName.append(",");
            mBeanName.append(entry.getKey());
            mBeanName.append("=");
            mBeanName.append(entry.getValue());
        }
        return mBeanName.toString();
    }

    public void close() {
        synchronized (LOCK) {
            for (KafkaMbean mbean : this.mbeans.values())
                unregister(mbean);
        }
    }

    private void unregister(KafkaMbean mbean) {
        MBeanServer server = ManagementFactory.getPlatformMBeanServer();
        try {
            if (server.isRegistered(mbean.name()))
                server.unregisterMBean(mbean.name());
        } catch (JMException e) {
            throw new KafkaException("Error unregistering mbean", e);
        }
    }

    private void reregister(KafkaMbean mbean) {
        unregister(mbean);
        try {
            ManagementFactory.getPlatformMBeanServer().registerMBean(mbean, mbean.name());
        } catch (JMException e) {
            throw new KafkaException("Error registering mbean " + mbean.name(), e);
        }
    }

    private static class KafkaMbean implements DynamicMBean {
        private final ObjectName objectName;
        private final Map<String, KafkaMetric> metrics;

        public KafkaMbean(String mbeanName) throws MalformedObjectNameException {
            this.metrics = new HashMap<String, KafkaMetric>();
            this.objectName = new ObjectName(mbeanName);
        }

        public ObjectName name() {
            return objectName;
        }

        public void setAttribute(String name, KafkaMetric metric) {
            this.metrics.put(name, metric);
        }

        @Override
        public Object getAttribute(String name) throws AttributeNotFoundException, MBeanException, ReflectionException {
            if (this.metrics.containsKey(name))
                return this.metrics.get(name).value();
            else
                throw new AttributeNotFoundException("Could not find attribute " + name);
        }

        @Override
        public AttributeList getAttributes(String[] names) {
            try {
                AttributeList list = new AttributeList();
                for (String name : names)
                    list.add(new Attribute(name, getAttribute(name)));
                return list;
            } catch (Exception e) {
                log.error("Error getting JMX attribute: ", e);
                return new AttributeList();
            }
        }

        public KafkaMetric removeAttribute(String name) {
            return this.metrics.remove(name);
        }

        @Override
        public MBeanInfo getMBeanInfo() {
            MBeanAttributeInfo[] attrs = new MBeanAttributeInfo[metrics.size()];
            int i = 0;
            for (Map.Entry<String, KafkaMetric> entry : this.metrics.entrySet()) {
                String attribute = entry.getKey();
                KafkaMetric metric = entry.getValue();
                attrs[i] = new MBeanAttributeInfo(attribute,
                                                  double.class.getName(),
                                                  metric.metricName().description(),
                                                  true,
                                                  false,
                                                  false);
                i += 1;
            }
            return new MBeanInfo(this.getClass().getName(), "", attrs, null, null, null);
        }

        @Override
        public Object invoke(String name, Object[] params, String[] sig) throws MBeanException, ReflectionException {
            throw new UnsupportedOperationException("Set not allowed.");
        }

        @Override
        public void setAttribute(Attribute attribute) throws AttributeNotFoundException,
                                                     InvalidAttributeValueException,
                                                     MBeanException,
                                                     ReflectionException {
            throw new UnsupportedOperationException("Set not allowed.");
        }

        @Override
        public AttributeList setAttributes(AttributeList list) {
            throw new UnsupportedOperationException("Set not allowed.");
        }

    }

}

```



**Jconsole查看** 

kafka.server:type=Request

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210224100710.png)





# 问题解决

刚开始觉得是我们使用的问题，是否资源没有关闭，查看源代码也未能看出哪里出了问题。
后来确定为kafka 0.11.0.2版本的Bug，在0.11.0.3版本已经修复。



ISSUE
https://issues.apache.org/jira/browse/KAFKA-6199
https://issues.apache.org/jira/browse/KAFKA-6307



两个版本的源码对比 JmxReporter

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210224100835.png)

