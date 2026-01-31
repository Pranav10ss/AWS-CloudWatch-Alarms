---
Title: MemoryDB 'CurrConnections' alarm
Author: Pranav
Date: 2026/01/31
---

# MemoryDB *CurrConnections* alarm

## Objective
* To set an alarm for monitoring `CurrConnections` metric, which represents the number of client connections that are currently established with the memorydb cluster. 
* Configure a cloudwatch alarm to detect the high connections count.

## Overview
`CurrConnections` metric for MemoryDB shows the number of active connections currently open to a MemoryDB node. 
MemoryDb enforces an internal maximum connection limit of 65,000 connections, which is derived from Redis `maxclients` parameter.

It is important to monitor `CurrConnections` metric closely to ensure that any abnormal increase in client connections are detected early, before it risks exhausting the available connection slots and causing `ERR max number of clients reached` errors.

## Solution
Configure an alarm with a critical threshold level of 80% of `maxclients` limit, so that we will be notified about the high connections issue.

$$
\text{Threshold} = \text{maxclients value x } 80\\% = \frac{\text{65,000 x 80}}{100} = 52,000
$$

So, setting the threshold to trigger the alarm if the active connections count crosses 52,000 is a safe way to monitor the `CurrConnections` metric and take actions before it effects the application performance.

## Glossary
| **Keyword** | **Description** |
| ------ | ----------- |
| Redis OSS | In context to MemoryDB, *Redis OSS* refers to the compatible engine used for the in-memory database. It stores data entirely in-memory for fast reads & writes, with durability provided by MemoryDB's underlying transactional log.|

## References
1. https://docs.aws.amazon.com/pdfs/memorydb/latest/devguide/memorydb-guide.pdf.pdf
2. https://pages.awscloud.com/rs/112-TZM-766/images/2021_0919-DAT_Slide-Deck.pdf
3. https://aws.amazon.com/blogs/database/monitor-server-side-latency-for-amazon-memorydb-for-valkey/
4. https://docs.aws.amazon.com/AmazonElastiCache/latest/dg/ParameterGroups.Engine.html
