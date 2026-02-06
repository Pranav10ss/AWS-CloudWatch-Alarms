---
Title: RDS Aurora 'WriteLatency' alarm
Author: Pranav
Date: 2026/02/01
---

# RDS Aurora *WriteLatency* alarm

## Objective
Configure a Cloudwatch alarm to detect high write latency in Aurora DB instance.

## Overview
### Write operation in Aurora
The client performs a write operation like an `INSERT`, `UPDATE` or `DELETE` → Aurora's writer instance places the *page* that is to be modified into the *buffer pool*(memory) → The record of the change to be made to that *page* is written to a transaction log(*redo log*), which describes the changes to be made → After the *redo log* is written, Aurora sends it to distributed storage volume(*Cluster volume*), which automatically replicates the write to six storage nodes across 3 AZs(counted as one write).
After **at least four** of the six storage nodes acknowledge receipt, the write is considered successful, and acknowledgement is returned to the client application. → The client receives a successful `COMMIT` acknowledgement, which confirms that the **write** is durably persisted in the *cluster volume*.

All the steps described above collectively lead to one **Write I/O operation**. 
The Cloudwatch `WriteLatency` metric measures the average time taken by Aurora to complete a write I/O operation to the *cluster volume*. 
Monitoring `WriteLatency` helps to identify high write latency. 
A high write latency can be because a spike in write-heavy transactions that can saturate database instance's *redo log* buffer.
A high write latency can also be due to write workload exceeding the instance's resource limits like CPU and memory.

## Alarm configuration
AWS docs says that the write latencies higher than 20 milliseconds(0.02 seconds) are likely a cause for investigation and the alarm threshold should be set after analyzing the historical behavior of `WriteLatency` metric to determine sensible threshold level.
Based on the AWS recommendation and considering the past metric values, let us set the threshold as 0.02 seconds(20 milliseconds).

Set the **statistic** of the alarm as `p90`.
Cloudwatch calculates the 90th percentile of all the `WriteLatency` values collected in that period.
Using `p90` is recommended because, using `Average` misses spikes in write latency. 
So, when the value of p90 is greater than 0.02 seconds, the alarm will be triggered based on the evaluation periods. 

## Glossary
| **Keyword** | **Description** |
| ------ | ----------- |
| cluster volume |Aurora data is stored in the cluster volume, which is a single, virtual volume that uses solid state drives (SSDs). A cluster volume consists of copies of the data across three Availability Zones in a single AWS Region. |
|Pages| Relational databases traditionally interact with block storage. On this block storage, the data is stored in blocks of rows called *pages*. *Page size* is the smallest I/O operation that the database engine performs. Aurora supports a 16 KB page size for **Aurora MySQL**.|
| `COMMIT`| In MySQL, COMMIT is a transaction statement that commits the current transaction, making its changes permanent to the database. |

## References
1. https://aws.amazon.com/blogs/database/planning-i-o-in-amazon-aurora/
2. https://aws.amazon.com/blogs/database/introducing-the-aurora-storage-engine/
3. https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Aurora.Overview.StorageReliability.html
4. https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Best_Practice_Recommended_Alarms_AWS_Services.html#RDS
5. https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Aurora.AuroraMonitoring.Metrics.html#Aurora.AuroraMySQL.Monitoring.Metrics.instances
6. https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Aurora.Overview.StorageReliability.html#aurora-storage-type