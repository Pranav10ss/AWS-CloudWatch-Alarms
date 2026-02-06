---
Title: RDS Aurora storage related alarms
Author: Pranav
Date: 2026/02/01
---

# RDS Aurora storage related alarms

## Objective
* Understand how Aurora stores the data.
* Use CloudWatch metrics that help us identify high cluster volume.
Configure alarms to monitor the storage utilization.

## Overview
In Aurora, data is stored in *cluster volume*, which consists of the data across three AZs in a single region.
This storage volume automatically scales with the data in the cluster volume. 
As the data grows, the cluster volume storage expands depending on the engine version. 
Let us assume that we are using Aurora MySQL with engine version 3.08.2, the maximum size of the cluster volume is 128 TiB(tebibytes).

This engine version also supports *dynamic resizing*, which drops or reorganizes tables and databases that are no longer needed. 
Thus, the overall billed space decreases resulting in reduced storage charges.

Aurora adds storage volumes in 10 GB logical blocks called *protection groups*.
As the volume of data increases & exceeds the currently allocated storage, Aurora expands the volume to meet the demand & adds new protection groups, as necessary. 
It continues to scale this way until it reaches the limit of 128 TiB.

## Cloudwatch Metrics
### AuroraVolumeBytesLeftTotal
`AuroraVolumeBytesLeftTotal` is a Cluster-level metric that represents the **remaining available space** in the cluster volume of Aurora cluster. 
As the cluster volume grows, this value decreases. 
If it reaches zero, the cluster reports an out-of-space error.

### VolumeBytesUsed
`VolumeBytesUsed` shows the current storage that is being used by the cluster.
Let us add this metric widget to Cloudwatch dashboard.

### Temporary storage metrics
Aurora uses separate instance-level storage for non-persistent temporary files. 
This temporary storage is local EBS backed disk attached to DB instance.
This includes files that are used for purposes such as sorting large data sets during query processing, and building indexes.
The data and temporary files on these volumes are lost when starting and stopping the DB instance, and during host replacement.

Aurora MySQL creates internal(implicit) temporary tables in the RAM(in-memory) using the `TempTable` storage engine.
These tables are automatically created to handle operations such as sorting aggregation(such as `GROUP BY` on large tables).
If temporary data exceed the size of in-memory pool, Aurora stores the overflow data to local temporary storage(EBS-backed disk). 
When such disk based temporary tables are used for querying processes, `TempStorageIOPS` and `TempStorageThroughput` increase, indicating query spillover to disk which degrades the performance.
  - *TempStorageIOPS* - The number of IOPS for both read & writes on local storage attached to the DB instance.
  - *TempStorageThroughput* - The amount of data(in bytes/second) transferred to & from the local storage associated with the DB instance. 

## Alarm configuration
### AuroraVolumeBytesLeftTotal
Let us configure an alarm to monitor `AuroraVolumeBytesLeftTotal` metric. 
Set the threshold to **20%** with the the comparison operator as `LessThanThreshold`.

  - **Alarm action**: We are using Aurora engine version `8.0.mysql_aurora.3.08.2`, which has upto 128 TiB of maximum storage capacity. 
We can increase this maximum capacity to 256 TiB by upgrading the Aurora engine version to 3.10(compatible with MySQL 8.0.42) or higher.

### TempStorageIOPS
Configure an anomaly detection alarm for `TempStorageIOPS`.
The alarm will trigger when the metric values are higher than the normal metric behavior.

### TempStorageThroughput
Configure an anomaly detection alarm for `TempStorageThroughput`.
The alarm will trigger when the metric values are higher than the normal metric behavior.

## References
1. https://aws.amazon.com/blogs/database/introducing-the-aurora-storage-engine/
2. https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Aurora.Overview.StorageReliability.html#aurora-storage-growth
3. https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-serverless-v2.how-it-works.html?utm_source=chatgpt.com#aurora-serverless.storage
4. https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Aurora.Managing.Performance.html#Aurora.Managing.Performance.StorageScaling
5. https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/CHAP_Limits.html#RDS_Limits.FileSize.Aurora
6. https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Aurora.AuroraMonitoring.Metrics.html#Aurora.AuroraMySQL.Monitoring.Metrics.instances
7. https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/ams3-temptable-behavior.html