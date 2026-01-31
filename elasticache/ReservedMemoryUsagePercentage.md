---
Title: Elasticache 'ReservedMemoryUsagePercentage' alarm
Author: Pranav
Date: 2026/01/31
---

# ELASTICACHE *ReservedMemoryUsagePercentage* ALARM

## Objective
* Define a custom metric using cloudwatch math expression to monitor the memory utilized by elasticache background processes relative to the total memory reserved.
* Use cloudwatch to create an alarm and take actions when the alarm breaches the specified threshold.

## Overview
Cloudwatch does not provide a metric to track the memory utilized by background processes like backups, replication and OS overhead. 
Lets combine all the background processes(BGP) and define a custom term called `MemoryUsedByBGP` to find out the memory consumed, which can be expressed using the formula below; 

```math
$$
\text{Node memory - FreeableMemory} = \text{BytesUsedForCache + MemoryUsedByBGP} \quad ----------(1)
$$
```

Let's understand what each term in the above formula mean:
- **Node memory**: Total host memory in bytes.
  - 25% of the node memory is reserved for background processes.

```math
$$
\text{MemoryReservedForBGP} = \text{Node memory} * \frac{25}{100} \quad ----------(2)
$$
```

- **FreeableMemory**: This is a cloudwatch metric that tracks the free memory remaining on the host, calculated after the memory consumed by all the elasticache processes.
- **BytesUsedForCache**: This is a cloudwatch metric that tracks the amount of memory that is being used by Redis data store.
- **MemoryUsedByBGP**: This is a custom metric that we are defining using cloudwatch math expression to track the memory used(in bytes) by background processes like replication, backups & OS overhead.
  - Using equation (1), let's define `MemoryUsedByBGP`;

```math
$$
\text{MemoryUsedByBGP} = \text{(Node memory - FreeableMemory) - (BytesUsedForCache)} \quad ----------(3)
$$
```

Using equations (2) and (3), the `ReservedMemoryUsagePercentage` can be calculated as;

$$
\text{ReservedMemoryUsagePercentage} = \frac{\text{(3)}}{\text{(2)}} * 100 = \frac{\text{MemoryUsedByBGP}}{\text{MemoryReservedForBGP}} * 100
$$

## Alarm configuration
We are going to make use of cloudwatch's math expression to calculate `MemoryUsedByBGP` (e1) using `FreeableMemory` and `BytesUsedForCache` metrics. 
After that we can define another math expression for `ReservedMemoryUsagePercentage` (e2), against which we are going to set a threshold of **80%**. 
The alarm triggers if the memory usage percentage is greater than 80%. 
With this, we get notified if the memory utilized by the background processes are high relative to the total memory reserved for them.
The way we define the math expression to find out `ReservedMemoryUsagePercentage` (e2) can be analyzed through the code below;

```
  metric_query {
    id = "m1"
    metric {
      metric_name = "FreeableMemory"
    }
  }

  metric_query {
    id = "m2"
    metric {
      metric_name = "BytesUsedForCache"
    }
  }

  metric_query {
    id = "e1"
    expression = "(Node memory - m1) - m2"
    label = "MemoryUsedByBGP"
  }

  metric_query {
    id = "e2"
    return_data = true
    expression = "(e1 / MemoryReservedForBGP) * 100"
    label = "ReservedMemoryUsagePercentage"
  }
```

## Glossary
| **Keyword** | **Description** |
| ------ | ----------- |
| CloudWatch | CloudWatch is AWS's monitoring and observability service. For elasticache, it collects metrics such as *CPUUtilization* and *EngineCPUUtilization*. These metrics can be used to set up alarms and take action when resource usage crosses a threshold. |

## References
1. https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Best_Practice_Recommended_Alarms_AWS_Services.html#ElastiCache
2. https://aws.amazon.com/blogs/database/monitoring-best-practices-with-amazon-elasticache-for-redis-using-amazon-cloudwatch/
3. https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/using-metric-math.html