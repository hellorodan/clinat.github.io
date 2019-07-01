---
layout:     post
title:      "Kafka Streams 设计原理之事件流和变更流"
subtitle:   " \"事件流，变更流，拓扑结构，聚合操作，重分区操作\""
date:       2019-07-01 16:10:00
author:     "Clinat"
header-img: "img/post-bg-rwd.jpg"
catalog: true
tags:
    - Kafka
---

> “本篇是Kafka Streams设计原理的第三部分”

**官方示例：统计单词数量**

下图展示的是Kafka官网提供的统计单词数量的Kafka Streams示例。

<img src="/img_post/KafkaStreamsKTable/ktable0.png" style="zoom:40%">

## 事件流和变更流

事件流（KStream）是最基本的数据流模型，每个节点处理完数据会将数据转发给下游节点。

变更流（KTable）会根据键记录最新值，会有状态存储，同时会将值的变更发送到变更日志主题中。

如下图所示，当变更流(KTable)接收到信息的消息之后，会现在本地找到具有相同键的数据，并更新本地存储状态和并将状态变更信息发送到变更日志主题中。同时将变更后的信息发送到下游节点。

<img src="/img_post/KafkaStreamsKTable/ktable1.png" style="zoom:40%">

但是变更流并不是接到消息之后便立即将状态变更之后的信息发送到下游节点。KTable可以设置缓存时间和缓存大小。主要流程如下图：

首先，当KTable接收到新的消息之后，会判断该变更流是否设置了缓：

如果没设置缓存，直接将新的状态保存到本地磁盘，并将变更后的状态发送到下游节点。

如果设置了缓存，将新接收到的信息保存到缓存中。如果达到了缓存的最大容量，或者达到了设置的刷新时间，则将缓存中的信息保存到本地，并将这些新的状态信息发送到下游节点。

<img src="/img_post/KafkaStreamsKTable/ktable2.png" style="zoom:45%">

其中，引入缓存的最主要原因就是为了减少写磁盘所带来的延时，缓存会自动更新刷新之前的这段时间的状态变更，只是将刷新之前这段时间的最后状态保存到磁盘中。

## 拓扑结构

下图展示的就是上面统计单词数量的拓扑结构，对于程序中调用的每个方法，其内部都会采用Kafka Streams的低级API添加相应的处理节点。

这里需要注意的是count是一个聚合函数，其中涉及到重分区操作。其中重分区的具体原理在下面介绍。此处在处理count方法时，是先添加了一个目标节点，将前面处理的数据按照map生成的key发送到中间主题的相应分区，然后添加源节点，从中间主题中读取数据，进行聚合操作。

<img src="/img_post/KafkaStreamsKTable/ktable4.png" style="zoom:45%">

下图比较形象的展示了该示例中数据流动的变化过程：

首先从源主题中读取数据，由于输入的数据是英文句子，即单词之间是有空格的；通过flatMapValues方法将输入的消息根据空格进行拆分并将拆分之后单词按照流的形式依次发送到下游节点；map方法接收到单词之后将其映射成key-value的形式；然后通过groupByKey方法对这些键值对按照key进行分组；之后调用count完成数量统计工作，具体聚合操作和重分区参考下面。

<img src="/img_post/KafkaStreamsKTable/ktable5.png" style="zoom:45%">

## 聚合操作

这里需要注意的是，groupByKey，groupBy等方法并不是真的对数据进行分组，只是根据当前的流类型生成相应的中间对象，如KGroupedStream，KGroupedTable。并通过这些对象调用聚合函数 (这两种对象的聚合函数的执行过程是不同的)，这两个对象的区别在于一个是通过事件流生成的，一个是通过变更流生成。

下图为通过KGroupedStream对象执行聚合函数的过程，这里以count方法为例：

<img src="/img_post/KafkaStreamsKTable/ktable6.png" style="zoom:45%">

执行过程大致为：接收到新消息之后，通过键，到本地的状态存储中查找相应的信息，如果没有则返回null，如果为null，则执行一个初始化的过程，将数量设置为0，并执行加法操作，然后将相加之后的操作保存到本地。如果查找本地不为null，则在查找的结果上直接相加，然后将结果保存到本地。

下图展示的是KGroupedTable对象执行聚合操作，相对于KGroupedStream的聚合操作要复杂一点。

<img src="/img_post/KafkaStreamsKTable/ktable7.png" style="zoom:45%">

上图展示的是执行聚合操作之前的一些操作过程：首先当变更流接收到新消息之后，会到本地查找本地的状态，如果本地存在该key的状态信息，则返回，否则返回null。这里注意KTable节点向下游节点传递的数据中value是Change对象，其中该对象包括两个值，即变更之后的新值和变更之前的旧值，前面为新值。在执行分组的过程中，会将这个key-value拆成两个键值对，如图，分为oldPair和newPair (如果旧值为null，则不生成oldPair)。并且oldPair中的新值为null，newPair中旧值为null，并将这些键值对发送到下游节点。

<img src="/img_post/KafkaStreamsKTable/ktable8.png" style="zoom:45%">

下面便是执行聚合操作的过程，如上图。左边对应的是没有oldPair的情况，右边对应的是既有oldPair也有newPair的情况。执行聚合操作的过程是，根据key从本地存储中读取状态信息，然后将读取的信息作为change对象的旧值，将读取的信息加上新值减去旧值作为change对象的新值。(因为是变更流，每个信息表示当前的状态，可以理解为先去除旧状态，然后添加新状态)。

下图为变更流执行聚合操作的例子：主要统计的是本地状态中各个颜色的数量。

<img src="/img_post/KafkaStreamsKTable/ktable9.png" style="zoom:45%">

## 重分区分区

流的内部有一个标志位，该标志位用于记录是否对流中的数据的key进行修改，如果修改了，该标志位便会设为true。当该标志位为true时，执行聚合操作时会自动执行重分区操作。

为什么执行重分区：

首先Kafka Streams对数据的处理是基于key进行的，聚合操作也是对相同的key进行聚合操作。由于Kafka Streams是完全基于Kafka的，所以对于订阅的topic中的数据，也是具有相同key的消息分配到相同的partition中。如果在流执行的过程中，没有对key进行修改，则相同key的消息一定会在相同的流任务中执行(每个流任务负责处理topic的一个partition)，所以此时执行聚合操作时并不需要重分区。但是如果修改了key，就可能导致具有相同key的消息在不同流任务中进行处理，这时如果不进行重分区便执行聚合操作便不能统计所有具有这个key的数据，所以执行聚合操作之前需要先执行重分区，使用produce发送消息时采用的消息分区分配算法将相同的key重新发送到相同的中间主题的相同分区，然后再执行聚合操作便能处理所有数据。如下图：

<img src="/img_post/KafkaStreamsKTable/ktable10.png" style="zoom:45%">

其中，在对拓扑结构进行分析时，重分区之前和重分区之后的部分会被分成不同的组，即形成不同拓扑结构的流任务并被分配到不同的流线程或者流实例中，如下图。

<img src="/img_post/KafkaStreamsKTable/ktable3.png" style="zoom:40%">















