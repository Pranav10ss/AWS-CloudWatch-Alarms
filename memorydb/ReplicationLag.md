---
Title: MemoryDB 'ReplicationLag' alarm
Author: Pranav
Date: 2026/01/31
---

# MemoryDB *ReplicationLag* alarm

## Objective
* Monitor the replication lag of MemoryDB replica nodes to detect delay in applying updates from the primary node. 
* Configure a Cloudwatch alarm to help us detect high replication lag.

## Overview
Each shard in MemoryDB uses passive replication, where the primary node executes mutations & acknowledges client writes only after the transaction log confirms persistence. 
The mutations are synchronously committed to the durable, distributed multi-AZ transaction log, which ensures data durability even if the primary fails immediately after. 

The replicas read the replication stream from the transaction log and asynchronously apply updates into their in-memory dataset. 
As a result, every replica holds an **eventually consistent** copy of the data set.
If the primary node fails, a failover is triggered to a consistent replica which ensures no data is lost during failover. 
The writes are now served from the new primary node. The old primary node is then replaced and synced from the transaction log.
This architecture offers strong consistency across failovers & ensures that the elected primary is fully up-to-date. 

`ReplicationLag` metric for MemoryDB represents how far behind (in seconds) the replica node is in applying changes from the primary node.
It is important to monitor `ReplicationLag` metric for MemoryDB because, high lag means the replica is behind in applying the transaction log-stream to its in-memory dataset. 
If the replica has to catch-up a lot during failover, failover takes longer & applications encounter longer downtime. 
High replication lag increases failover time, delaying processing until the new primary is ready. 

## Alarm configuration
The `ReplicationLag` metric is emitted by the replica node, and the lag on the primary node will be always zero. 
Setting a fixed threshold to catch the replication lag is difficult because the normal baseline varies by write load on the cluster. 
Let us configure an alarm with Cloudwatch anomaly detection.
Anomaly detection generates a **band**, which is a range of expected values from `ReplicationLag` metric's past data, and represents it as normal metric behavior. 
By setting the alarm to trigger when the metric values are 'above' the **band**, we can detect when the time taken by replication is significantly higher than the usual. 

## Glossary
| **Keyword** | **Description** |
| ------ | ----------- |
| Cloudwatch anomaly detection | Anomaly detection is a method in which Cloudwatch applies statistical and machine learning algorithms for metric's past data. These algorithms continuosly analyze metrics performance to determine normal baselines and generates an anomaly detection model. The model generates a range of expected values that represent normal metric behavior. These expected range of values are shown as a **band**. We can choose whether the alarm is triggered when the metric value is above or below the **band** of expected values, or both.|

## References
1. https://docs.aws.amazon.com/memorydb/latest/devguide/metrics.memorydb.html
2. https://docs.aws.amazon.com/memorydb/latest/devguide/consistency.html
3. https://docs.aws.amazon.com/memorydb/latest/devguide/metrics.whichshouldimonitor.html#metrics-latency
4. https://assets.amazon.science/e0/1b/ba6c28034babbc1b18f54aa8102e/amazon-memorydb-a-fast-and-durable-memory-first-cloud-database.pdf
5. https://aws.amazon.com/blogs/database/monitor-server-side-latency-for-amazon-memorydb-for-valkey/
6. https://aws.amazon.com/blogs/database/monitoring-best-practices-with-amazon-elasticache-for-redis-using-amazon-cloudwatch/