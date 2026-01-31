---
Title: MemoryDB 'EngineCPUUtilization' alarm
Author: Pranav
Date: 2026/01/31
---

# MemoryDB *EngineCPUUtilization* alarm

## Objective
* Monitor the percentage CPU utilization of the Valkey or Redis OSS engine thread. 
* Use **Cloudwatch** to monitor the load placed on the Redis engine's vCPU core. 
  Set an alarm that notifies when the engine CPU utilization is higher than the specified threshold. 

## Overview
`EngineCPUUtilization` metric for MemoryDB tracks the CPU utilization of the Redis OSS engine's vCPU core.
Redis OSS is single-threaded, which means client command processing will be handled by a single vCPU core.
For example, a `db.t4g.small` node has 2 vCPUs. 
Both client command processing and network processing will be handled on the same vCPU core, which means most of the application's performance is directly related to the health of this Redis core. 
Redis core can reach high CPU usage level due to some of the reasons like long-running commands, a high number of requests, an increase of new client connection requests in a short period of time etc,..

It is important to monitor for `EngineCPUUtilization` because, when a Redis core's CPU usage has reached a saturation level, it cannot process commands quickly enough, causing new commands to queue up. 
This directly leads to latency. 

## Solution
We should set the alarm threshold that reflects a critical engine CPU utilization level for our application. 
Setting the alarm threshold around **80%** helps us to identify if the Redis core is under stress and helps us take action before the application experiences any performance degradation. 
A consistently high `EngineCPUUtilization` indicates that the current cluster set-up is not able to handle the high workload, through which we can consider scaling the cluster horizontally.

## Glossary
| **Keyword** | **Description** |
| ------ | ----------- |
| CloudWatch | CloudWatch is AWS's monitoring and observability service. For MemoryDB, it collects metrics such as *CPUUtilization* and *EngineCPUUtilization*. These metrics can be used to set up alarms and take action when resource usage crosses a threshold. |
| *EngineCPUUtilization* | A CloudWatch metric that measures the percentage of CPU used by the Redis thread. Since Redis is single threaded, this metric is a good indicator of how much pressure is being placed on the Redis engine itself. |
| Horizontal scaling | Horizontal scaling of a MemoryDB cluster can be done by *sharding*, which is a method of database partitioning that separates large databases into smaller, faster and more easily managable parts called **data shards**. *Sharding* can be done by increasing the number of shards in the cluster, through which dataset is automatically partitioned and distributed across primary nodes in each shard. Each primary node is responsible for a subset of the total keys which allows us to scale cluster's read/write capacity thus improving the overall application performance. |
| Redis OSS | In context to MemoryDB, *Redis OSS* refers to open source software version of Redis, which is a compatable engine used for in-memory caching. It stores data entirely in-memory for fast read and writes.|
| Vertical scaling | Vertical scaling means increasing or decreasing the size of the MemoryDB instance. When a node size is not sufficient for the workload, we can choose a larger node type with more powerful vCPU that can process commands faster. |

## References
1. https://aws.amazon.com/blogs/database/monitor-server-side-latency-for-amazon-memorydb-for-valkey/
2. https://docs.aws.amazon.com/memorydb/latest/devguide/metrics.memorydb.html
3. https://docs.aws.amazon.com/memorydb/latest/devguide/metrics.whichshouldimonitor.html