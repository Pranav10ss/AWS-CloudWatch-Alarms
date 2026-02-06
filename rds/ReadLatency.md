---
Title: RDS Aurora 'ReadLatency' alarm
Author: Pranav
Date: 2026/02/01
---

# RDS Aurora *ReadLatency* alarm

## Objective
Configure a Cloudwatch alarm to detect high Read latency in Aurora DB instance.

## Overview
### Read operation in Aurora
The client sends a `SELECT` query to the Aurora writer or reader instance → The instance checks the *buffer pool*(memory) that holds recently requested *pages* → If the *pages* are already present in the *buffer pool*, the query is served entirely from **memory**, with no read I/O to the *cluster volume* and the I/O for that query is zero. → *Pages* required to complete the query that don't exist in the *buffer pool* must first be read from the *cluster volume* into the *buffer pool*. Once all required pages for the query are available in the *buffer pool*, the query gets executed from the *buffer pool*. → The Aurora MySQL engine reads *pages* and stores the data to be returned to the client in a table called *result-set*. → Aurora sends the query results to the client. *Pages* in the *buffer pool* remain in memory indefinitely until it’s evicted due to memory pressure from other pages being read into the buffer pool or an engine restart.

**One read I/O operation** in Aurora corresponds to a *page* read from the **distributed storage volume**(*cluster volume*) into the buffer pool. 
If all the *pages* are already cached in the buffer pool, the read I/O will be zero.
Cloudwatch `ReadLatency` metric measures the time from when Aurora db instance sends a read request to the *cluster volume*, until the *page* is cached in the *buffer pool*. 
When this happens, the metric will show a non-zero value.

## Alarm configuration
AWS documentation states that the read latencies higher than 20 milliseconds(0.02 seconds) are likely a cause for investigation and the alarm threshold should be set after analyzing the historical behavior of `ReadLatency` metric to determine sensible threshold level.
Based on the past metric values in our environment, let us set the threshold as 0.01 seconds(10 milliseconds).

Set the **statistic** of the alarm as `p90`.
Cloudwatch calculates the 90th percentile of all the `ReadLatency` values collected in that period.
Using `p90` is recommended because, using `Average` misses spikes in read latency. 
So, when the value of p90 is greater than 0.01 seconds, the alarm will be triggered based on the configured evaluation periods. 

## Glossary
| **Keyword** | **Description** |
| ------ | ----------- |
| cluster volume |Aurora data is stored in the cluster volume, which is a single, virtual volume that uses solid state drives (SSDs). A cluster volume consists of copies of the data across three Availability Zones in a single AWS Region. |
|Pages| Relational databases traditionally interact with block storage. On this block storage, the data is stored in blocks of rows called *pages*. *Page size* is the smallest I/O operation that the database engine performs. Aurora supports a 16 KB page size for **Aurora MySQL**.|
| `SELECT`| In MySQL, SELECT is a transaction statement that is used to retrieve information from rows selected from one or more tables. |

## References
1. https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Best_Practice_Recommended_Alarms_AWS_Services.html#RDS
2. https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Aurora.AuroraMonitoring.Metrics.html#Aurora.AuroraMySQL.Monitoring.Metrics.instances
3. https://aws.amazon.com/blogs/database/introducing-the-aurora-storage-engine/
4. https://aws.amazon.com/blogs/database/planning-i-o-in-amazon-aurora/
5. https://dev.mysql.com/doc/refman/8.4/en/select.html