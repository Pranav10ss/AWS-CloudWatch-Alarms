---
Title: Elasticache 'CPUUtilization' alarm
Author: Pranav
Date: 2026/01/31
---

# ELASTICACHE *CPUUtilization* ALARM

## Objective
* Accurately monitor elasticache instance's overall CPU utilization.
* Monitor workload on redis-core and non-redis core from host-level perspective.
* Understand the difference between elasticache *CPUUtilization* metric and *EngineCPUUtilization* metric. 
* Use cloudwatch to monitor the elasticache cluster performance by collecting the metric data from every node in our cluster. 
  Understand what might be impacting the performance, set alarms based on threshold that correspond to their workload. 

## Summary
### Issue
`CPUUtilization` metric for elasticache tracks the average CPU utilization across all the CPU cores on the node.
We need to use this metric to monitor the workload on redis and non-redis CPU cores.
Since Redis OSS is single-threaded and most of the workload is tied directly to the single vCPU core used by the redis engine, hardcoding the value of alarm threshold to about 80~90% of the average CPU usage is ineffective as redis engine can become bottleneck long before the overall CPU usage reaches a saturation level.

### Proposal
We propose adopting the AWS recommended best practice for setting the threshold, where we need to determine the threshold based on the number of cores in the cache node that we are using.
We should calculate the threshold value as a fraction of the node's total capacity.
For example, suppose we want to set the threshold to a percentage that reflects critical CPU usage level of 80% for redis-core and around 30% for non-redis core and we are using a node type that has two cores.
In this case, the threshold for `CPUUtilization` would be (80+30)/2, or 55%. 
By implementing this approach we can define the alarm threshold that adapts to the number of vCPU cores the underlying node has.

## Overview
In order to calculate the alarm threshold, we need to understand the CPU architecture of the elasticache nodes.
For example, lets consider `cache.t3.medium` as the node type, which comes with two vCPUs.
As we know redis OSS is single-threaded which means it uses single thread(one vCPU core) to process client commands.
Lets assume that we are using redis OSS v5.0.6, where the network processing is also handled on the same thread.
Where as the other available vCPU is used for background tasks like backup, replication, sending the metrics to cloudwatch, system-level operations etc,.

We need to understand that most of the application's performance is directly related to the health of the vCPU core that redis uses to process commands, because in a typical workload `EngineCPUUtilization`(percentage of CPU usage on the redis thread) is often the one contributing to the overall CPU usage across the node.
The total CPU usage by the non-redis vCPU core will be significantly lower in comparision to the main core unless the background tasks are heavy. 

For example, lets assume that the `EngineCPUUtilization` is 60% and the CPU used by the non-redis core is 10%.
The average `CPUUtilization` reported will only be 35%.

$$
\text{Average CPU Utilization} = \frac{\text{CPU used by redis core } + \text{ CPU used by non-redis core}}{\text{Total number of vCPU cores}} = \frac{60\\% + 10\\%}{2} = 35\\% 
$$

## Alarm threshold calculation
* While calculating the threshold we need to set a critical CPU usage level for both redis-core and non-redis cores. 
  Lets set the critical CPU usage-level for redis core at 80% and non-redis core around 30%. 
* Lets also define a formula that dynamically helps us calculate the threshold regardless of the number of vCPUs the instance type has.

$$
\text{Threshold} = \frac{\text{(Critical CPU usage level for redis core) + ((no. of vCPU cores - 1) x (Critical CPU usage level for non-redis core))}} {\text{Total number of vCPU cores}}
$$

* Using the above formula lets create a table which shows us the threshold for '*n*' number of vCPU cores the elasticache instance has.

### The below table shows the alarm threshold for instances with *'n'* vCPU cores:
| Number of vCPU cores | Formula | Threshold value |
|:--------------------:|:-----------------------------------------------------:|:------------------|
| 2 | $$\displaystyle \frac{(80\\%)+((2-1)(30\\%))}{2}$$ | **55%** |
| 4 | $$\displaystyle \frac{(80\\%)+((4-1)(30\\%))}{4}$$ | **42.5%** |
| 8 | $$\displaystyle \frac{(80\\%)+((8-1)(30\\%))}{8}$$ | **36.25%** |
| 16 | $$\displaystyle \frac{(80\\%)+((16-1)(30\\%))}{16}$$ | **33.125%** |

## Glossary
| **Keyword** | **Description** |
| ------ | ----------- |
| CloudWatch | CloudWatch is AWS's monitoring and observability service. For elasticache, it collects metrics such as *CPUUtilization* and *EngineCPUUtilization*. These metrics can be used to set up alarms and take action when resource usage crosses a threshold. |
| Redis OSS | In context to elasticache, *Redis OSS* refers to open source software version of redis, which is a compatable engine used for in-memory caching. It stores data entirely in-memory for fast read and writes. Redis stopped being open source from v7.4 onwards.|
| *CPUUtilization* | A CloudWatch metric that measures the percentage of the total CPU capacity used by an entire elasticache host. For multi-core instances, this is averaged across all vCPUs. |
| *EngineCPUUtilization* | A CloudWatch metric that measures the percentage of CPU used only by the redis thread. Since redis is single threaded, this metric is a good indicator of how much pressure is being placed on the redis engine itself. |
| I/O | In context to elasticache redis, I/O refers to input/output operations between clients and the redis server. *input* includes the commands sent by clients to redis and *output* includes the responses returned by redis.|
| bottleneck | A situation where CPU component hits its performance limit and subsequently constraints the overall speed and the performance of the application. |
| Valkey | Valkey is an open source high performance key-value datastore. It is designed as a drop-in replacement for Redis OSS. Valkey is a fork of Redis OSS 7.2. Valkey was created to continue the development of a fully open-source, redis compatible in-memory data store. |

## References
1. https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Best_Practice_Recommended_Alarms_AWS_Services.html#ElastiCache
2. https://docs.aws.amazon.com/AmazonElastiCache/latest/dg/CacheMetrics.WhichShouldIMonitor.html
3. https://aws.amazon.com/blogs/database/boosting-application-performance-and-reducing-costs-with-amazon-elasticache-for-redis/
4. https://aws.amazon.com/blogs/database/monitoring-best-practices-with-amazon-elasticache-for-redis-using-amazon-cloudwatch/
5. https://aws.amazon.com/blogs/database/enhanced-io-multiplexing-for-amazon-elasticache-for-redis/
6. https://redis.io/blog/redis-adopts-dual-source-available-licensing/
7. https://aws.amazon.com/blogs/database/amazon-elasticache-and-amazon-memorydb-announce-support-for-valkey/