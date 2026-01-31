---
Title: MemoryDB 'SwapUsage' alarm
Author: Pranav
Date: 2026/01/31
---

# MemoryDB *SwapUsage* alarm

## Objective
* Monitor high swap usage on the MemoryDB host. 
* Use cloudwatch alarms to alert us when the swap usage is high.

## Overview
`SwapUsage` metric for MemoryDB shows the amount of swap space currently being used on the host's disk.
**Swap space** is a reserved area on hard disk that the operating system uses when the memory(RAM) is full. 
When the total memory demand from Redis exceeds the **available instance memory**(can be measured by `FreeableMemory` metric), the linux kernel swaps infrequently accessed data from memory to this swap space.
This process is called **swapping**, which frees up memory for other active Redis operations.

**Swapping** allows the process to continue running but at a significantly slower pace because disk I/O is slower than accessing from memory.
This leads to degradation of application performance as the latency increases.

It is important to measure `SwapUsage` metric because, MemoryDB for Redis is an in-memory database, meaning that the core purpose is to serve the data from the memory to reduce latency.
- When the swap usage is significantly high, it is an indicator that the node is under memory pressure. 
That means the free memory available on the host is low. 
In such cases, we can scale up to a larger node type or scale out by adding shards to reduce memory pressure on the host.

## Alarm configuration
We need to create an alarm that notifies us to take actions when the swap usage is high.  
We can't hardcode the alarm threshold to a specific value because, we don't get to know the actual swap space of the node type that we are using since MemoryDB is an AWS managed service. 

So, lets set up a cloudwatch anomaly detection alarm. 
This alarm generates a **band**, which is a range of expected values from the `SwapUsage` metric's past data, and represents it as normal metric behavior. 
When the swap usage deviates significantly from the normal behavior for a specified period of time, the alarm will enter 'In alarm' state.
Since we want to get notified when the `SwapUsage` is higher than usual, we should configure the alarm to trigger when the metric values are 'above' the **band**.

## Glossary
| **Keyword** | **Description** |
| ------ | ----------- |
| *FreeableMemory* | *FreeableMemory* is a cloudwatch metric for MemoryDB which represents the amount of free memory available on the host for OS to use. This is derived from the RAM, buffers, and cache that the OS reports as freeable. If memory usage increases because of cache data or OS overhead, then the *FreeableMemory* decreases. A decrease in *FreeableMemory* indicates low memory on the host. |
| Cloudwatch anomaly detection | Anomaly detection is a method in which cloudwatch applies statistical and machine learning algorithms for metric's past data. These algorithms continuosly analyze metrics performance to determine normal baselines and generates an anomaly detection model. The model generates a range of expected values that represent normal metric behavior. These expected range of values are shown as a **band**. We can choose whether the alarm is triggered when the metric value is above or below the **band** of expected values, or both.|

## References
1. https://aws.amazon.com/blogs/database/monitor-server-side-latency-for-amazon-memorydb-for-valkey/
2. https://docs.aws.amazon.com/memorydb/latest/devguide/metrics.whichshouldimonitor.html#metrics-swap-usage
3. https://docs.aws.amazon.com/memorydb/latest/devguide/metrics.HostLevel.html
4. https://aws.amazon.com/memorydb/pricing/
5. https://www.datadoghq.com/blog/amazon-memorydb-integration/