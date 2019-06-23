---
layout:     post
title:      "Kafka Streams 设计原理之线程模型"
subtitle:   " \"拓扑结构，线程模型，容错机制，分区分配，流任务处理\""
date:       2019-06-23 09:00:00
author:     "Clinat"
header-img: "img/post-bg-2019.jpg"
catalog: true
tags:
    - Kafka
---

> “本篇是Kafka Streams设计原理的第一部分”

## 拓扑结构

拓扑结构是一个有向无环图，是整个Kafka Streams 对数据进行处理的整个过程，如果该处理过程存在循环，会导致数据在处理过程中产生死循环。

其中多个topic的数据可以输入到相同的源节点，但是必须保证，输入到相同源节点的topic的key和value的类型是相同的。(之后还会有连接操作，注意区分该部分和连接操作的对topic要求的区别)





## 线程模型





## 后记


