---
Title: Elasticache 'CurrConnections' alarm
Author: Pranav
Date: 2026/01/31
---

# ELASTICACHE *CurrConnections* ALARM

## Objective
* To set an alarm for monitoring `CurrConnections` metric, which represents the number of client connections that are currently established with the Redis server. 
* Use cloudwatch to monitor the high connection count, review the capacity and expected workload of the elasticache cluster to establish a baseline and then select a threshold accordingly.
* Take actions if the alarm triggers to prevent heavy load or performance issues.

## Overview
`CurrConnections` metric for elasticache shows the number of concurrent and active connections registered by the redis engine. 
Redis enforces a maximum connection limit called `maxclients`, which is a Redis parameter that specifies the maximum number of clients that can be connected at one time, beyond which new connections are rejected. 
The `maxclients` value for most of the elasticache instance types is 65,000 [[↗]](https://docs.aws.amazon.com/AmazonElastiCache/latest/dg/ParameterGroups.Engine.html). 

It is important to monitor the number of active client connections because of the following reasons:
* A constant increase in connections can lead to exhaustion of the 65,000 available connections. 
  When a cluster node reaches its maximum connection quota, the application attempting to connect will receive an error like:

```
ERR max number of clients reached
```
* Connection count nearing the `maxclients` limit may indicate an issue on the application side and connections improperly closed, leaving the connections established on the server side.

## Solution
Configure an alarm with a critical threshold level of 80% of `maxclients` limit, so that we will be notified about the high connections issue.

$$
\text{Threshold} = \text{maxclients value x } 80\\% = \frac{\text{65,000 x 80}}{100} = 52,000
$$

So, setting the threshold to trigger the alarm if the active connections count crosses 52,000 is a safe way to monitor the `CurrConnections` metric and take actions before it effects the application performance.

### Actions to take when the alarm triggers:
1. Ensure that the cluster is using `tcp-keepalive` to detect and terminate unclosed connections.
2. If the high connection count is due to increase in traffic, scale the cluster horizontally.
3. Use a client connection pool with a **finite size**, so that the application can re-use existing connections to avoid extra cost of creating a new connection. 
    
    It is important to specify a cap for the pool size with `max_connections` possible.  
    Without this, if the incoming requests are higher than the number of idle connections in the pool, new connections are opened and once the request is served, the connections are put back to the pool of idle connections without actually closing them. 
    These connections will lead to increase in `CurrConnections` count.

## Glossary
| **Keyword** | **Description** |
| ------ | ----------- |
| CloudWatch | CloudWatch is AWS's monitoring and observability service. For elasticache, it collects metrics such as *CPUUtilization* and *EngineCPUUtilization*. These metrics can be used to set up alarms and take action when resource usage crosses a threshold. |
| Redis OSS | In context to elasticache, *Redis OSS* refers to open source software version of redis, which is a compatable engine used for in-memory caching. It stores data entirely in-memory for fast read and writes. Redis stopped being open source from v7.4 onwards.|
| *EngineCPUUtilization* | A CloudWatch metric that measures the percentage of CPU used only by the redis thread. Since redis is single threaded, this metric is a good indicator of how much pressure is being placed on the redis engine itself. |
| Connection pooling | When we initialize a connection pool, the client(Redis library) opens a small number of connections and adds them to the pool. Each time the application 'opens' a connection from the pool, the client returns one of these existing connections and notes the fact that it is in use. Later when the application 'closes' the connection, the client puts it back into the pool of available connections without actually closing it. The TCP connection remains open to Redis so it can be reused quickly. If all connections in the pool are in use but the app needs more, then the client can simply open new connections as necessary. In this way, the client eventually finds the right number of connections to satisfy the app's demands. Without connection pooling, every new request involves a TCP connection, leading to performance degradation, particularly under high request rates. Connection pooling saves time and resources by reusing existing TCP connections instead of repeatedly opening and closing them. |
| *tcp-keepalive* | *tcp-keepalive* is a Redis parameter. From version 3.2 onwards, Redis has *tcp-keepalive* enabled by default and set to about 300 seconds. TCP keepalive defines the idle time(in seconds), after which the elasticache kernel starts sending TCP ACKs to clients. If the client responds, the connection stays open. If the client doesn't respond, the kernel considers the connection dead and closes it. This prevents potential dead connections from consuming Redis `maxclients` slots. |

## References
1. https://docs.aws.amazon.com/AmazonElastiCache/latest/dg/ParameterGroups.Engine.html
2. https://repost.aws/knowledge-center/elasticache-redis-client-error-messages
3. https://aws.amazon.com/blogs/database/monitoring-best-practices-with-amazon-elasticache-for-redis-using-amazon-cloudwatch/
4. https://redis.io/docs/latest/develop/clients/pools-and-muxing/
5. https://redis.io/docs/latest/develop/reference/clients/
6. https://docs.aws.amazon.com/AmazonElastiCache/latest/dg/BestPractices.Clients.Redis.Connections.html
7. https://aws.amazon.com/blogs/database/best-practices-valkey-redis-oss-clients-and-amazon-elasticache/