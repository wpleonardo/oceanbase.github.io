---
title: 常用的几种容灾部署方案
weight: 4
---

## 常用的几种容灾部署模式
上述两种高可用解决方案可以组合使用。OceanBase 数据库推荐如下多种部署模式，用户可根据对机房配置以及性能和可用性的需求进行灵活选择。

|   部署方案         |       容灾能力                                |  RTO  | RPO  |
|-------------------|-----------------------------------------------|-------|------|
| 同机房三副本       | （少数派副本故障时）机器级无损容灾/机架级无损容灾 | 8s 内 | 0    |
| 同城双机房物理备库 | （主机房故障时）机房级有损容灾                   | 秒级   | 大于 0 |
| 同城三机房三副本   | （少数派副本故障时）机房级无损容灾               | 8s 内  | 0    |
| 两地两中心物理备库 | （地域故障时）有损容灾                          | 秒级   | 大于 0 |
| 两地三中心加物理备库|（机房故障时）无损容灾/（地域故障时）有损容灾     | 秒级   | 机房故障时，RPO = 0；地域故障时，RPO 大于 0  |
| 三地三中心五副本   | （地域故障时）无损容灾                          | 8s 内  | 0  |
| 三地五中心五副本   | （地域故障时）无损容灾                          | 8s 内   | 0  |


### 同机房三副本

如果只有一个机房，可以部署三副本或更多副本，来达到机器级无损容灾。当单台 Server 或少数派 Server 宕机情况下，不影响业务服务，不丢数据。如果一个机房内有多个机架，可以为每个机架部署一个 Zone，从而达到机架级无损容灾。

### 同城双机房物理备库

如果同城只有双机房，又想达到机房级容灾能力，可以采用物理备库，每个机房部署一个集群。当任何一个机房不可用时，另一个机房可以接管业务服务。如果备机房不可用，此时业务数据不受影响，可以持续提供服务；如果主机房不可用，备库需要激活成新主库，接管业务服务，由于备库不能保证同步所有数据，因此可能会丢失数据。


**特点：**

* 每个机房都部署一个 OceanBase 集群，一个为主库一个为备库；每个集群有自己单独的 Paxos group，多副本一致性。
* "集群间" 通过 RedoLog 做数据同步，形式上类似传统数据库 "主从复制" 模式，从主库 "异步同步" 到备库，类似 Oracle Data Guard 中的 "最大性能" 模式。

**部署方案示图：**

![3](https://obbusiness-private.oss-cn-shanghai.aliyuncs.com/doc/img/observer-enterprise/V4.2.1/400.deploy/200.introduction-to-oceanbase-cluster-high-availability-deployment-scheme/3.deploy_plan13zb2.png)


### 同城三机房三副本

如果同城具备三机房条件，还可以为每个机房部署一个 Zone，从而达到机房级无损容灾能力。任何一个机房不可用时，可以利用剩下的两个机房继续提供服务，不丢失数据。这种部署架构不依赖物理备库，不过不具备地域级容灾能力。

**特点：**

* 同城 3 个机房组成一个集群（每个机房是一个 Zone），机房间网络延迟一般在 0.5 ~ 2 ms 之间。
* 机房级灾难时，剩余的两份副本依然是多数派，依然可以同步 RedoLog 日志，保证 RPO=0。
* 无法应对城市级的灾难。

**部署方案示图：**

![1](https://obbusiness-private.oss-cn-shanghai.aliyuncs.com/doc/img/observer-enterprise/V4.2.1/400.deploy/200.introduction-to-oceanbase-cluster-high-availability-deployment-scheme/1.deploy_plan13.png)

### 两地两中心物理备库

用户希望达到地域级容灾，但是每个地域只有一个机房时，可以采用物理备库架构，选择一个地域作为主地域，部署主库，另一个地域部署备库。当备地域不可用时，不影响主地域的业务服务；当主地域不可用时，备库可以激活为新主库继续提供服务，这种情况下可能会丢失业务数据。

更进一步，用户可以利用两地两中心实现双活，部署两套物理备库，两个地域互为主备。这样可以更加高效利用资源，并且达到更高的容灾能力。

### 两地三中心加物理备库

如果用户在两个不同的地域共有三个机房，可以使用 “两地三中心加物理备库” 的方案提供地域级容灾能力。

我们将有两个机房的地域称为主地域，业务在主地域两个机房里各部署一个或两个全功能副本，数据库的读写服务在主地域提供。另外一个地域机房中部署仲裁服务和物理备库，提供容灾服务。

在主地域一个机房出现故障时，仲裁方案会自动执行降级，确保业务在秒级恢复，同时不丢失数据。在主地域两个机房同时出现故障时，需要将物理备库激活成主库提供服务，此时业务有损，RPO > 0。

**特点：**

* 主城市与备城市组成一个 5 副本的集群。任何主城市 IDC 的故障，最多损失 2 份副本，剩余的 3 份副本依然满足多数派。
* 备用城市建设一个独立的 3 副本集群，做为一个备库，从主库 "异步同步" 到备库。
* 一旦主城市遭遇灾难，备城市可以接管业务。

**部署方案示图：**

![4](https://obbusiness-private.oss-cn-shanghai.aliyuncs.com/doc/img/observer-enterprise/V4.2.1/400.deploy/200.introduction-to-oceanbase-cluster-high-availability-deployment-scheme/4.deploy_plan25zb.png)



### 三地三中心五副本

为了支持地域级无损容灾，通过 Paxos 协议的原理可以证明，至少需要 3 个地域。该方案包含三个城市，每个城市一个机房，前两个城市的机房各有两个副本，第三个城市的机房只有一个副本。和两地三中心的不同点在于，每次执行事务至少需要同步到两个城市，需要业务容忍异地复制的延时。

### 三地五中心五副本

与三地三中心五副本类似，不同点在于，三地五中心会把每个副本部署到不同的机房，进一步强化机房容灾能力。


**特点：**

* 三个城市，组成一个 5 副本的集群。
* 任何一个 IDC 或者城市的故障，依然构成多数派，可以确保 RPO=0。
* 由于 3 份以上副本才能构成多数派，但每个城市最多只有 2 份副本，为降低时延，城市 1 和城市 2 应该离得较近，以降低同步 Redo Log 的时延。

**部署方案示图：**

![2](https://obbusiness-private.oss-cn-shanghai.aliyuncs.com/doc/img/observer-enterprise/V4.2.1/400.deploy/200.introduction-to-oceanbase-cluster-high-availability-deployment-scheme/2.deploy_plan35.png)