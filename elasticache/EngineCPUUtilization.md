---
Title: Elasticache 'EngineCPUUtilization' alarm
Author: Pranav
Date: 2026/01/31
---

# ELASTICACHE *EngineCPUUtilization* ALARM

## Objective
* Accurately monitor redis core's CPU utilization within the elasticache instance.
* Use **cloudwatch** to monitor the load of the redis process by collecting the metric data from the cache node. 
  Understand what might be impacting the performance, set an alarm based on the threshold that correspond to the workload. 

## Overview
`EngineCPUUtilization` metric for elasticache tracks the CPU utilization of the Redis OSS engine's vCPU core.
Redis OSS is single-threaded, which means client command processing will be handled by a single vCPU core.
Let us assume that we are using a `cache.t3.medium` node type with 2 vCPUs and with Redis OSS engine v5.0.6, both command processing and network processing will be handled on the same core, which means most of the application's performance is directly related to the health of the redis core. 
The total CPU usage by non-redis core will be significantly lower in comparision to the main core unless the background tasks like backup, replication, sending the metrics to cloudwatch and system-level operations are heavy. 
Redis core can reach high CPU usage level due to some of the reasons like long-running commands, a high number of requests, an increase of new client connection requests in a short period of time and high evictions when the cache doesn't have enough memory to hold new data.

It is important to monitor for `EngineCPUUtilization` because, when a redis core's CPU usage has reached a saturation level, it cannot process commands quickly enough, causing new commands to queue up. 
This directly leads to latency. 

## Solution
We should set the alarm threshold that reflects a critical engine CPU utilization level for our application. 
Setting the alarm threshold around **80%** helps us to identify if the Redis core is under stress and take action before the application experiences performance degradation. 
A consistently high `EngineCPUUtilization` indicates that the current node size is not sufficient for the workload, through which we can consider scaling the cluster horizontally.

## Glossary
| **Keyword** | **Description** |
| ------ | ----------- |
| CloudWatch | CloudWatch is AWS's monitoring and observability service. For elasticache, it collects metrics such as *CPUUtilization* and *EngineCPUUtilization*. These metrics can be used to set up alarms and take action when resource usage crosses a threshold. |
| *EngineCPUUtilization* | A CloudWatch metric that measures the percentage of CPU used by the redis thread. Since redis is single threaded, this metric is a good indicator of how much pressure is being placed on the redis engine itself. |
| Horizontal scaling | Horizontal scaling of an elasticache cluster can be done by *sharding*, which is a method of database partitioning that separates large databases into smaller, faster and more easily managable parts called **data shards**. *Sharding* can be done by enabling the `cluster mode`, through which dataset is automatically partitioned and distributed across multiple primary nodes. Each primary node is responsible for a subset of the total keys which allows us to scale cluster's read/write capacity thus improving the overall application performance. |
| Redis OSS | In context to elasticache, *Redis OSS* refers to open source software version of redis, which is a compatable engine used for in-memory caching. It stores data entirely in-memory for fast read and writes. Redis stopped being open source from v7.4 onwards.|
| Vertical scaling | Vertical scaling means increasing or decreasing the size of the elasticache instance. When a node size is not sufficient for the workload, we can choose a larger node type with more powerful vCPU that can process commands faster. |

## References
1. https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Best_Practice_Recommended_Alarms_AWS_Services.html#ElastiCache
2. https://docs.aws.amazon.com/AmazonElastiCache/latest/dg/CacheMetrics.WhichShouldIMonitor.html
3. https://docs.aws.amazon.com/AmazonElastiCache/latest/dg/WhatIs.Components.html#WhatIs.Components.Shards
4. https://docs.aws.amazon.com/AmazonElastiCache/latest/dg/Clusters.html
5. https://aws.amazon.com/blogs/database/monitoring-best-practices-with-amazon-elasticache-for-redis-using-amazon-cloudwatch/
6. https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/burstable-credits-baseline-concepts.html