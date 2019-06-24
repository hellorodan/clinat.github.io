---
layout:     post
title:      "Kafka Streams 设计原理之线程模型"
subtitle:   " \"拓扑结构，线程模型，容错机制，分区分配，流任务处理\""
date:       2019-06-23 09:00:00
author:     "Clinat"
header-img: "img/post-bg-travel.jpg"
catalog: true
tags:
    - Kafka
---

> “本篇是Kafka Streams设计原理的第一部分”

## 拓扑结构

拓扑结构是一个有向无环图，是整个Kafka Streams 对数据进行处理的整个过程，如果该处理过程存在循环，会导致数据在处理过程中产生死循环。

其中多个topic的数据可以输入到相同的源节点，但是必须保证，输入到相同源节点的topic的key和value的类型是相同的。(之后还会有连接操作，注意区分该部分和连接操作的对topic要求的区别)

<img src="/img_post/KafkaStreamsThreadModel/thread_model0.png" style="zoom:50%" />



## 线程模型

### 线程模型

Kafka流处理的线程模型主要由三个部分组成：

1.流实例（KafkaStreams）：通常一个机器只运行一个流实例。

2.流线程（StreamThread）：一个流实例可以运行多个流线程。

3.流任务（StreamTask）：一个流线程可以运行多个流任务，其中流任务的数量是由输入主题的分区数量决定的。

流实例对应一个进程，同时一般情况下，同一台机器上只运行流处理应用的一个流实例。

<img src="/img_post/KafkaStreamsThreadModel/thread_model1.png" style="zoom:40%" />

可以由多个流实例来订阅相同的主题，Kafka Streams采用的是Kafka集群本身的订阅机制，所以会将各个实例中的消费者加入到同一个消费组中，各个实例负责的topic分区是不同的。

每个流实例中可以创建多个流线程，其中流线程中又可以创建多个流任务，针对只有一个输入主题的情况，每个流任务只能负责输入主题的一个分区。

对于每个流线程，都有一个与其对应的消费者和生产者，流线程的消费者会从流任务的所负责的主题分区中获取消息，并将消息保存在记录缓冲区中，每个流任务从记录缓冲区中获取消息，并进行处理，并交由流线程的生产者向输出主题中输出信息。其中消费者和生产者会负责流线程中所有的流任务的消息获取和消息发送工作。

<img src="/img_post/KafkaStreamsThreadModel/thread_model2.png" style="zoom:40%" />

对于多个输入主题的情况，每个流任务可以负责多个主题，但是只能负责每个主题的一个分区，所以流任务的数量不是程序决定的的，而是由主题的分区数量决定的。

此外，流实例中分配的流线程的数量可以是不同的，流线程的数量是由程序决定的。每个流线程中分配的流任务数量也可以是不同的。

<img src="/img_post/KafkaStreamsThreadModel/thread_model3.png" style="zoom:40%" />

### 容错机制

如果流线程2出现故障，容错机制会在流线程1中创建一个流任务2，用来处理之前分配给流线程2中流任务2处理的主题分区的数据，而不是直接将主题分区分配给流线程1的流任务。

流线程出现故障，会导致流线程中的消费者出现故障，进而触发再平衡操作，会将流任务分配给其他线程或实例，具体根据分配策略的具体分配方式。

<img src="/img_post/KafkaStreamsThreadModel/thread_model5.png" style="zoom:50%" />

### 分区分配

**1.分区分组器**

分区分组器主要的作用就是根据拓扑结构，生成流任务和主题分区之间的对应关系。这里应该注意，在对主题进行分组的时候，不是一个拓扑结构订阅的topic为一组，而是对拓扑结构进行分析，如下图，对于同一个拓扑结构可能会分成多个组。

<img src="/img_post/KafkaStreamsThreadModel/thread_model6.png" style="zoom:50%" />

如下图为Kafka Streams默认的分区分组器，主要通过partitionGroups方法完成对分区的分组默认的分区分组器的代码实现:

1.遍历拓扑结构中每个组

2.找到该组中主题的分区数最大的数量maxNumPartitions

3.因为每个流任务对一个一个分区，所以最少要maxNumPartitions个流任务（各个主题的分区数不完全相同，所以有些任务可能分不到某些主题的分区）的数据处理的流任务的数量便是该主题的分区数。然后通过组号和分区号形成流任务号。

<img src="/img_post/KafkaStreamsThreadModel/thread_model7.png" style="zoom:50%" />

由于各个topic的分区数量可能是不同的，所以有些流任务(一个分组)订阅了多个主题，可能分不到有些主题的分区。例如下图中TaskId4，该流任务便没有订阅到主题2的分区。

<img src="/img_post/KafkaStreamsThreadModel/thread_model8.png" style="zoom:50%" />

除了流任务，还有备份任务，备份任务的分组方式和流任务是类似的，因为流任务和备份任务是对应的。

<img src="/img_post/KafkaStreamsThreadModel/thread_model9.png" style="zoom:50%" />

其中流任务和备份任务是相对应的，同时备份任务的ID和其备份的流任务的ID是相同端的。同一个流任务中不能有相同ID的任务，即相同ID的流任务和备份任务也不能分配到同一个流线程中。

**2.再平衡监听器**

再平衡操作前，流线程调用监昕器的onPartitionsRevoked()方法，清空“活动的流任务集”和“分区与流任务的映射”。
再平衡操作后，流线程调用监昕器的onPartitionsAssigned()方法，根据最新分配的分区从分区分配器中获取任务编号，然后创建流任务， 并更新“活动的流任务集”和“分区与流任务的映射”。

**3.分区分配器**

涉及到消费者加入消费组的过程，该部分暂不分析，可以参考如下流程图。

<img src="/img_post/KafkaStreamsThreadModel/thread_model10.png" style="zoom:50%" />



## 流任务处理流程

流任务的处理流程采用的是深度优先的方式。

对于多个源节点的情况，流任务先根据各个分区中消息的时间戳，选择时间戳最小的数据进行处理，当该数据流经整个拓扑结构之后，再选择其他数据进行处理。

<img src="/img_post/KafkaStreamsThreadModel/thread_model11.png" style="zoom:35%" />

对于多个目标节点的情况，会先流经分支节点的一条分支，处理完之后再通过另一条分支进行处理。

<img src="/img_post/KafkaStreamsThreadModel/thread_model4.png" style="zoom:40%" />

