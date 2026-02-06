---
Title: RDS Aurora 'ACUUtilization' alarm
Author: Pranav
Date: 2026/02/01
---

# RDS Aurora *ACUUtilization* alarm

## Objective
* Understand how RDS Aurora serverless works. 
* Add a Cloudwatch alarm to monitor Aurora Serverless DB instance's compute capacity utilization. 
* Add `ServerlessDatabaseCapacity` metric widget to Cloudwatch dashboard to monitor ACU utilization.  

## Overview
Let us assume that we are using RDS with Aurora MySQL engine. 
A RDS cluster can have one primary(writer) DB instance and upto 15 replica(reader) DB instance in addition to the primary DB instance.
Aurora DB cluster separates the **shared storage volume** from the **compute capacity** of the DB instances in the cluster.

### Aurora Shared storage volume
Aurora data is stored in the *cluster* volume, which consists of copies of the data across three AZs in a single AWS region.
The Aurora shared storage architecture makes the data independent from the DB instances in the cluster. 
When we add a DB instance, Aurora doesn't make a new copy of the table data.
Instead, the DB instance connects to the shared volume that already contains all the data.

### Compute in Aurora
Aurora provides **Provisioned On-Demand Instances** and **Serverless instances**. 
We are currently using **Aurora Serverless v2**, which is an auto scaling configuration that automatically adjusts database capacity based on application needs. 
With Aurora Serverless, the database instance capacity automatically scales up or down to meet the application workload needs & auto-pauses(by scaling all the way to 0 ACUs if the MinCapacity is set to 0) during periods of inactivity. 
Aurora Serverless measures database instance capacity in Aurora Capacity Units(ACUs).
1 ACU has approximately 2GiB of memory with corresponding CPU & networking.

Each Aurora Serverless v2 DB writer or reader has a *capacity*.
The capacity increases or decreases whenever the writer or reader scales. 
We define the *capacity range* at the cluster level. 
The capacity range is the same for each Aurora Serverless v2 writer or reader in a DB cluster.
Capacity range is defined with **MaxCapacity** & **MinCapacity** parameters, which are ACU values that each writer or reader in the cluster can scale between.
For example, in the `helium-stg-rds-cluster` we have set the minimum capacity as 2 ACUs and the maximum capacity as 64 ACUs.
That means, the DB instances in the cluster can scale between 2-64 ACUs depending on the workload.
When the workload increases, Aurora Serverless v2 increases the number of ACUs in increments. 
Scaling happens in increments as small as 0.5 ACUs.

## Metrics
* `ServerlessDatabaseCapacity` metric represents the absolute value of ACUs that are being used by the DB instance. 
* `ACUUtilization` metric represents the percentage of ACUs used by the DB instance in relative to the maximum ACUs configured for the cluster. 
This metric can be defined with a formula;

$$
\text{ACUUtilization} = \frac{\text{Value of the ServerlessDatabaseCapacity metric}} {\text{Maximum ACU value of the DB cluster}}
$$

It is important to monitor `ACUUtilization` because, if this metric approaches a value of 100%, it is an indicator that the DB instance has scaled up as high as it can. 
This allows us to consider increasing the maximum ACU setting for the cluster.

**NOTE**: Suppose that a read-only workload causes a reader DB instance to approach an `ACUUtilization` of 100%, while the writer DB instance isn't close to its maximum capacity. 
In that case, we can consider adding additional reader DB instances to the cluster. 
That way, we can spread the read-only part of the workload spread across more DB instances, reducing the load on each reader DB instance. 

## Alarm configuration
Let us configure an alarm to monitor `ACUUtilization` metric. 
Set the threshold at **90%** of the maximum ACUs configured for the cluster.

## References
1. https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Aurora.Overview.html
2. https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-serverless-v2.setting-capacity.html#aurora-serverless-v2.viewing.monitoring
3. https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-serverless-v2.how-it-works.html
4. https://aws.amazon.com/rds/aurora/pricing/
5. https://aws.amazon.com/blogs/database/introducing-the-aurora-storage-engine/
6. https://docs.aws.amazon.com/AmazonRDS/latest/APIReference/API_ServerlessV2ScalingConfiguration.html
7. https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Aurora.Overview.StorageReliability.html#aurora-storage-growth
8. https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Aurora.Managing.Performance.html#Aurora.Managing.Performance.StorageScaling