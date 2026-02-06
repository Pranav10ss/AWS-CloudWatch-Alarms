---
Title: RDS Aurora 'BlockedTransactions' alarm
Author: Pranav
Date: 2026/02/01
---

# RDS Aurora *BlockedTransactions* alarm

## Objective
Configure a Cloudwatch alarm to detect a high blocked transaction count in an Aurora DB instance in order to prevent transaction rollbacks & performance degradation.

## Overview
`BlockedTransactions` metric shows the number of currently `ActiveTransactions` that are **blocked** in the database. 
Blocked transactions(BT) can either end in a `ROLLBACK` or a `COMMIT`.
High concurrency, idles in transaction, or long running transactions can lead to blocked transactions. 
  - **High concurrency**: RDS MySQL can use row-level locking semantics. 
  When many concurrent transactions try to access or modify the same rows, the probability of lock contention increases. This increases lock waits and thus blocked transactions.
  - **Idle in transaction**: A session is considered **idle** when the transaction has started(`START TRANSACTION` or `BEGIN`), but haven't yet issued a `COMMIT` or `ROLLBACK`. Transaction get blocked because, the row in the table that a transaction want to update is being **locked** by another transaction(thus preventing other sessions from acquiring the lock).
  So, a BT is waiting for another transaction to release the **lock**.
  The blocked transaction can proceed only when the other transaction commits or rolls back.
  - **Long-running transactions**: A long running query statement can hold its **lock** for longer duration, blocking any other transaction that need to update the same row.

A high `BlockedTransactions` can lead to high query response times which leads to increased latency and application performance degradation.
A high BT can also increase the `DatabaseConnections` as a result of consuming significantly high `max_connections` available for the Aurora MySQL DB instance.

Cloudwatch counts a transaction as blocked and shows a **non-zero** value whenever a transaction is in `LOCK WAIT` state. 
Once the transaction `COMMIT` or `ROLLBACK`, the lock is released and the BT proceeds, thus the metric falls back to zero.

## Alarm configuration
AWS document says that we should calculate the threshold value for `BlockedTransactions` as 5% of all `ActiveTransactions` of the DB instance.
We can use Cloudwatch math expression in the following way to configure the alarm.

```
m1 = BlockedTransactions
m2 = ActiveTransactions
e1 = m2 * 0.05
ALARM when m1 > e1
```

Since our Database activity is too low, the above math expression would lead to false positive if the DB has too low or 0 active transactions.
With the below expression, we can avoid false positives.

```
e1 = MAX([m2 * 0.05, 1])
```

## Glossary
| **Keyword** | **Description** |
| ------ | ----------- |
|`ActiveTransactions`| This is a Cloudwatch metric for RDS that shows the number of currently active database transactions. It shows the transactions that have `BEGIN`ed but not yet `COMMIT`ed or `ROLLBACK`ed. Idle connections that are not in a transaction(has not `BEGIN`ed) are not counted in this metric. |
|`DatabaseConnections`|This Cloudwatch metric includes all open connections, both **active** and **idle**. An idle session that hasn’t started a transaction is not counted in `ActiveTransactions` but is included in `DatabaseConnections`.|
|Concurrency| Database concurrency refers to the capability of a DB to allow multiple transactions(sequence of operations) to access & modify shared data seemingly at the same time. Without concurrency, transactions would have to run one after another, severely limiting the throughput & responsiveness of any multi-user application.|
|`START TRANSACTION` or `BEGIN`|In MySQL, a START TRANSACTION or BEGIN is a transaction statement that starts a new transaction.|
| `COMMIT` | In MySQL, COMMIT is a transaction statement that commits the current transaction, making its changes permanent to the database. |
| `ROLLBACK` | In MySQL, ROLLBACK is a transaction statement that rolls back the current transaction, cancelling its changes.|
|`LOCK TABLES`| MySQL enables client sessions to acquire table locks during a transaction to prevent other sessions from modifying the table. This means that during an operation, the table would be locked, preventing other read & write operations until the current operation is complete. MySQL locks the table to ensure data consistency. |

## References
1. https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Best_Practice_Recommended_Alarms_AWS_Services.html#RDS
2. https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Aurora.AuroraMonitoring.Metrics.html#Aurora.AuroraMySQL.Monitoring.Metrics.instances
3. https://dev.mysql.com/doc/refman/8.4/en/commit.html
4. https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/ams-waits.row-lock-wait.html
5. https://dev.mysql.com/doc/refman/5.7/en/innodb-autocommit-commit-rollback.html
6. https://dev.mysql.com/doc/refman/8.4/en/lock-tables.html
7. https://dev.mysql.com/doc/refman/8.0/en/innodb-information-schema-examples.html
8. https://docs.aws.amazon.com/ko_kr/whitepapers/latest/amazon-aurora-mysql-db-admin-handbook/transaction-management-and-autocommit.html
9. https://repost.aws/knowledge-center/blocked-mysql-query
10. https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/wait-event.locktransactionid.html
11. https://www.netdata.cloud/academy/what-is-database-concurrency/