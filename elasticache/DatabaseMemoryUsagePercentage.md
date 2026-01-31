---
Title: Elasticache 'DatabaseMemoryUsagePercentage' alarm
Author: Pranav
Date: 2026/01/31
---

# ELASTICACHE *DatabaseMemoryUsagePercentage* ALARM

## Objective
* Monitor the percentage of memory that is being consumed by the redis process.
* Use cloudwatch to create an alarm and take actions when the alarm breaches the specified threshold.

## Overview
To be able to understand `DatabaseMemoryUsagePercentage` metric, we need to understand two other metrics called `BytesUsedForCache` and `maxmemory`.
* Cloudwatch `BytesUsedForCache` metric shows us how much memory(in bytes) Redis is currently using.
  This metric shows the **absolute memory** usage in bytes.
* Where as `maxmemory` is a Redis parameter, which sets the maximum amount of memory(in bytes) for the dataset and other uses, on the node.
  In elasticache, the default `maxmemory` value is subject to an another elasticache for Redis parameter called `reserved-memory-percent`.
  - Elasticache reserves a portion of `maxmemory` for OS-level processes, backups and failovers by using `reserved-memory-percent` parameter.
  - AWS recommends to reserve **one-fourth(25%)** of node type's `maxmemory` value as `reserved-memory-percent`. This is the default value and recommended for most cases.
  Because of this, the `maxmemory` value of the cluster is reduced. 
  For example, the **cache.t3.medium** node type has a `maxmemory` of 3.09GiB, but after applying the reserved memory of 25%, the applicable `maxmemory` will be:

$$
\text{maxmemory} = \text{(Overall maxmemory) - [(Overall maxmemory) x (reserved-memory-percent)]} = 3.09 - \frac{\text{3.09 x 25}}{100} = 2.31\text{GiB}
$$

Now, coming to the `DatabaseMemoryUsagePercentage` metric, this is a cloudwatch metric for elasticache which shows the percentage view of how much of the configured `maxmemory` limit Redis has consumed.
It shows the **relative memory** usage by Redis.
It is calculated as follows:

$$
\text{DatabaseMemoryUsagePercentage} = \frac{\text{BytesUsedForCache}}{\text{maxmemory}} \text{ x 100} = \frac{\text{BytesUsedForCache}}{\text{2.31}} \text{ x 100}
$$

## Alarm threshold calculation
Lets define a formula to calculate the threshold for `DatabaseMemoryUsagePercentage` alarm. 

$$
\text{Threshold} = \text{[(maxmemory)] - headroom} = \text{[(Overall maxmemory) - (reserved-memory-percent)] - headroom}= [100\\% - 25\\%] - 15\\% = 60\\%
$$

The above formula helps us calculate the threshold. 
The `reserved-memory-percent` used is a default of 25%. 
That means we are going to treat 75% of the `maxmemory` as the limit of safe usage, to which we need to specify a headroom(buffer) of about 15% to set the ideal threshold. 

## Glossary
| **Keyword** | **Description** |
| ------ | ----------- |
| CloudWatch | CloudWatch is AWS's monitoring and observability service. For elasticache, it collects metrics such as *CPUUtilization* and *EngineCPUUtilization*. These metrics can be used to set up alarms and take action when resource usage crosses a threshold. |
| GiB(gibibyte) | GiB is a **binary prefix** (1,073,741,824 bytes)used to measure computer storage and data capacity. It differs from GB(gigabyte), which is a **decimal prefix** equalling 1,000,000,000 bytes. Computers operate on binary systems, while humans are more accustomed to decimal systems. A GiB provides a precise measurement of storage space in binary terms, whereas a GB is a more general decimal-based unit.|


## References
1. https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Best_Practice_Recommended_Alarms_AWS_Services.html#ElastiCache
2. https://aws.amazon.com/blogs/database/monitoring-best-practices-with-amazon-elasticache-for-redis-using-amazon-cloudwatch/
3. https://docs.aws.amazon.com/AmazonElastiCache/latest/dg/redis-memory-management.html
4. https://redis.io/docs/latest/commands/INFO/
5. https://docs.aws.amazon.com/AmazonElastiCache/latest/dg/ParameterGroups.Engine.html
6. https://repost.aws/knowledge-center/available-memory-elasticache-redis-node