---
title: Disaster Recovery Architecture Overview
weight: 1
---

The database system supports data storage and queries in the application architecture. It is crucial for data security and business continuity of enterprises. High availability is the primary consideration in the architecture design of the database system. High availability includes high availability of services and high reliability of data. This topic describes the technologies for ensuring the high availability of services in OceanBase Database.

OceanBase Database provides a variety of technologies for ensuring high availability of services, including intra-cluster multi-replica disaster recovery and inter-cluster disaster recovery with the Physical Standby Database solution.

> Note: At present, *OceanBase Advanced Tutorial for DBAs* applies only to OceanBase Database Community Edition. Therefore, the arbitration replica feature of OceanBase Database Enterprise Edition is not described in this topic. For more information about the differences between the two editions, see [Differences between Enterprise Edition and Community Edition](https://en.oceanbase.com/docs/common-oceanbase-database-10000000001714481).

## Multi-replica Disaster Recovery

![High availability](/img/user_manual/operation_and_maintenance/en-US/disaster_recovery_architecture_design/01-001.png)

As shown in the preceding figure, the data service layer is an OceanBase cluster. This cluster contains three sub-clusters (zones). Each zone comprises multiple physical servers, and each physical server is called a data node, or an OBServer node. OceanBase Database adopts the shared-nothing distributed architecture, and OBServer nodes are equivalent to each other.

The data stored in OceanBase Database is distributed on multiple OBServer nodes in a zone, and other zones store multiple data replicas. In the preceding figure, the data in the OceanBase cluster has three replicas. Each replica is stored in a separate zone. The three zones comprise a complete database cluster to provide services for users.

OceanBase Database can implement disaster recovery at different levels based on different deployment modes.

* Server-level lossless disaster recovery: The unavailability of a single OBServer node is acceptable, and a lossless switchover can be automatically performed.

* Zone-level lossless disaster recovery: The unavailability of a single Internet data center (IDC) (zone) is acceptable, and a lossless switchover can be automatically performed.

* Region-level lossless disaster recovery: The unavailability in a city (region) is acceptable, and a lossless switchover can be automatically performed.

If your cluster is deployed on multiple OBServer nodes in an IDC, server-level disaster recovery can be achieved. If your OBServer nodes are deployed in multiple IDCs of a region, IDC-level disaster recovery can be achieved. If your OBServer nodes are deployed in multiple IDCs across different regions, region-level disaster recovery can be achieved.

OceanBase Database supports multi-replica disaster recovery based on the Paxos protocol. This solution provides high availability with a recovery point objective (RPO) of 0 and a recovery time objective (RTO) of less than 8 seconds when a minority of replicas fail, meeting the level 6 high availability standard in GB/T 20988-2007.

Multiple OBServer nodes in an OceanBase distributed cluster concurrently provide database services to ensure the high availability of database services. In the preceding figure, the application layer sends a request to OceanBase Database Proxy (ODP), also known as OBProxy, and ODP routes the request to the OBServer node where the requested service data is located. The request result is then returned to the application layer along the same path in the opposite direction. During the entire process, different components contribute to high availability in different ways.

In a cluster consisting of OBServer nodes, all data is stored based on partitions. Each partition has multiple replicas to ensure the high availability. The multiple replicas of a partition are distributed across different zones. Among the replicas, only one replica supports modifications and is called the leader, and other replicas are called followers. Data consistency is ensured between the leader and followers based on the Multi-Paxos protocol. If the OBServer node where the leader is located fails, a follower is elected as the new leader to continue to provide services.

The election service is the basis of high availability. Unlike earlier versions, OceanBase Database V4.x changes the granularity of the election service from partitions to log streams, with partitions mounted to log streams. Among multiple replicas of a log stream, one is elected as the leader based on the election protocol. This type of election is carried out when the cluster restarts or when the previously elected leader becomes faulty.

In OceanBase Database V4.x, the election service no longer depends on clock synchronization services such as Network Time Protocol (NTP) to ensure clock consistency across OBServer nodes in the cluster. Instead, the election service uses local clocks and an enhanced lease mechanism to ensure consistency. The election service adopts a priority mechanism to ensure that the optimal replica is elected as the leader. The priority mechanism also takes the specified primary zone and the status of OBServer nodes into consideration.

After the leader starts to provide services, user operations produce data modifications. All modifications are logged and synchronized to the followers. OceanBase Database uses the Multi-Paxos protocol for log synchronization. Based on the Multi-Paxos protocol, if the persistence of log data is completed on the majority of followers, the log data is retained even if the minority of replicas become faulty. Based on the multiple synchronous replicas, the Multi-Paxos protocol ensures no data loss or service interruption when the minority of OBServer nodes fail. The downtime of the minority of OBServer nodes is acceptable to the data written by users. When an OBServer node fails, the system can select a new replica as the leader to continue to provide database services.

Global Timestamp Service (GTS) is activated for each OceanBase Database tenant. GTS provides a read snapshot version and a commit version for all transactions executed within a tenant to ensure that transactions are committed in order. When GTS is abnormal, transaction-related operations within the tenant are affected. OceanBase Database ensures the reliability and availability of GTS in the same way as that for partition replicas. The GTS location for a tenant is determined by a special partition, which also has multiple replicas. For this special partition, a leader is selected by using the election service, and the OBServer node where the leader is located also hosts GTS. If this OBServer node fails, another replica is selected as the leader for the special partition to provide services. In addition, GTS automatically switches over to the OBServer node where the new leader is located and continues to provide services.

The preceding content describes the key components that help ensure the high availability of OBServer nodes. ODP also requires high availability to maintain its service performance. User requests are first forwarded to ODP. If ODP is abnormal, user requests cannot be properly processed. ODP also needs to handle OBServer node failures and perform fault tolerance operations.

Unlike a database cluster, ODP has no persistent state. ODP obtains the required data by accessing the relevant database. Therefore, ODP failures do not cause data loss. ODP is also a cluster service consisting of multiple nodes. The specific ODP node used to execute user requests is subject to a load balancing component such as F5. If an ODP node fails, the corresponding load balancing component automatically removes the node to ensure that new requests are not forwarded to the node.

ODP monitors the database cluster status in real time. It obtains the cluster system table in real time to check the health status of each server and the real-time location of each partition. ODP also monitors the service status of OBServer nodes based on network connections. If ODP detects an exception, it marks the corresponding OBServer node as faulty and switches over the services.

## Disaster Recovery with the Physical Standby Database Solution

The Physical Standby Database solution is an important part of the high availability capability of OceanBase Database.

This solution is applicable to multi-cluster deployment mode. Transaction logs are transmitted between multiple clusters to provide log-based physical hot backup services. In OceanBase Database V4.2.0 and later, the Physical Standby Database solution adopts an independent primary/standby architecture. Primary and standby tenants can be created. Only logs are transmitted between primary and standby tenants by using a direct network connection or a transmission channel established by using a third-party log service. Unlike the centralized architecture in earlier versions, clusters in the independent primary/standby architecture are separated from each other. You can manage the clusters more flexibly.

Logs are asynchronously transmitted from the primary tenant to standby tenants, and only the Maximum Performance mode is supported. Therefore, if you want to ensure strong data consistency during disaster recovery, use the multi-replica or arbitration-based disaster recovery solution.
