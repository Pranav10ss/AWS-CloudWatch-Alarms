---
Title: Elasticache 'SwapUsage' alarm
Author: Pranav
Date: 2026/01/31
---

# ELASTICACHE *SwapUsage* ALARM

## Objective
* Monitor high swap usage on elasticache host. 
* Use cloudwatch alarms to alert us when the swap usage is high.
* Ensure that the Redis performance remains stable.

## Overview
`SwapUsage` metric for elasticache shows the amount of swap space currently being used on the host's disk.
**Swap space** is a reserved area on hard disk that the operating system uses when the memory(RAM) is full. 
When Redis needs more memory than the **available instance memory**(can be measured by `FreeableMemory` metric), the linux kernel swaps infrequently accessed data from memory to this swap space.
This process is called **swapping**, which frees up memory for other active Redis operations.

**Swapping** allows the process to continue running but at a significantly slower pace because disk I/O is slower than accessing from memory.
This leads to degradation of application performance as the latency increases.

It is important to measure `SwapUsage` metric because, elasticache for Redis is an in-memory data store, meaning that the core purpose is to serve the data from the memory and reduce latency.
- When the swap usage is significantly high, it is an indicator that the node is under memory pressure. 
That means the free memory available on the host is low. 
In such cases, if the cluster mode is disabled, scaling up to a larger node type with larger memory can be a solution. 
However, for cluster mode enabled, scaling out to progressively increase the memory capacity is the most appropriate solution.

## Alarm configuration
We need to create an alarm that notifies us to take actions when the swap usage is high.  
We can't hardcode the alarm threshold to a specific value because, we don't get to know the actual swap space of the node type that we are using since elasticache is an AWS managed service. 

So, lets set up a cloudwatch anomaly detection alarm. 
This alarm generates a **band**, which is a range of expected values from the `SwapUsage` metric's past data, and represents it as normal metric behavior. 
When the swap usage deviates significantly from the normal behavior for a specified period of time, the alarm will enter 'In alarm' state.
Since we want to get notified when the `SwapUsage` is higher than usual, we should configure the alarm to trigger when the metric values are 'above' the **band**.

## Glossary
| **Keyword** | **Description** |
| ------ | ----------- |
| CloudWatch | CloudWatch is AWS's monitoring and observability service. For elasticache, it collects metrics such as *CPUUtilization* and *EngineCPUUtilization*. These metrics can be used to set up alarms and take action when resource usage crosses a threshold. |
| *FreeableMemory* | *FreeableMemory* is a cloudwatch metric for elasticache which represents the amount of free memory available on the host for OS to use. This is derived from the RAM, buffers, and cache that the OS reports as freeable. If memory usage increases because of cache data or OS overhead, then the *FreeableMemory* decreases. A decrease in *FreeableMemory* indicates low memory on the host. |
| Cloudwatch anomaly detection | Anomaly detection is a method in which cloudwatch applies statistical and machine learning algorithms for metric's past data. These algorithms continuosly analyze metrics performance to determine normal baselines and generates an anomaly detection model. The model generates a range of expected values that represent normal metric behavior. These expected range of values are shown as a **band**. We can choose whether the alarm is triggered when the metric value is above or below the **band** of expected values, or both.|

## References
1. https://repost.aws/knowledge-center/elasticache-swap-activity
2. https://aws.amazon.com/blogs/database/monitoring-best-practices-with-amazon-elasticache-for-redis-using-amazon-cloudwatch/
3. https://docs.aws.amazon.com/AmazonElastiCache/latest/dg/redis-memory-management.html
4. https://docs.aws.amazon.com/AmazonElastiCache/latest/dg/CacheMetrics.WhichShouldIMonitor.html#metrics-swap-usage
5. https://aws.amazon.com/elasticache/pricing/?nc=sn&loc=5
6. https://repost.aws/knowledge-center/elasticache-cluster-high-memory-usage