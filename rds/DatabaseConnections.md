---
Title: RDS Aurora 'DatabaseConnections' alarm
Author: Pranav
Date: 2026/02/01
---

# RDS Aurora *DatabaseConnections* alarm

## Objective
Detect high number of client network connections to the database instance by configuring a Cloudwatch alarm.

## Overview
`DatabaseConnections` metric shows the number of client connections currently open to the database instance, whether **active**(currently executing queries) or **idle**(connections sitting idle in a connection pool). 
It shows the count of how many concurrent sessions the DB is handling at the moment.
The maximum number of connections allowed to an Aurora MySQL DB instance is determined by the `max_connections` parameter in the instance-level parameter group for the DB instance.
The default value of this parameter is calculated using a formula based on the memory size derived from the maximum Aurora Capacity Units(ACUs) configured for the DB instance, rather than the currently active ACU value.

For example, the default maximum connections that a Aurora MySQL Serverless v2 DB instance with maximum ACU defined as 64(128GiB) can have up to 4,000 active client connections at a time.
When we change the maximum ACU capacity of an Aurora Serverless v2 DB cluster, the DB instance has to be rebooted to update the `max_connections` value. 
However, if we set the `max_connections` parameter directly in a custom DB parameter group, no reboot is required. 

It is important to monitor `DatabaseConnections` metric because, a constant increase in connections can lead to exhaustion of the `max_connections` value for the DB instance, causing new client connections to be rejected.

If the clients frequently open & close connections, or keep a large number of long-lived connections open, we can use **Amazon RDS proxy**.
RDS proxy is a database proxy that uses connection pooling, which establishes a DB connection pool & reuses connections in the pool.
This approach avoids the memory & CPU overhead of opening a new DB connection each time. 
We can limit the number of connections that an RDS proxy can establish with a DB instance in the cluster by specifying the limit as a percentage of the `max_connections` available for the DB instance.

## Alarm configuration
Let us create a Cloudwatch alarm to identify high `DatabaseConnections`. 
We want to get notified when the connection count is around 80% of the `max_connections` value for the DB instance.

The alarm threshold for a DB instance with `max_connections` value of 4,000 can be calculated as;

$$
\text{Threshold} = \frac{\text{maximum connections value for the DB instance x 80}} {\text{100}} = \frac{\text{4000 x 80}} {\text{100}} = 3200
$$

So, the alarm threshold has to be set to **3200**.
The 80% threshold provides an early warning before the DB instance reaches its connection limit. 
This allows us to take actions such as RDS proxy or increasing the `max_connections` before client connections starts getting rejected.

## References
1. https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Best_Practice_Recommended_Alarms_AWS_Services.html#RDS
2. https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Aurora.AuroraMonitoring.Metrics.html#Aurora.AuroraMySQL.Monitoring.Metrics.instances
3. https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraMySQL.Managing.Performance.html#AuroraMySQL.Managing.MaxConnections
4. https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-serverless-v2.setting-capacity.html#aurora-serverless-v2.max-connections
5. https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-serverless-v2.setting-capacity.html#aurora-serverless-v2-examples-setting-capacity-range-walkthrough-ams
6. https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/rds-proxy.html
7. https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/rds-proxy-connections.html