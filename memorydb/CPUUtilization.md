---
Title: MemoryDB 'CPUUtilization' alarm
Author: Pranav
Date: 2026/01/31
---

# MemoryDB *CPUUtilization* alarm

## Objective
* Monitor the overall CPU load of the entire host instance.
* Use cloudwatch to monitor the MemoryDB cluster performance by collecting the metric data from every node in our cluster. 
  Understand what might be impacting the performance, set alarms based on threshold that correspond to their workload. 

## Overview
`CPUUtilization` metric for MemoryDB tracks the average CPU utilization across all the CPU cores on the node.
We need to use this metric to monitor the workload on redis and non-redis vCPU cores.
It is important to monitor the `CPUUtilization` metric as it gives us the complete picture of entire node's CPU health to ensure the instance is not overloaded by background processes and the database engine.
For example, the `db.t4g.small` instance type has 1 shard with 2 nodes, where each node has 2vCPUs.

  - Out of these 2vCPUs, one vCPU is dedicated to Redis for processing client commands and to handle network I/O. 
    CPU usage on this core can be tracked using the `EngineCPUUtilization` metric.
    On node types with 4 or more vCPUs, MemoryDB uses available vCPUs to offload network I/O from the main thread.
  - The other vCPU core handles background tasks like replication, backups and system overhead.

Background processes can take up a significant portion of the CPU workload. 
This is not significant on larger hosts with more than two vCPUs. 
But it can affect smaller hosts with 2vCPUs or fewer. 
If we only monitor the `EngineCPUUtilization` metric, we will be unaware of situations where the host is overloaded with both high CPU usage from the Valkey or Redis OSS engine and high CPU usage from the background monitoring processes. 
Therefore, we should monitor the `CPUUtilization` metric for hosts with two vCPUs or less.

## Alarm configuration
### Alarm threshold calculation: 
We need to determine the threshold based on the number of vCPU core the instance type has.
Threshold value should be calculated as a fraction of the node's total capacity.
For example, suppose we want to set the threshold to a percentage that reflects critical CPU usage level of 80% for redis-core and around 30% for non-redis core and we are using a node type that has two cores.
In this case, the threshold for `CPUUtilization` would be (80+30)/2, or 55%. 

The reason why we are setting higher baseline threshold on Redis's vCPU core and significantly lower on non-redis core is because, in a typical workload, the Redis core is often the one contributing to the overall CPU usage across the node.
The total CPU usage by the non-redis vCPU core will be significantly lower in comparision to the main core unless the background tasks are heavy. 

Lets define a formula that helps us calculate the threshold dynamically, regardless of the number of vCPUs the instance has.

$$
\text{Threshold} = \frac{\text{(Critical CPU usage level for redis core) + ((no. of vCPU cores - 1) x (Critical CPU usage level for non-redis core))}} {\text{Total number of vCPU cores}}
$$

* Using the above formula lets create a table which shows us the threshold for varying number of vCPU cores.

### The table below shows the alarm threshold for instances with *'n'* vCPU cores:
| Number of vCPU cores | Formula | Threshold value |
|:--------------------:|:-----------------------------------------------------:|:------------------|
| 2 | $$\displaystyle \frac{(80\\%)+((2-1)(30\\%))}{2}$$ | **55%** |
| 4 | $$\displaystyle \frac{(80\\%)+((4-1)(30\\%))}{4}$$ | **42.5%** |
| 8 | $$\displaystyle \frac{(80\\%)+((8-1)(30\\%))}{8}$$ | **36.25%** |
| 16 | $$\displaystyle \frac{(80\\%)+((16-1)(30\\%))}{16}$$ | **33.125%** |

## Glossary
| **Keyword** | **Description** |
| ------ | ----------- |
| CloudWatch | CloudWatch is AWS's monitoring and observability service. For MemoryDB, it collects metrics such as *CPUUtilization* and *EngineCPUUtilization*. These metrics can be used to set up alarms and take action when resource usage crosses a threshold. |
| Redis OSS | In context to MemoryDB, *Redis OSS* refers to open source software version of redis, which is a compatable engine used for in-memory caching. It stores data entirely in-memory for fast read and writes. |
| *EngineCPUUtilization* | A CloudWatch metric that measures the percentage of CPU used only by the redis thread. Since redis is single threaded, this metric is a good indicator of how much pressure is being placed on the redis engine itself. |
| I/O | In context to MemoryDB redis, I/O refers to input/output operations between clients and the redis server. *input* includes the commands sent by clients to redis and *output* includes the responses returned by redis.|
| Valkey | Valkey is an open source high performance key-value datastore. It is designed as a drop-in replacement for Redis OSS. Valkey is a fork of Redis OSS 7.2. Valkey was created to continue the development of a fully open-source, redis compatible in-memory data store. |

## References
1. https://docs.aws.amazon.com/memorydb/latest/devguide/what-is-memorydb.html
2. https://docs.aws.amazon.com/memorydb/latest/devguide/metrics.whichshouldimonitor.html
3. https://pages.awscloud.com/rs/112-TZM-766/images/2021_0919-DAT_Slide-Deck.pdf
4. https://aws.amazon.com/blogs/database/monitor-server-side-latency-for-amazon-memorydb-for-valkey/
5. https://assets.amazon.science/e0/1b/ba6c28034babbc1b18f54aa8102e/amazon-memorydb-a-fast-and-durable-memory-first-cloud-database.pdf
6. https://docs.aws.amazon.com/memorydb/latest/devguide/best-practices-pubsub.html
7. https://aws.amazon.com/memorydb/features/#topic-1