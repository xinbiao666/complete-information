- [知乎基于Kubernetes容器平台实践](https://cloud.tencent.com/developer/news/338520)



# 基于 Kubernetes 的大数据融合

![image-20210420092404601](https://gitee.com/AiShiYuShiJiePingXing/img/raw/master/img/image-20210420092404601.png)

在大数据的场景下，知乎使用两条处理路径，实现了容器平台和大数据组件的融合。如上图所示，左边绿色的是实时处理，右边灰色的是批处理。

由于出发点的不同，这些组件的设计思想有着较大的差异：

批处理，主要是运行 ETL 任务，包括数据仓库的构建、离线分析等，因此它追求的是数据吞吐率和资源利用率，而对于时延本身并不敏感。

例如：一个 Map-Reduce 任务需要运行 1～2 个小时，这是非常正常的。并且它被设计为具有高容错性。

例如：某个 Map-Reduce 任务“挂掉”了，你完全可以通过上层的 Ozzie 或 Azkaban 对整个任务（job）进行重试。只要最终完成了重启，这些对于上层业务都将是“无感”的。

实时处理，对于时延较为敏感，且对于组件的可用性也要求比较高。一旦其中的任何节点“挂掉”或重启，都会导致数据“落地”（运营指标）的延迟，以及数据展示的失败。因此它的组件要求机器的负载不能太高。

![image-20210420092429008](https://gitee.com/AiShiYuShiJiePingXing/img/raw/master/img/image-20210420092429008.png)

当然，我们在对大数据生产环境的维护过程中，也经常遇到如下各种问题：

由于某个业务的变更，伴随着 Kafka 写入的流量出现猛增，也会拉高 Kafka 整个集群的负载。那么如果无法恢复的话，就会导致集群的瘫痪，进而影响整个生产环境。

治理的思路是：按照业务方将集群予以划分和隔离。

我们按照业务方将集群划分出几十套，那么面对这么多的集群所带来的成本，又该如何统一进行配置与部署管理呢？

我们通过使用K8S 模板，方便地实现了一键搭建出多种相同的运行环境。

由于每个业务方使用量的不同，会造成那些业务方使用量较小的应用，也需要被分配多台机器，且需要维护该集群的高可用性，从而带来了大量的资源浪费。

解决方案是：运用容器来实现细粒度的资源分配和配置。例如：对于这些较小的业务，我们仅分配一个配有单 CPU、单磁盘和 8G 内存的容器，而不是一整台物理机。

# 基于 Kubernetes 的 Kafka 集群平台

![image-20210420092502656](https://gitee.com/AiShiYuShiJiePingXing/img/raw/master/img/image-20210420092502656.png)

由于Kafka 的性能瓶颈主要存在于磁盘存储的 IOPS 上，我们通过如下的合理设计，实现了资源的分配管理。

具体方案是：以单块磁盘为资源单位，进行细粒度分配，即：用单个 Broker 去调度一块物理磁盘。

如此划分资源的好处在于：

它本身就能对 IOPS 和磁盘容量予以隔离。

对于几个 T 的硬盘而言，资源的划分粒度较细，而不像物理机那样动辄几十个 T。因此资源的利用率会有所提升，而且更适合于小型应用。

我们利用 Kafka 自身的 Replica 实现了数据的高可用性。不过容器与物理机在具体实现策略上有所区别：原来我们将 Broker 部署到一台机器之上，如今将 Broker 部署在一个容器里。

此时容器就变成了原来的物理机，而包含容器的物理机就相当于原来物理机所在的物理机架。

同时，我们也对控制 Replica 副本的分布策略进行了调整。我们把 Broker 的机架式感知，改成按机器予以处理，这样就避免了出现相同 topic 的副本被放在同一台机器上的 Broker 的情况。

采用了容器方式之后，故障的处理也变得相对简单。

由于采取的是单块硬盘的模式，因此一旦出现任何一块硬盘的故障，运维人员只需将故障盘更换下来，通过 Kafka 的 Replication 方案，从他处将数据拷贝过来便可，而不需要其他部门人员的介入。

![image-20210420092535376](https://gitee.com/AiShiYuShiJiePingXing/img/raw/master/img/image-20210420092535376.png)

在创建流程方面，由于当时 K8S 并不支持 LocalPV，因此我们采用了自定义的第三方资源接口，自己实现了类似于如今生产环境中 Local Volume 的机制。

其流程为：我们静态地根据磁盘的资源创建 LocalPV，而在平台创建Kafka 集群时，动态地创建 LocalPVC。

此时调度器就可以根据其LocalPVC 和现有的 LocalPV 资源去创建 RC，然后在对应的节点上去启动 Broker。

当然，目前K8S 已有了类似的实现方式，大家可以直接使用了。

# 容器与 HBase 融合

![image-20210420092551424](https://gitee.com/AiShiYuShiJiePingXing/img/raw/master/img/image-20210420092551424.png)

我们的另一个实践案例是将 HBase 平台放到了容器之中。具体需求如下：

根据业务方去对HBase 集群予以隔离。

由于HBase 的读写都是发生在 Region Server 节点上，因此需要对 Region Server 予以限制。

由于存在着大、小业务方的不同，因此我们需要对资源利用率予以优化。

同时，由于数据都是被存放到 HDFS 上之后，再加载到内存之中，因此我们可以在内存里通过 Cache 进行高性能的读写操作。

可见，RegionServer 的性能瓶颈取决于 CPU 与内存的开销。

![image-20210420092609303](https://gitee.com/AiShiYuShiJiePingXing/img/raw/master/img/image-20210420092609303.png)

因此我们将每个HBase Cluster 都放到了 Kubernetes 的 Namespace 里，然后对HBase Master 和 Region Server 都采用 Deployment 部署到容器中。

其中 Master 较为简单，只需通过Replication 来实现高可用性；而 Region Server 则针对大、小业务方进行了资源限制。

例如：我们给大的业务方分配了 8 核 + 64G 内存，而给小的业务方只分配 2 核 + 16G 内存。

另外由于 ZooKeeper 和 HDFS 的负载较小，如果直接放入容器的话，则会涉及到持久化存储等复杂问题。

因此我们让所有的 HBase 集群共享相同的 ZooKeeper 和 HDFS 集群，以减少手工维护 ZooKeeper 和 HDFS 集群的开销。

# 容器与 SparkStreaming 融合

![image-20210420092621232](https://gitee.com/AiShiYuShiJiePingXing/img/raw/master/img/image-20210420092621232.png)

不同于那些需要大量地读写 HDFS 磁盘的 Map-Reduce 任务，Spark Streaming 是一种长驻型的任务。

它在调度上，不需要去优化处理大数据在网络传输中的开销，也不需要对 HDFS 数据做Locality。

然而，由于它被用来做实时处理计算，因此对机器的负载较为敏感。如果机器的负载太高，则会影响到它处理的“落地”时延。

而大数据处理集群的本身特点就是追求高吞吐率，因此我们需要将 Spark Streaming 从大数据处理的集群中隔离出来，然后将其放到在线业务的容器之中。

![image-20210420092631009](https://gitee.com/AiShiYuShiJiePingXing/img/raw/master/img/image-20210420092631009.png)

在具体实践上，由于当时尚无 Spark 2.3，因此我们自己动手将YARN 的集群放到了容器之中。

即：首先在Docker 里启动 YARN 的 Node Manager，将其注册到Resource Manager 之上，以组成一个在容器里运行的 YARN 集群。

然后我们再通过 Spark Submit 提交一项 Spark Streaming 任务到该集群处，让 Spark Streaming 的 Executor 能够运行在该容器之中。

如今，Spark 2.3 之后的版本，都能支持对于 Kubernetes 调度器的原生使用，大家再也不必使用 YARN，而可以直接通过Kubernetes 使得 Executor 运行在容器里了。

# 大数据平台的 DevOps 管理

![image-20210420092645287](https://gitee.com/AiShiYuShiJiePingXing/img/raw/master/img/image-20210420092645287.png)

在大数据平台的管理方面，我们践行了 DevOps 的思想。例如：我们自行研发了一个 PaaS 平台，以方便业务方直接在该平台上自助式地对资源进行申请、创建、使用、扩容、及管理。

根据DevOps 的思想，我们定位自己为工具开发的平台方，而非日常操作的运维方。

我们都通过该 PaaS 平台的交付，让业务方能够自行创建、重启、扩容集群。

此举的好处在于：

减少了沟通的成本，特别是在公司越来越大、业务方之间的沟通越来越复杂之时。

业务便捷且有保障，他们的任何扩容需求，都不需要联系我们，而直接可以在该平台上独立操作完成。

减轻了日常工作的负担，我们能够更加专注于技术本身，专注于如何将该平台的底层技术做得更好。

![image-20210420092657832](https://gitee.com/AiShiYuShiJiePingXing/img/raw/master/img/image-20210420092657832.png)

由于业务本身对于像 Kafka、HBase 之类系统的理解较为肤浅，因此我们需要将自己积累的对于集群的理解和经验，以一种专家的视角呈现给他们。

作为 DevOps 实践中的一环，我们在该大数据平台上提供了丰富的监控指标。

图中以 Kafka 为例，我们提供的监控指标包括 Topic Level、Broker Level，和Host Level。

可见，我们旨在将 Kafka 集群变成一个“白盒”，一旦发生故障，业务方就能直接通过我们所定制的报警阈值，在指标界面上清晰地看到各种异常，并能及时自行处理。