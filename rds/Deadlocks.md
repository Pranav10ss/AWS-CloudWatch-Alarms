---
Title: RDS Aurora MySQL 'Deadlocks' alarm
Author: Pranav
Date: 2026/02/01
---

# RDS Aurora MySQL *Deadlocks* alarm

## Objective
Configure a Cloudwatch alarm to detect high number of deadlocks in the database.

## Overview
A Deadlock in MySQL is a situation where **two or more transactions** are unable to proceed(neither `COMMIT` nor `ROLLBACK`), because each holds a **lock** to the resources(like rows or tables) that the other transaction needs. 
Because both transactions are waiting for a resource to become available, neither one ever releases the locks it holds.
MySQL's **deadlock detection mechanism** automatically detects this ***circular waiting*** when a deadlock occurs, and automatically **rolls back** one of the **transactions** involved(the **victim**). InnoDB tries to pick small transactions to roll back, where the size of the transaction is determined by the number of rows inserted, updated or deleted.

`Deadlocks` is a Cloudwatch metric that reports the average number of deadlocks in the database per second during the monitoring period. 
A **non-zero** metric value indicates that transaction(s) are failing and the application must retry them.
Frequent deadlocks reduces the concurrency of the database and degrades the application performance.

## Alarm configuration
Configure a Cloudwatch alarm for `Deadlocks` metrics. 
Set the threshold to **0**, with the comparison operator `GreaterThanThreshold`.

## Glossary
| **Keyword** | **Description** |
| ------ | ----------- |
| `COMMIT`| In MySQL, COMMIT is a transaction statement that commits the current transaction, making its changes permanent to the database. |
|`ROLLBACK`|In MySQL, ROLLBACK is a transaction statement that rolls back the current transaction, cancelling its changes.|
|InnoDB|In MySQL, InnoDB is the default general-purpose storage engine. InnoDB implements the locking at row-level(fine grained locking), which increases the concurrency and performance of the database. |
|Concurrency|Database concurrency refers to the capability of a DB to allow multiple transactions(a sequence of operations) to access & modify shared data seemingly at the same time. Without concurrency, transactions would have to run one after another, severely limiting the throughput & responsiveness of any multi-user application.|

## References
1. https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Aurora.AuroraMonitoring.Metrics.html#Aurora.AuroraMySQL.Monitoring.Metrics.instances
2. https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraMySQL.BestPractices.FeatureRecommendations.html
3. https://dev.mysql.com/doc/refman/8.0/en/glossary.html#glos_deadlock
4. https://severalnines.com/blog/understanding-deadlocks-mysql-postgresql/
5. https://dev.mysql.com/doc/refman/8.4/en/innodb-deadlock-example.html
6. https://dev.mysql.com/doc/refman/8.0/en/innodb-deadlock-detection.html