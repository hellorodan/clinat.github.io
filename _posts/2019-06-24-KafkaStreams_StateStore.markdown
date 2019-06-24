---
layout:     post
title:      "Kafka Streams 设计原理之状态存储"
subtitle:   " \"状态存储，状态工作流程\""
date:       2019-06-24 10:30:00
author:     "Clinat"
header-img: "img/post-bg-magic.jpg"
catalog: true
tags:
    - Kafka
---

> “本篇是Kafka Streams设计原理的第二部分”


## 状态存储

有状态的流任务会将状态信息存储在本地，同时将状态信息发送到kafka集群的内部主题中。

备份任务负责将备份的流任务的状态回复到本地。（只有有状态的流任务才有备份任务）

<img src="/img_post/KafkaStreamsStateStore/state_store0.png" style="zoom:40%">

本地状态存储的路径为  状态存储配置/应用程序名称/任务编号/rocksdb/状态存储名称

由于本地存储状态是根据任务编号来决定存储路径，同时同一个应用程序的存储位置是相同的，也就是说，在同一个流实例中，不能由相同的任务编号（包括流任务和备份任务），否则存储位置会产生冲突。

同时，当流任务关闭时，会在存储目录中生成检查点文件，检查点文件用于记录本地和变更日志主题中状态的同步位置。

<img src="/img_post/KafkaStreamsStateStore/state_store1.png" style="zoom:40%">



## 工作流程

**阶段一：**首先启动两个流实例，流实例1和流实例2，此时变更日志主题中已经有msg1和msg2两条状态信息，这里注意，每一个流任务的状态信息只存储在变更日志主题的一个partition，因为备份任务和流任务类似，只能处理topic的一个分区。

下图中，流实例1已经完成了状态同步，而流实例2还没有完成数据同步。流实例1中包括流任务T0和流任务T1的备份任务，其中流任务T0将状态信息同步到T3P0分区，备份任务T1负责同步T3P1分区的状态信息。流实例2还没有完成状态同步。

<img src="/img_post/KafkaStreamsStateStore/state_store2.png" style="zoom:40%">

**阶段二：**流实例2完成状态同步，流实例2中流任务T1从T3P1分区中同步状态信息到本地，备份任务T0从T3P0分区中同步状态信息到本地。

<img src="/img_post/KafkaStreamsStateStore/state_store3.png" style="zoom:40%">

**阶段三：**流任务T0处理消息msg3，将状态信息保存到本地，同时将状态信息发送到变更日志主题的P0分区中，此时备份任务T0会获取流任务T0发送到变更日志主题的状态信息，并同步到本地。

<img src="/img_post/KafkaStreamsStateStore/state_store4.png" style="zoom:40%">

**阶段四：**该部分与阶段三类似，流任务T1处理消息msg4，将状态信息保存到本地，同时向变更日志主题的P1分区发送状态变更信息，备份任务T1会获取流任务T1发送到变更日志主题的状态信息，并同步到本地。

<img src="/img_post/KafkaStreamsStateStore/state_store5.png" style="zoom:40%">

**阶段五：**关掉流实例1，流任务T0和备份任务T1会在本地生成检查点文件，用于记录当前本地状态和变更日志主题中的状态的同步位置。触发再平衡操作，并将流任务T0重新分配到流实例2中，由于流实例2之前运行过备份任务T0，所以流任务的相关状态在本地已经保存，所以可以快速恢复流任务T0的状态并开始执行。

<img src="/img_post/KafkaStreamsStateStore/state_store6.png" style="zoom:40%">

**阶段六：**重启流实例1，并关闭流实例2，此时触发再平衡操作，流任务T2分配到流实例1上，通过检查点文件进行状态恢复。因为关闭流实例1时生成的检查点文件记录的位置不是最新的位置，所以需要从该位置开始进行同步，同步到变更日志主题的最新位置。

<img src="/img_post/KafkaStreamsStateStore/state_store7.png" style="zoom:35%">

<img src="/img_post/KafkaStreamsStateStore/state_store8.png" style="zoom:35%">

**阶段七：**重启流实例2，再次触发再平衡操作，同样，流实例2需要通过检查点文件进行状态恢复。

<img src="/img_post/KafkaStreamsStateStore/state_store9.png" style="zoom:35%">

**阶段八：**再启动两个新的流实例，任务重新分配，备份任务被分配到新实例中。由于新的流实例中没有本地状态，也没有检查点文件，所以，新的流实例中的备份任务T0和T1会从变更日志主题的开始位置进行状态同步。

<img src="/img_post/KafkaStreamsStateStore/state_store10.png" style="zoom:40%">

**阶段九：**新实例中的备份任务T0和T1进行状态恢复，同时原来的流实例中也保留了之前的状态

<img src="/img_post/KafkaStreamsStateStore/state_store11.png" style="zoom:40%">






