---
Title: MemoryDB 'ReplicationDelayedWriteCommands' alarm
Author: Pranav
Date: 2026/01/31
---

# MemoryDB *ReplicationDelayedWriteCommands* alarm

## Objective
* Monitor if the client write operations are being delayed by the MemoryDB primary node because the multi-AZ durable transaction log is slow to acknowledge persistence. 
* Add an alarm using Cloudwatch to detect delayed writes.

## Overview
`ReplicationDelayedWriteCommands` metric for MemoryDB tracks the number of write commands that were delayed by the primary node because the durable, multi-AZ transaction log was slow to confirm the write.

When the application sends a write command to the primary node, the primary node sends the write to the multi-AZ transaction log. 
The transaction log must first confirm that it has persisted the write, & only after receiving the confirmation, the primary node sends the 'OK' response back to the application. 
The `ReplicationDelayedWriteCommands` metric counts the number of times the primary node delays in sending successful write 'OK' response to the client because transaction log was taking too long to confirm the write persistence. 

A high value in this metric would mean increased write latency.

A high count in delayed write commands is a clear indicator that the write volume is overwhelming the transaction log for the current number of shard in the cluster. 
A high traffic can also increase high write load.
We can use `ReplicationBytes` metric to check the traffic volume, which can be used for correlation with `ReplicationDelayedWriteCommands` metric.
The primary solution is to add more shards(horizontal scaling). 
This distributes the write load across more primary nodes, each with its own independent transaction log, allowing the cluster's overall write capacity to increase.

## Alarm configuration
When the write commands are delayed by MemoryDB, the `ReplicationDelayedWriteCommands` metric shows a **non-zero** value. 
Let's configure the alarm with a threshold of **0** and the comparision operator as **GreaterThanThreshold**. 
Let us set the `evaluation_periods` and `datapoints_to_alarm` to **5 minutes**.
Set the alarm period as **60 seconds**.
`Stat` of the alarm should be set to `Sum` to accurately capture the total number of delayed writes.

## References
1. https://docs.aws.amazon.com/memorydb/latest/devguide/metrics.memorydb.html
2. https://docs.aws.amazon.com/memorydb/latest/devguide/metrics.whichshouldimonitor.html#metrics-replication
3. https://docs.aws.amazon.com/memorydb/latest/devguide/replication.html