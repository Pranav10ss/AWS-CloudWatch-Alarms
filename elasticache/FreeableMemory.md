---
Title: Elasticache 'FreeableMemory' alarm
Author: Pranav
Date: 2026/01/31
---

# ELASTICACHE *FreeableMemory* ALARM

## Objective
* Track the amount of free memory available on the elasticache host.
  While the `DatabaseMemoryUsagePercentage` metric alarm effectively tracks the data consumed by Redis data store relative to `maxmemory` limit, it misses the memory consumed by background processes like replication and backups as we have set the alarm threshold at 60% of the `maxmemory` limit by excluding the memory reserved for background processes. 
  `FreeableMemory` metric alarm adresses that gap by tracking memory consumed by both Redis and background processes relative to the overall host memory, which helps us identify overall high memory usage from host-level perspective.
* Use cloudwatch to create an alarm to determine whether a node is under memory pressure, and take actions when the alarm breaches the specified threshold.

## Overview
Let's take a look at the formula below so that we get a clear understanding of how `FreeableMemory` metric works. 

$$
\text{Total node memory} = \text{BytesUsedForCache + reserved memory + system overhead + FreeableMemory}
$$

Let's understand what each terms in the above formula mean:
* **Total node memory** - This is the total memory of the elasticache instance type that the node is running on. 
* **BytesUsedForCache** - A metric that shows the memory(in bytes) used by Redis to store the data.
* **Reserved memory** - Reserved memory is the memory that is set aside for non-data processes like backups and replication.
  Reserved memory can be calculated as,

$$
\text{Reserved Memory} = \text{maxmemory x } \frac{\text{reserved-memory-percent}}{100}
$$

* **System overhead** - This includes memory consumed by underlying Operating system for OS level processes, sending metrics to cloudwatch etc,. 
* **FreeableMemory** - This metrics tracks the actual amount of memory(in bytes) that is available on the host after accounting for all the above.
  `FreeableMemory` being close to 0 indicates that the node is under memory pressure.
  `FreeableMemory` can be calculated as:

$$
\text{FreeableMemory} = \text{Total node memory - (BytesUsedForCache + reserved memory + system overhead)}
$$

## Alarm configuration
25% of the `maxmemory` is reserved for background processes, which would leave 75% of the `maxmemory` for Redis processes. 
Since we have considered a buffer of 15% for safe usage limit, the overall memory percentage that would be left for Redis is 60%. 
We should also add a buffer of 5% to reserved memory's 25%. 
Let's calculate the threshold for `FreeableMemory` using the formula below: 

$$
\text{FreeableMemory Threshold} = \text{maxmemory - ((\\% cache memory - buffer) + (\\% of memory reserved for background processes - buffer))}
$$

$$
\text{FreeableMemory Threshold} = \text{100\\% - ((75\\% - 15\\%) + (25\\% - 5\\%))} = 20\\%
$$

To justify on why it is good to add a buffer on reserved memory percentage, lets look at a scenario where Redis's memory consumption is already at 60% & the reserved memory usage is also high(lets say above 20%), if we don't add a buffer and set the threshold significantly lower than 20% we would only be alerted when the reserved memory is at saturation level, which will be too late as it effects the background processes. 
We should be notified way before this happens to take action, so that we can consider increasing the reserved memory percentage.

So, set the threshold to **20%**, so that when the available memory on the host goes below 20%, the alarm will be triggered. 

## Glossary
| **Keyword** | **Description** |
| ------ | ----------- |
| CloudWatch | CloudWatch is AWS's monitoring and observability service. For elasticache, it collects metrics such as *CPUUtilization* and *EngineCPUUtilization*. These metrics can be used to set up alarms and take action when resource usage crosses a threshold. |
| maxmemory | **maxmemory** is a Redis parameter, which sets the maximum amount of memory(in bytes) for the Redis dataset.|
| reserved-memory-percent | **reserved-memory-percent** is a parameter used by elasticache to reserve a portion of **maxmemory** for processes like backups and replication. The default value of **reserved-memory-percent** is 25% of the **maxmemory** value.|

## References
1. https://repost.aws/knowledge-center/elasticache-cluster-high-memory-usage
2. https://aws.amazon.com/blogs/database/monitoring-best-practices-with-amazon-elasticache-for-redis-using-amazon-cloudwatch/
3. https://docs.aws.amazon.com/AmazonElastiCache/latest/dg/CacheMetrics.HostLevel.html
4. https://repost.aws/knowledge-center/elasticache-swap-activity