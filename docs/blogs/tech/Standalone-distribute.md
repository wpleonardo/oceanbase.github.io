---
slug: Standalone-distributed
title: 'Thoughts on the Integrated Architecture of OceanBase Database for Standalone and Distributed Modes'
---

> **About the author**:
> Yang Chuanhui, CTO of OceanBase, joined the OceanBase team in 2010 as one of the founding members. He led all major tasks of OceanBase architecture design and technology upgrades, and nurtured OceanBase Database to its full blossom in Ant Group. Under his leadership, the OceanBase team took the TPC-C benchmark test and broke the world record twice. He is also the author of Large-scale Distributed Storage Systems: Theory Analysis and Practical Framework. Mr. Yang is determined to lead the OceanBase team in making the next-generation enterprise-level distributed database more open, flexible, efficient, and easier to use.

My journey with large-scale distributed systems began in 2007, inspired by Google File System (GFS), MapReduce, and Bigtable of Google. In 2010, I joined Taobao to work on OceanBase Database.

OceanBase Database initially adopted a distributed architecture with very limited SQL functionality, which was extended to support more SQL features and made more versatile over time. When I first delved into large-scale distributed systems, I saw it as a sophisticated field, much like ChatGPT is today. The technology was ahead of its time, and protocols were hard to grasp. It took me over a year to understand Paxos, reading a dozen papers and engaging in extensive technical discussions with colleagues.

I once viewed distributed architectures as the pinnacle of IT software technology, believing only a distributed design can make a system cutting-edge. However, when we applied early versions of OceanBase Database to Taobao and Alipay, the feedback from users and database administrators (DBAs) centered on SQL compatibility, cost, and performance. They would not choose OceanBase Database over standalone MySQL unless OceanBase Database provides full compatibility and higher cost-effectiveness. They acknowledged the excellence of OceanBase Database in scalability and lossless disaster recovery and agreed that a distributed architecture is the right direction. However, they apologized, saying that due to rapid business growth this year, they could not afford to invest additional manpower and servers in database transformation.

They wanted a versatile database. I remember discussing with the developers of Google Spanner and asking why Google accepted its poor performance in standalone mode. They explained that Google's engineers are skilled enough to adapt applications to be asynchronous. Additionally, Google had Jeff Dean, who was able to enforce a unified infrastructure from the top down. I admired the developers working on infrastructure at Google, but also realized that this model was not scalable. For developers, a truly user-friendly distributed database must deliver both the high performance and low complexity of a standalone system and the flexible scalability and high availability of a distributed system.

In 2016, we released OceanBase Database V1.0 with a fully distributed architecture, where all nodes were both readable and writable. However, the high overhead for distributed operations on each node became an issue. With a large number of tables and partitions, even when idle, the system consumed several CPU cores for distributed operations. Therefore, the OceanBase Database V1.x series addressed database issues for large enterprises but struggled to gain widespread adoption in small and medium-sized enterprises.

In 2018, we started exploring ways to lower the barrier to a distributed database, making it accessible to everyone. Adjusting the underlying architecture of the database required great caution. We spent over two years on technical discussions and overall architecture design, and began detailed design and coding in mid-2020. After two more years, we released OceanBase Database V4.0, codenamed "Little Fish," in August 2022. OceanBase Database V4.0 laid the foundation for the integrated architecture for standalone and distributed modes but had many known issues. These issues were resolved in OceanBase Database V4.1, which was unveiled at the developer conference in March 2023.

We introduced the concept of integrated architecture in 2021 and, following the marketing team's suggestion, renamed it from "the integrated architecture for centralized and distributed modes" to "the integrated architecture for standalone and distributed modes" for clarity.

## 1. Feasibility Analysis: How to Eliminate the Overhead Brought by a Distributed Architecture**

  

The first step in architecture design is feasibility analysis. Anyone in technology knows that architecture design is about trade-offs—what makes it possible, the underlying principles, and what to sacrifice. In designing the integrated architecture, we assumed that, for a distributed database, despite its large data volume, 80% of operations are performed on single nodes, and 20% of operations are performed across nodes.

In the early days of promoting OceanBase Database within Alibaba Group, I proactively interacted with developers of each business line as both BD and SA. Despite the complexity of Alibaba Group's business lines, covering e-commerce, finance, logistics, local services, entertainment, maps, and healthcare, I came to the conclusion that most online B2C businesses could be distributed through user ID (user_id) sharding. After sharding by user_id, most operations are performed within single users, with only a few being performed across users.

**For example, in online banking, most of the time we are managing our own accounts, with only a small portion spent on cross-account actions such as transfers. Therefore, during the optimization of the system, we first ensured that the distributed architecture brought no overhead to the 80% single-server operations so that OceanBase Database could have comparable performance as standalone databases in terms of such operations. Then, we focused on maximizing the performance of the 20% cross-server operations.**

The distributed architecture brought overhead to single-server operations mainly due to its following two features: high availability and scalability. In 2013, a DBA at Alipay told me that enabling strong synchronization in Oracle could result in a performance drop of at least 30%. OceanBase Database adopted a strong synchronization design based on Paxos for lossless disaster recovery. Without changes to the architecture, OceanBase Database would not be comparable to standalone databases in performance.

Our approach was to make redo log commits asynchronous, thus eliminating the need for database worker threads to wait for log commit results. This minimized the overhead of strong synchronization, even with poor network and disk performance. In our sysbench test on a three-server OceanBase cluster, strong synchronization based on Paxos caused only about an 8% performance loss for OceanBase Database. This loss was acceptable and could be offset by optimizing other modules. The performance loss from scalability mainly came from data sharding, with each shard writing its own redo logs. You can think of each data shard as a mini database. The larger the number of shards, the greater the overhead brought by the distributed architecture for shard management on each node.

We have innovatively introduced dynamic log streams into the integrated architecture of OceanBase Database V4.0 for standalone and distributed modes. Each node has one log stream per tenant, with all data partitions of a tenant dynamically bound to its log stream. This avoids the overhead resulting from substantial log streams. Additionally, when you add new servers to the system, partitions can be unbound from the source log stream and re-bound to the target log stream for dynamic migration.

Many may wonder, after all these years of database development, why the OceanBase team came up with this idea while others did not. There is no magic. I believe the key lies in the fact that very few distributed databases in the world need to handle the extreme business scenarios facing OceanBase Database, such as the Double 11 shopping festival.

In the industry, scalability is implemented in different ways. Classic standalone databases do not support scalability and require changes to applications when a distributed architecture is needed. NewSQL implements scalability at the storage layer and implements functionality at the SQL layer. This approach is simpler but has a downside: every SQL request requires extra remote access, even when you are accessing your own account. OceanBase Database evolves from the fully distributed architecture in V1.x, V2.x, and V3.x to the integrated architecture for standalone and distributed modes in V4.x.

  

## 2. What Does the Integrated Architecture Mean to Developers?**

  

The integrated architecture for standalone and distributed modes seems versatile. So, what is its main focus at this stage? On one hand, I believe that the integrated architecture is a technological leap. It is more user- and developer-friendly than before and will gradually become a mainstream choice. On the other hand, it takes time for the new technology to mature and for OceanBase Database to have the same user experience and ecosystem as standalone databases. In the short term, the value of the integrated architecture for developers is reflected in the following aspects:

**First, it greatly lowers the barrier to distributed databases.** Common standalone NewSQL systems, such as CockroachDB and YugabyteDB, have poor performance, which is only one-tenth to one-fifth of that of MySQL in sysbench performance tests. As databases with an integrated architecture mature, these NewSQL systems will be phased out. I have indeed seen many users switch from their NewSQL systems to OceanBase Database to cut costs and boost efficiency.

**Second, it suits the scalability needs of growing businesses.** In my conversations with many small and medium-sized enterprises, I have found most of them ambitious. Although a standalone database suffices to handle their current data volumes, they are optimistic about future growth. They prefer to choose a database with an integrated architecture from the start to avoid the hassle of modifying applications and switching databases in the future.

Will databases with an integrated architecture eventually replace standalone databases? I believe this is the trend, but I also think it will take quite a long time.

  

## 3. Core Technical Mechanism**

  

Dynamic log streams are the core technology used in the integrated architecture for standalone and distributed modes. To achieve true integration, the following technical challenges must be addressed:

  

*   **Application transparency**: To eliminate the need for application changes during the switch from the standalone mode to the distributed mode, the client must support dynamic routing. During partition migrations in the backend database, data must be dynamically routed to target servers. Additionally, both standalone and distributed modes must support all SQL features.
*   **Single-server operations**: In standalone mode, only one set of redo logs exists, and single-server transactions write redo logs in a similar way as a classic standalone database. Classic standalone databases use a B+ tree in their storage engine. OceanBase Database has innovatively integrated the idea of a B+ tree into its storage engine based on a log-structured merge-tree (LSM tree). This retains the high compression ratio of the LSM tree, enables hotspot data to be stored in memory, and minimizes write amplification incurred by the LSM tree. As a result, even with strong synchronization enabled across three servers, OceanBase Database V4.1 outperforms MySQL 8.0 in both the performance and the storage cost of single-server operations.
*   **Cross-server operations**: Cross-server operations must be supported by the underlying distributed architecture and cause no impact on the upper-layer SQL functionality. If a transaction involves only one server, it must be processed on that server. If a transaction involves multiple servers, it must be processed across the servers based on the two-phase commit protocol. Additionally, performance must be further optimized through distributed, parallel, and asynchronous technologies.
*   **Migration cost**: Migration operations are performed in the background, typically with throttling applied during execution. Assume that data is migrated at the maximum speed of 200 MB/s, which uses about 20% of the bandwidth of a 10 Gbit/s network interface card (NIC). During the migration process, data is copied with minimal CPU usage. Unless in extreme scenarios such as midnight on the Double 11 shopping festival, the background migration does not affect foreground online transactions. If the data volume is 1 TB, the migration duration, calculated based on the formula 1 TB/200 MB/s = 5000s, is about 1.5 hours.

  

## 4. Actual Results**

  

We shared the performance data of OceanBase Database at the developer conference in March 2023 and demonstrated its scalability through TPC-C tests over the past years.

  

*   **Performance in standalone mode**: In scenarios where OceanBase Database and MySQL were deployed on servers with 32 CPU cores, OceanBase Database V4.1 outperformed MySQL 8.0 in all sysbench tests, including point-select, read-only, write-only, read-write, insert, and update scenarios. In the read-write scenario, the most comprehensive test, OceanBase Database V4.1 outperformed MySQL 8.0 by 39%.
*   **Cost-effectiveness on public clouds**: We deployed MySQL in primary/standby mode on two servers with 4 CPU cores and 16 GB of memory, and deployed OceanBase Database on three servers of the same specifications, with two servers holding full-featured replicas and one server holding log-only replicas. Regardless of the storage size—from 100 GB, 300 GB, and 500 GB to 1 TB—OceanBase Database V4.1 delivered higher cost-effectiveness than MySQL 8.0 on Alibaba Cloud and Amazon Web Services (AWS). As the storage size increased, the advantage of OceanBase Database became more noticeable. Compared to MySQL on clouds, OceanBase Database cuts the total cost of ownership by 18.57% to 42.05% to offer the same performance while enabling lossless, three-replica disaster recovery.
*   **Scalability in TPC-C tests**: OceanBase Database participated in two TPC-C tests, with the latest using more than 1,500 servers. The TPC-C workload well reflected real-world scenarios because it included 10%‒15% distributed transactions and 85%‒90% local transactions. As shown in the TPC-C report released on the official website, the performance of OceanBase Database increased proportionally with the number of servers.

  

## 5. Issues Worth Discussion**

  

The integrated architecture for standalone and distributed modes is not perfect. Some of its issues are worth further discussions with developers and users.

  

### i. From distributed to standalone versus from standalone to distributed

  

Which approach is better: from distributed to standalone or from standalone to distributed? I believe only the path from distributed to standalone is viable. Distributed databases are an order of magnitude more technically challenging and are less widely adopted than standalone databases. From a return on investment (ROI) perspective, it is unlikely for a standalone database to trade mainstream scenarios for higher-end but smaller-scale scenarios at greater expense, especially with all the existing technical debt. For all business cases, the most effective strategy is to start with the high-end market and then expand to the low-end market.

  

Technological innovations often come from external sources. For example, it is Tesla, an electric vehicle company, rather than traditional fuel vehicle manufacturers, that led the transformation to electrification. Tesla first rolled out Model X and Model S for the high-end market, and then unveiled more affordable Model 3 to gradually capture the mainstream market. In this sense, distributed technology is much like the battery of electric vehicles.

  

### ii. Fully distributed scenarios

  

We have built the integrated architecture for standalone and distributed modes based on the following assumption: In a distributed database, the majority of requests are performed on single servers, while the minority of requests are performed across servers. If this assumption does not hold, the performance per server of the distributed database decreases when the number of servers increases. How do we address this? We can further divide fully distributed scenarios into two types. One is online analytical processing (OLAP) scenarios, which are hard to localize due to large data volumes and complex dimensions for individual users.

  

However, this type of scenario does not require high concurrency. Therefore, the key to better performance is fully utilizing server resources through techniques such as parallelization and vectorization. As each SQL statement is very large, an extra network request contributes very little to the overhead throughout the execution of the SQL statement. The other is online transaction processing (OLTP) scenarios. Assume that an OLTP business involves only cross-user transfers. If the data volume is small, the integrated architecture can be deployed on a single node to avoid the overhead incurred by the distributed mode. If the data volume is large, the integrated architecture must be deployed on multiple servers. In this case, although a dramatic decrease in the performance per server is inevitable, a database with an integrated architecture is still comparable to a database with a shared-nothing architecture.