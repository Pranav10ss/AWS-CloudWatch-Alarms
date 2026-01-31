---
Title: Elasticache 'NetworkBandwidthAllowanceExceeded' alarm
Author: Pranav
Date: 2026/01/31
---

# ELASTICACHE *NetworkBandwidthAllowanceExceeded* ALARM

## Objective
* Detect when elasticache nodes exceed their allocated network bandwidth, which can lead to throttling.
* Leverage cloudwatch to create and trigger alarms whenever throttling occurs, enabling early detection of application performance issues.
* Add `NetworkBytesIn/Out` metric widgets to the cloudwatch dashboard.

## Overview
For example, a `cache.t3.medium` instance type has up to **5 Gigabit(Gbps)** of network bandwidth, which apply to both inbound and outbound traffic of the instance. 
If an instance specifies up to 5 Gbps of bandwidth, that means it has up to 5 Gbps of bandwidth for inbound traffic and, simultaneously, up to 5 Gbps for outbound traffic. 

AWS [network specifications](https://docs.aws.amazon.com/ec2/latest/instancetypes/gp.html#gp_network) page shows that the `Baseline / Burst bandwidth (Gbps)` for the EC2 instance type of **t3.medium** as `0.256 / 5.0`, these specifications applies to elasticache nodes as well since they're built on the same EC2 instance types.
- **0.256Gbps(~256Mbps)** is the baseline guaranteed network bandwidth of the instance.
    Even if the host is under heavy load, the instance should get at least 256Mbps for the traffic.
- **5 Gbps** is the maximum burst bandwidth the instance can reach.  
    To meet the additional network demand, instances can use network I/O credit mechanism to burst beyond their baseline bandwidth if the underlying host's NIC has unused capacity. 
    Instances can use burst bandwidth for a limited time, typically from 5 to 60 minutes, depending on the instance size.
    - The maximum capacity an instance can burst to, is capped by AWS & is subject to the documented burst bandwidth of the instance type.
      Actual achievable burst may vary depending on how network capacity is shared with other instances on the same host.
      This burst allowance is always ≥ baseline(256Mbps). 

$$
\text{Total Allowed bandwidth} = \text{baseline + burst}
$$

- The cloudwatch metrics `NetworkBandwidthInAllowanceExceeded` and `NetworkBandwidthOutAllowanceExceeded` metrics track the number of network packets that elasticache shapes(throttling by dropping packets) when the throughput exceeds the total allowed bandwidth(baseline + whatever burst AWS permits at that moment). 
When the packets are dropped, these metrics show a **non-zero** value. 

### Note:
Small bursts of traffic can cause traffic shaping, even if the average bandwidth is within the limits. 
If there are occasional spikes in these bandwidth allowance metrics with no effects on the application side, then no further action is required. 
Because Redis/Valkey OSS use TCP, and TCP retransmits dropped packets.

If these bandwidth allowance metrics are constantly high & the application sees latency issues, we can correlate the error timestamps with the throughput by checking the `NetworkBytesIn` and `NetworkBytesOut` metric widgets on the cloudwatch dashboard, which shows the inbound and outbound traffic volume respectively. 
After correlating the bandwidth allowance exceeded event with the `NetworkBytesIn/Out` data, we can conclude on the decisions of actions to be taken.
If the spike in throughput is caused by temporary reasons and causes no noticeable application impact, it may not require immediate action, as TCP will retransmit the dropped packets. 
If the high throughput is recurring and expected due to organic growth, then we can scale the cluster to resolve the issue.

When the packets are dropped by throttling, the application will have to retransmit the dropped packets, which will lead to latency.

## Alarm configuration
Cloudwatch provides `NetworkBandwidthInAllowanceExceeded` metric for ingress(inbound traffic) & `NetworkBandwidthOutAllowanceExceeded` metric for egress(outbound traffic), which are tracked separately at the ENI level.

`NetworkBandwidthInAllowanceExceeded` tracks the bandwidth throttling for traffic entering the node. 
Incoming traffic can be client requests, replication from primary node, backup downloads etc,. 

`NetworkBandwidthOutAllowanceExceeded` tracks the bandwidth throttling for traffic leaving the node. 
Outgoing traffic can be response to clients, replication to replica etc,.

When the incoming/outgoing traffic exceeds the node's current bandwidth allowance, throttling occurs which leads to drop in packets, the metric(s) show a non-zero value when this happens.
Let's configure the alarm for both of these metrics with a threshold of **0** and the comparision operator of `GreaterThanThreshold`.
Let's set the `evaluation_periods` and `datapoints_to_alarm` around **15 minutes** to avoid false positives.

## Glossary
| **Keyword** | **Description** |
| ------ | ----------- |
| Network I/O credit mechanism | An instance receives the maximum number of network I/O credits at launch. If the instance exhausts its network I/O credits, it returns to its baseline bandwidth. A running instance earns network I/O credits whenever it uses less network bandwidth than its baseline bandwidth. A stopped instance does not earn network I/O credits. Instance burst is on a best effort basis, even when the instance has credits available, as burst bandwidth is a shared resource. There are separate network I/O credit buckets for inbound and outbound traffic. |
|Throttling|Throttling is an intentional process of discarding network traffic when it exceeds the provisioned limits of an instance, in order to enforce a performance ceiling and ensure stability.|

## References
1. https://repost.aws/knowledge-center/elasticache-network-allowance-exceeded
2. https://aws.amazon.com/blogs/database/monitoring-best-practices-with-amazon-elasticache-for-redis-using-amazon-cloudwatch/
3. https://docs.aws.amazon.com/AmazonElastiCache/latest/dg/CacheMetrics.HostLevel.html
4. https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-instance-network-bandwidth.html
5. https://docs.aws.amazon.com/ec2/latest/instancetypes/gp.html#gp_network
6. https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/burstable-performance-instances.html