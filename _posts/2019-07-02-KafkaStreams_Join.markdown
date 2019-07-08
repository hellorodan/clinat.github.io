---
layout:     post
title:      "Kafka Streams 设计原理之连接与窗口"
subtitle:   " \"连接操作，窗口操作\""
date:       2019-07-02 19:30:00
author:     "Clinat"
header-img: "img/post-bg-kuaidi.jpg"
catalog: true
tags:
    - Kafka
---

> “本篇是Kafka Streams设计原理的第四部分”


## 连接操作

连接操作是指将两个流的数据进行连接，使其成为另外一个流的数据源，如下图：

<img src="/img_post/KafkaStreamsJoinWindow/join0.png" style="zoom:50%">

上图展示的便是Stream1从topic1中读取数据，Stream2从topic2读取数据，通过join()将两个流的数据进行连接作为Stream3处理的数据，并将处理后的数据发送到topic3中。

这里需要区分的是连接操作和一个流从多个数据源(topic)中读取数据之间的区别。首先，一个流处理多个数据源要求每个作为数据源的topic中保存的消息的键值对的类型是相同的，两个topic中的数据相互之间没有关系，仅仅用来作为流处理数据的源头。而连接操作要求进行连接操作的两个topic中端的消息的键的类型是相同的，同时两个topic所拥有的partition的数量相同，向这两个topic发送数据的producer所采用的分区分配策略也是相同的，因为流的连接操作是根据key进行的连接，只有key相同的消息才能进行连接操作，所以需要key的类型相同；此外partition数量相同和使用相同的分区分配策略是为了保证相同key的消息发送到各个topic中相同编号的partition中，这是因为对于具有连接操作的拓扑结构，在对其进行分区分配时，会将这个连接操作的两个流作为一个组，即对于这个组的每个流任务在处理topic1和topic2时只能处理相同编号的分区，这样边可以保证具有相同key的消息会在同一个流任务中进行连接操作，如下图：

<img src="/img_post/KafkaStreamsJoinWindow/join8.png" style="zoom:45%">

Kafka Streams的连接操作氛围内连接，左连接和外链接。支持KStream连接KStream，KStream连接KTable，KTable连接KTable，但是不支持KTable连接KStream。其中KStream连接KStream必须使用窗口，这是为了解决KStream事件流所带来的无限数据规模的问题，而对于KTable的连接则不需要，因为KTable是变更流，记录的是消息的当前状态，所以数据的规模是有限的。

<img src="/img_post/KafkaStreamsJoinWindow/join1.png" style="zoom:45%">

上图展示的是模拟各种类型数据连接的结果，该图中的数据默认所有的消息的key均是相同的，即每个消息之间都可以进行连接，同时各个消息到达的时间均在时间窗口范围之内。

内连接：可以通俗理解为两边的数据进行连接操作时都不能为null。

左连接：左边的Stream去连接右边的Stream时，如果右边的Stream还没有数据，则可以和null进行连接。

外连接：两边都可以和null进行连接。

下图为连接操作的执行过程，了解执行过程之后就可以很好的理解上图的结果：

<img src="/img_post/KafkaStreamsJoinWindow/join2.png" style="zoom:50%">

首先，KStream1就收到消息，之后通过JoinWindow将该消息保存到本地状态存储中，之后发送给下游节点Join，Join会从KStream2的本地状态存储中读取消息，此时KStream2还没有处理消息，所以如果采用的连接方式为内连接，则无法进行连接即不进行处理，如果采用的是外连接或者左连接，则可以和null进行连接。

<img src="/img_post/KafkaStreamsJoinWindow/join3.png" style="zoom:50%">

此时，KStream2接收到消息，与之前是同样的过程，先将消息保存在本地状态中，然后由Join节点读取KStream1的本地状态，因为之前KStream1已经处理过消息，同时key是相同的，所以可以进行连接操作，并将连接结果作为之后流处理的数据。

这里在之前的学习过程中被问到为什么两个流可以读取到对方的状态存储信息：因为对于连接操作，在对拓扑结构进行分组时会分成一个组，即由一个流任务处理进行连接操作的两个流，两个流的状态存储其实保存的是在同一台机器上 ，所以是可以互相读取的。



## 窗口操作

在上面介绍连接操作的过程时，说到JoinWindow会将接收到的消息保存到本地，方便另外一个流查找消息进行连接操作。但是因为是流处理，数据是有一定的时效性，并且在进行连接操作时，也不能无限制的查找所有之前接收到的数据，这显然是不合理的，所以在定义连接窗口时可以指定窗口数据保存的时间，具体可以参考之前介绍Kafka Streams DSL的文章。

如下图，对窗口的消息进行保存的方式也是采用segment的方式进行保存的，窗口接收到消息之后，会将消息放置到指定的segment中，并且根据设置的窗口保存时间，将超时的segment进行删除，这样连接操作在进行消息查找时便不会查找所有接收到的数据。

<img src="/img_post/KafkaStreamsJoinWindow/join4.png" style="zoom:45%">

#### 跳跃窗口

跳跃窗口可以设置窗口大小以及跳跃间隔，如下图，跳跃窗口的最大的特点就是同一个消息可能属于同一个窗口。该窗口常用于计算一段时间消息的聚合值。

<img src="/img_post/KafkaStreamsJoinWindow/join5.png" style="zoom:45%">

#### 滚动窗口

滚动窗口和跳跃窗口的区别在于滚动窗口的跳跃间隔就是窗口的大小，这样对于每条消息只能属于一个窗口，如下图：

<img src="/img_post/KafkaStreamsJoinWindow/join6.png" style="zoom:45%">

#### 会话窗口

会话窗口可以设置不活跃时间，当消息到达之后，会判断该消息之前和之后的不活跃时间范围内是否有其他窗口，如果有其他消息，则这两条消息之间的部分作为一个窗口，如果没有其他窗口则该消息本身作为一个窗口。并且可以根据不同的key构建不同的窗口。该窗口一般用来统计类似活跃度的数据，例如用户A在一段时间内的点击，如果两次点击的时间间隔不大，则认为是在一个时间窗口内。

<img src="/img_post/KafkaStreamsJoinWindow/join7.png" style="zoom:45%">