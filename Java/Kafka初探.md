# Kafka初探

本文档基于 [Apache Kafka官网](http://kafka.apache.org/) 进行翻译和整理，以梳理知识点，便于读者更快地了解与学习Kafka。

Kafka单节点部署步骤您可参考：[Kafka单节点部署](https://github.com/Sunxiai51/Note/blob/master/Java/Kafka%E5%8D%95%E8%8A%82%E7%82%B9%E9%83%A8%E7%BD%B2.md)

# 1. 定义与意义

> Kafka is used for building real-time data pipelines and streaming apps. It is horizontally scalable, fault-tolerant, wicked fast, and runs in production in thousands of companies.
>
> Kafka用于构建实时数据管道和流式应用程序。它水平可扩展、具备容错性、速度快成一匹野马，并被数以千计的公司运用于生产。

Kafka是一个*分布式流平台 (distributed streaming platform)*，它具备以下三项关键功能：

1. **发布和订阅记录流**。在这方面，它类似于消息队列或企业消息传递系统。
2. 以容错的方式**存储记录流**。
3. 在发生记录时**处理记录流**。

Kafka主要有以下两类应用场景：

1. 在系统或应用程序之间构建可靠的获取数据的实时流数据管道
2. 构建对数据流进行转换或响应的实时流式应用程序

# 2. 特性 

Kafka在一个或多个服务器上以集群方式运行，其中的记录流(streams of records)通过**topic**分类存储。每个记录(record)都由一个key、一个value和一个时间戳组成。

Kafka中，客户端和服务器之间使用TCP协议通信。此协议是版本控制的, 并与旧版本保持向后兼容性。

## 2.1 Topics and Logs

Topic是所发布记录的类别名称，一个topic可以有零至多个consumer订阅。

对于每个topic，Kafka群集维护一个如下的分区日志：

![img](http://kafka.apache.org/10/images/log_anatomy.png)

每个**partition**都是一个结构化的commit log，它有序且不可变，新产生的记录将追加到末尾。每个分区中的记录都分配了一个称为**offset**的连续id，id唯一标识分区内的每个记录。

Kafka集群使用可配置的retention period保留所有已发布的记录 (无论记录是否已被消费)。例如设置保留策略为两天，则记录发布后的两天内均可被消费，过期后将被丢弃以释放空间。数据量大小并不会影响Kafka的性能，因此Kafka集群可以长期存储数据。

事实上，Kafka仅在每个consumer中保留其在日志中的offset，而且offset能够由consumer自由控制。在读取记录时，consumer可以线性地提高offset，也可以按其喜好的任意顺序使用记录，例如重置为旧的offset以重新消费历史数据，或跳过最近的记录并从“现在”开始消费。

这些特性的组合让Kafka的consumer可以任意读取记录而不会对集群或其它consumer产生太大影响。例如，您可以使用命令行工具来“跟踪”任何topic的内容，而不会更改任何现有consumer的消费行为。示意图如下：

![img](http://kafka.apache.org/10/images/log_consumer.png)

对commit log进行分区的目的在于：首先，分区允许根据单个服务器承载能力控制日志规模。每个单独的partition受限于承载它的服务器，但一个topic可以有许多partition，因此topic可以处理任意数量的数据。其次，它们能够充当并行的单元，在下节详述。

## 2.2 Distribution

日志的partition分布在Kafka集群中的服务器上，这些服务器共享这些partition进行数据处理。每个partition都可在指定数量的服务器中备份以保证容错性。

每个partition都有一个"leader server"和零至多个"follower server"。leader处理partition的所有读写请求，follower作为leader的备份被动复制。如果leader宕机了，其中一个follower将自动成为新的leader。每个服务器都充当它的某些partition的领导者，并且成为其他partition的follower，因此可以有效地均衡负载。

## 2.3 Producer

生产者将数据发布到他们所选择的topic中，并负责选择将记录分配给topic中的哪个分区：可以简单地通过轮询来选择以均衡负载，也可以根据一些语义分区函数来选择(根据记录中的某些关键词)。More on the use of partitioning in a second!

## 2.4 Consumers

Consumer使用group name来标识自己，每个发布到topic的记录都将投递给每个订阅了该topic的**consumer group**中的一个consumer。Consumer可以运行在不同的进程或计算机上。

如果所有consumer都有相同的consumer group，则将有效地平衡记录产生的负载。

如果所有consumer都有不同的consumer group，则每个记录都将被广播到所有的consumer。

下图是一个包含两个Kafka服务器的集群，承载四个分区(P0-P3)与两个consumer group。Group A包含两个consumer，Group B包含四个consumer。

![img](http://kafka.apache.org/10/images/consumer-groups.png)

然而更常见的是，我们发现topic有较少的consumer group，每一个consumer group都代表一个逻辑上的订阅者。每个group由许多consumer组成以提供可伸缩性和容错能力。这无非是Pub/Sub模式——订阅者是集群而不是单个进程。

在Kafka中实现消费的方式是将日志的partition分配给consumer，这样每个consumer在任何时间点都是“公平共享分区”的独占使用者。此维护group成员身份的过程是由卡夫卡协议动态处理的。如果新consumer加入group，它们将从group的其他成员接管一些partition；如果consumer宕机，则其partition将分布到其余consumer。

卡夫卡只对partition内的记录排序而不会扩展到不同partition间。对于大多数应用程序来说，按partition排序和按键划分数据的能力是足够的，如果需要对记录进行全排序，可以使用只有一个partition的topic来实现，但这意味着每个group都只有一个consumer。

## 2.5 Guarantees

高级别的Kafka提供以下保证：

- 由producer发送到特定partition的消息将按发送顺序追加。即：如果记录M1由与记录M2相同的producer发送，并且首先发送M1，则M1的offset将比M2低，并在日志中较早显示。
- 每个consumer按记录在日志中的存储顺序查看。
- 对于具有复制因子N的topic，N-1的服务器故障不会丢失已提交到日志的任何记录。

