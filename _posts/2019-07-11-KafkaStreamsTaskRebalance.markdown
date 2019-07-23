---
layout:     post
title:      "Kafka Streams 再平衡流任务分配算法"
subtitle:   " \"StreamsPartitionAssginor类中assign方法源码分析\""
date:       2019-07-11 15:00:00
author:     "Clinat"
header-img: "img/post-bg-js-module.jpg"
catalog: true
tags:
    - Kafka
---

> “再平衡流任务分配算法中完成对流任务和备份任务的分配”



## 再平衡分配算法

再平衡分配算法的触发条件是属于同一个消费组的消费者增加或者减少时，就会触发再平衡操作，再平衡分配算法主要是完成流任务和备份任务在各个客户端的分配工作。

该算法的实现在StreamsPartitionAssginor类中的assign方法中。原本的Kafka集群中存在默认的消费组的分区分配算法的实现类，Kafka Streams为了满足流处理的需求，采用了StreamsPartitionAssginor类作为消费组的分区分配的实现类。



## 算法流程

该算法的主要流程如下图左侧，右侧为每个过程的子过程。

 ![assgin0](/img_post/KafkaStreamsTaskRebalance/assgin0.png)



##源码分析

Kafka Streams流任务分配算法的实现是在assign方法中实现的，该方法的代码长度比较长。

```java
public Map<String, Assignment> assign(final Cluster metadata,
                                          final Map<String, Subscription> subscriptions) {
        // construct the client metadata from the decoded subscription info
        final Map<UUID, ClientMetadata> clientsMetadata = new HashMap<>();
        final Set<String> futureConsumers = new HashSet<>();

        minReceivedMetadataVersion = SubscriptionInfo.LATEST_SUPPORTED_VERSION;
        supportedVersions.clear();
        int futureMetadataVersion = UNKNOWN;
  
  			//遍历订阅信息
        for (final Map.Entry<String, Subscription> entry : subscriptions.entrySet()) {
            final String consumerId = entry.getKey();
            final Subscription subscription = entry.getValue();
            final SubscriptionInfo info = SubscriptionInfo.decode(subscription.userData());
            final int usedVersion = info.version();
            supportedVersions.add(info.latestSupportedVersion());
            if (usedVersion > SubscriptionInfo.LATEST_SUPPORTED_VERSION) {
                futureMetadataVersion = usedVersion;
                futureConsumers.add(consumerId);
                continue;
            }
            if (usedVersion < minReceivedMetadataVersion) {
                minReceivedMetadataVersion = usedVersion;
            }

            // create the new client metadata if necessary
          	//根据订阅的信息生成每个Client的元信息
            ClientMetadata clientMetadata = clientsMetadata.get(info.processId());
						
          	//如果元信息为空，则创建了该Client的元信息，并将其添加到clientsMetadata中
            if (clientMetadata == null) {
                clientMetadata = new ClientMetadata(info.userEndPoint());
                clientsMetadata.put(info.processId(), clientMetadata);
            }

            // add the consumer to the client
          	//将consumer添加到其对应的Client的元信息中
            clientMetadata.addConsumer(consumerId, info);
        }

  			//一些版本的验证和日志的输出
        final boolean versionProbing;
        if (futureMetadataVersion != UNKNOWN) {
            if (minReceivedMetadataVersion >= EARLIEST_PROBEABLE_VERSION) {
                log.info("Received a future (version probing) subscription (version: {}). Sending empty assignment back (with supported version {}).",
                    futureMetadataVersion,
                    SubscriptionInfo.LATEST_SUPPORTED_VERSION);
                versionProbing = true;
            } else {
                throw new IllegalStateException("Received a future (version probing) subscription (version: " + futureMetadataVersion
                    + ") and an incompatible pre Kafka 2.0 subscription (version: " + minReceivedMetadataVersion + ") at the same time.");
            }
        } else {
            versionProbing = false;
        }

        if (minReceivedMetadataVersion < SubscriptionInfo.LATEST_SUPPORTED_VERSION) {
            log.info("Downgrading metadata to version {}. Latest supported version is {}.",
                minReceivedMetadataVersion,
                SubscriptionInfo.LATEST_SUPPORTED_VERSION);
        }

        log.debug("Constructed client metadata {} from the member subscriptions.", clientsMetadata);

        // ---------------- Step Zero ---------------- //

        // parse the topology to determine the repartition source topics,
        // making sure they are created with the number of partitions as
        // the maximum of the depending sub-topologies source topics' number of partitions
  			//解析整个的拓扑结构来确定重分区的源topic
  			//并且确保这些源topic能够正确被创建，并且和分区数和其依赖的子拓扑的源topic的分区数是相同的
        
  			//taskManager.builder().topicGroups()方法用于分析整个拓扑结构，形成多个子拓扑，其中每一个子拓扑中包含的源topic，目标topic以及状态变更日志topic都属于同一个group
  			final Map<Integer, InternalTopologyBuilder.TopicsInfo> topicGroups = taskManager.builder().topicGroups();
		
        final Map<String, InternalTopicMetadata> repartitionTopicMetadata = new HashMap<>();
  
  			//遍历topicGroups，即遍历每个子拓扑
        for (final InternalTopologyBuilder.TopicsInfo topicsInfo : topicGroups.values()) {
          	//遍历每个group的重分区源topic
            for (final InternalTopicConfig topic: topicsInfo.repartitionSourceTopics.values()) {
              //将重分区topic的信息记录到repartitionTopicMetadata中
                repartitionTopicMetadata.put(topic.name(), new InternalTopicMetadata(topic));
            }
        }

        boolean numPartitionsNeeded;
  			//可能一次循环不能确定所有的重分区topic的分区数，所以可能需要多次遍历
        do {
            numPartitionsNeeded = false;
						//遍历所有的group
            for (final InternalTopologyBuilder.TopicsInfo topicsInfo : topicGroups.values()) {
              //遍历每个group的重分区源topic
                for (final String topicName : topicsInfo.repartitionSourceTopics.keySet()) {
                    int numPartitions = repartitionTopicMetadata.get(topicName).numPartitions;

                    // try set the number of partitions for this repartition topic if it is not set yet
                  	//如果这个重分区topic的分区数没有确定，则尝试设置分区数
                    if (numPartitions == UNKNOWN) {
                      //遍历所有的group
                        for (final InternalTopologyBuilder.TopicsInfo otherTopicsInfo : topicGroups.values()) {
                            final Set<String> otherSinkTopics = otherTopicsInfo.sinkTopics;
                            if (otherSinkTopics.contains(topicName)) {
                                //如果重分区topic是这个group的目标topic
                                //则使用这个group中源topic中最大分区数作为这个重分区topic的分区数
                                for (final String sourceTopicName : otherTopicsInfo.sourceTopics) {
                                    final Integer numPartitionsCandidate;
                                    //这个源topic有可能是另一个重分区topic，所以需要判断
                                    if (repartitionTopicMetadata.containsKey(sourceTopicName)) {
                                        numPartitionsCandidate = repartitionTopicMetadata.get(sourceTopicName).numPartitions;
                                    } else {
                                        numPartitionsCandidate = metadata.partitionCountForTopic(sourceTopicName);
                                        if (numPartitionsCandidate == null) {
                                            repartitionTopicMetadata.get(topicName).numPartitions = NOT_AVAILABLE;
                                        }
                                    }

                                    if (numPartitionsCandidate != null && numPartitionsCandidate > numPartitions) {
                                        numPartitions = numPartitionsCandidate;
                                    }
                                }
                            }
                        }
                        //如果仍然没有确定这个重分区topic的分区数，则需要再进行一次循环
                        if (numPartitions == UNKNOWN) {
                            numPartitionsNeeded = true;
                        } else {
                            repartitionTopicMetadata.get(topicName).numPartitions = numPartitions;
                        }
                    }
                }
            }
        } while (numPartitionsNeeded);

  			//确保具有相同分区划分的topic具有相同的分区数，并且强制这些topic的分区编号是相同的
        ensureCopartitioning(taskManager.builder().copartitionGroups(), repartitionTopicMetadata, metadata);

        //创建重分区topic
        prepareTopic(repartitionTopicMetadata);

        // 将所有重分区topic的分区信息添加到allRepartitionTopicPartitions
        final Map<TopicPartition, PartitionInfo> allRepartitionTopicPartitions = new HashMap<>();
        for (final Map.Entry<String, InternalTopicMetadata> entry : repartitionTopicMetadata.entrySet()) {
            final String topic = entry.getKey();
            final int numPartitions = entry.getValue().numPartitions;

            for (int partition = 0; partition < numPartitions; partition++) {
                allRepartitionTopicPartitions.put(new TopicPartition(topic, partition),
                        new PartitionInfo(topic, partition, null, new Node[0], new Node[0]));
            }
        }
				//创建集群元信息
        final Cluster fullMetadata = metadata.withPartitions(allRepartitionTopicPartitions);
        taskManager.setClusterMetadata(fullMetadata);

        log.debug("Created repartition topics {} from the parsed topology.", allRepartitionTopicPartitions.values());

        // ---------------- Step One ---------------- //

        //获取所有的源topic，并将这些源topic按照group进行划分
        final Set<String> allSourceTopics = new HashSet<>();
        final Map<Integer, Set<String>> sourceTopicsByGroup = new HashMap<>();
        for (final Map.Entry<Integer, InternalTopologyBuilder.TopicsInfo> entry : topicGroups.entrySet()) {
            allSourceTopics.addAll(entry.getValue().sourceTopics);
            sourceTopicsByGroup.put(entry.getKey(), entry.getValue().sourceTopics);
        }
				//调用partitionGrouper.partitionGroups方法实现流任务的划分以及所负责的分区的分配
        final Map<TaskId, Set<TopicPartition>> partitionsForTask = partitionGrouper.partitionGroups(sourceTopicsByGroup, fullMetadata);

        //判断是否所有的partition都已经分配，并且保证没有partition被分配到两个流任务中
        final Set<TopicPartition> allAssignedPartitions = new HashSet<>();
        final Map<Integer, Set<TaskId>> tasksByTopicGroup = new HashMap<>();
        for (final Map.Entry<TaskId, Set<TopicPartition>> entry : partitionsForTask.entrySet()) {
            final Set<TopicPartition> partitions = entry.getValue();
            for (final TopicPartition partition : partitions) {
                if (allAssignedPartitions.contains(partition)) {
                    log.warn("Partition {} is assigned to more than one tasks: {}", partition, partitionsForTask);
                }
            }
            allAssignedPartitions.addAll(partitions);

            final TaskId id = entry.getKey();
            tasksByTopicGroup.computeIfAbsent(id.topicGroupId, k -> new HashSet<>()).add(id);
        }
        for (final String topic : allSourceTopics) {
            final List<PartitionInfo> partitionInfoList = fullMetadata.partitionsForTopic(topic);
            if (!partitionInfoList.isEmpty()) {
                for (final PartitionInfo partitionInfo : partitionInfoList) {
                    final TopicPartition partition = new TopicPartition(partitionInfo.topic(), partitionInfo.partition());
                    if (!allAssignedPartitions.contains(partition)) {
                        log.warn("Partition {} is not assigned to any tasks: {}"
                                 + " Possible causes of a partition not getting assigned"
                                 + " is that another topic defined in the topology has not been"
                                 + " created when starting your streams application,"
                                 + " resulting in no tasks created for this topology at all.", partition, partitionsForTask);
                    }
                }
            } else {
                log.warn("No partitions found for topic {}", topic);
            }
        }

        //将task添加到状态变更topic的订阅者中
        final Map<String, InternalTopicMetadata> changelogTopicMetadata = new HashMap<>()
  			//遍历所有的group
        for (final Map.Entry<Integer, InternalTopologyBuilder.TopicsInfo> entry : topicGroups.entrySet()) {
            final int topicGroupId = entry.getKey();
          	//获取每个group的状态变更topic
            final Map<String, InternalTopicConfig> stateChangelogTopics = entry.getValue().stateChangelogTopics;
						//遍历所有的状态变更topic
            for (final InternalTopicConfig topicConfig : stateChangelogTopics.values()) {
                //将变更日志topic的分区数设置为该组中所有task的最大编号加1，因为task的编号是从0开始的
                int numPartitions = UNKNOWN;
                if (tasksByTopicGroup.get(topicGroupId) != null) {
                    for (final TaskId task : tasksByTopicGroup.get(topicGroupId)) {
                        if (numPartitions < task.partition + 1)
                            numPartitions = task.partition + 1;
                    }
                    final InternalTopicMetadata topicMetadata = new InternalTopicMetadata(topicConfig);
                    topicMetadata.numPartitions = numPartitions;

                    changelogTopicMetadata.put(topicConfig.name(), topicMetadata);
                } else {
                    log.debug("No tasks found for topic group {}", topicGroupId);
                }
            }
        }

  			//创建状态变更topic
        prepareTopic(changelogTopicMetadata);

        log.debug("Created state changelog topics {} from the parsed topology.", changelogTopicMetadata.values());

        // ---------------- Step Two ---------------- //

        //将task分配给每个Client
        final Map<UUID, ClientState> states = new HashMap<>();
        for (final Map.Entry<UUID, ClientMetadata> entry : clientsMetadata.entrySet()) {
            states.put(entry.getKey(), entry.getValue().state);
        }

        log.debug("Assigning tasks {} to clients {} with number of replicas {}",
                partitionsForTask.keySet(), states, numStandbyReplicas);
				//创建StickyTaskAssignor对象，并调用其assign方法完成流任务的分配工作
        final StickyTaskAssignor<UUID> taskAssignor = new StickyTaskAssignor<>(states, partitionsForTask.keySet());
        taskAssignor.assign(numStandbyReplicas);

        log.info("Assigned tasks to clients as {}.", states);

        // ---------------- Step Three ---------------- //

        //构建所有topic的partition和host的对应关系
        final Map<HostInfo, Set<TopicPartition>> partitionsByHostState = new HashMap<>();
        if (minReceivedMetadataVersion == 2 || minReceivedMetadataVersion == 3) {
            for (final Map.Entry<UUID, ClientMetadata> entry : clientsMetadata.entrySet()) {
                final HostInfo hostInfo = entry.getValue().hostInfo;

                if (hostInfo != null) {
                    final Set<TopicPartition> topicPartitions = new HashSet<>();
                    final ClientState state = entry.getValue().state;

                    for (final TaskId id : state.activeTasks()) {
                        topicPartitions.addAll(partitionsForTask.get(id));
                    }

                    partitionsByHostState.put(hostInfo, topicPartitions);
                }
            }
        }
        taskManager.setPartitionsByHostState(partitionsByHostState);

  			//创建最后的流任务分配的返回结果
        final Map<String, Assignment> assignment;
        if (versionProbing) {
            assignment = versionProbingAssignment(clientsMetadata, partitionsForTask, partitionsByHostState, futureConsumers, minReceivedMetadataVersion);
        } else {
            assignment = computeNewAssignment(clientsMetadata, partitionsForTask, partitionsByHostState, minReceivedMetadataVersion);
        }

        return assignment;
    }
```

下面来看一下流任务分配的整个过程中比较关键的两个方法：

**分区分组（partitionGrouper.partitionGroups）**

该方法的主要工作是完成流任务ID的生成以及该流任务所负责处理的partition的分配，partitionGroups方法是在DefaultPartitionGrouper类中实现的，该方法的代码实现如下：

```java
public Map<TaskId, Set<TopicPartition>> partitionGroups(Map<Integer, Set<String>> topicGroups, Cluster metadata) {
        Map<TaskId, Set<TopicPartition>> groups = new HashMap<>();
				//遍历所有的group
        for (Map.Entry<Integer, Set<String>> entry : topicGroups.entrySet()) {
            Integer topicGroupId = entry.getKey();
            Set<String> topicGroup = entry.getValue();
						//获取该组中所有topic中最大的分区数
            int maxNumPartitions = maxNumPartitions(metadata, topicGroup);
						
          	//创建maxNumPartitions个流任务，以为每个流任务只能处理一个topic的一个主题
            for (int partitionId = 0; partitionId < maxNumPartitions; partitionId++) {
                Set<TopicPartition> group = new HashSet<>(topicGroup.size());
								//遍历该组中的所有topic，如果该topic有这个编号的分区，则分配给该task
                for (String topic : topicGroup) {
                    List<PartitionInfo> partitions = metadata.partitionsForTopic(topic);
                    if (partitionId < partitions.size()) {
                        group.add(new TopicPartition(topic, partitionId));
                    }
                }
              	//添加TaskId和处理的分区的对应关系
                groups.put(new TaskId(topicGroupId, partitionId), Collections.unmodifiableSet(group));
            }
        }

        return Collections.unmodifiableMap(groups);
    }
```



**流任务分配（taskAssignor.assign）**

该方法的主要工作是决定每个流任务以及备份任务运行在哪个Client上，assign方法是在StickyTaskAssignor类中实现的，代码实现如下：

```java
//该方法首先分配流任务，然后再分配备份任务
public void assign(final int numStandbyReplicas) {
        assignActive();
        assignStandby(numStandbyReplicas);
    }
		
		//分配备份任务
    private void assignStandby(final int numStandbyReplicas) {
        for (final TaskId taskId : taskIds) {
            for (int i = 0; i < numStandbyReplicas; i++) {
                final Set<ID> ids = findClientsWithoutAssignedTask(taskId);
                if (ids.isEmpty()) {
                    log.warn("Unable to assign {} of {} standby tasks for task [{}]. " +
                                     "There is not enough available capacity. You should " +
                                     "increase the number of threads and/or application instances " +
                                     "to maintain the requested number of standby replicas.",
                             numStandbyReplicas - i,
                             numStandbyReplicas, taskId);
                    break;
                }
                allocateTaskWithClientCandidates(taskId, ids, false);
            }
        }
    }

		//分配流任务
    private void assignActive() {
        final int totalCapacity = sumCapacity(clients.values());
        final int tasksPerThread = taskIds.size() / totalCapacity;
        final Set<TaskId> assigned = new HashSet<>();

        //首先将之前存在的流任务分配给之前该任务所在的Client
      	//遍历之前的流任务分配信息
        for (final Map.Entry<TaskId, ID> entry : previousActiveTaskAssignment.entrySet()) {
            final TaskId taskId = entry.getKey();
          	//如果当前分配的流任务列表中存在该流任务(之前的流任务)
            if (taskIds.contains(taskId)) {
              	//获取该任务之前分配的Client
                final ClientState client = clients.get(entry.getValue());
              	//判断该Client是否还有可分配流任务的空间
                if (client.hasUnfulfilledQuota(tasksPerThread)) {
                  	//将该流任务分配给该Client
                    assignTaskToClient(assigned, taskId, client);
                }
            }
        }
				
      	//从所有需要分配的task中去除上一步中已经分配的task
        final Set<TaskId> unassigned = new HashSet<>(taskIds);
        unassigned.removeAll(assigned);

        //遍历所有没有分配的流任务
        for (final Iterator<TaskId> iterator = unassigned.iterator(); iterator.hasNext(); ) {
            final TaskId taskId = iterator.next();
          	//获取当前流任务的备份任务之前所在的Client列表，因为可能为多个备份任务，所以Client的数量也为多个
            final Set<ID> clientIds = previousStandbyTaskAssignment.get(taskId);
            if (clientIds != null) {
                for (final ID clientId : clientIds) {
                    final ClientState client = clients.get(clientId);
                  	//如果当前的Client没有分配满，则将该流任务分配给该Client
                    if (client.hasUnfulfilledQuota(tasksPerThread)) {
                        assignTaskToClient(assigned, taskId, client);
                        iterator.remove();
                        break;
                    }
                }
            }
        }

        //分配还没有被分配的流任务
        List<TaskId> sortedTasks = new ArrayList<>(unassigned);
        Collections.sort(sortedTasks);
        for (final TaskId taskId : sortedTasks) {
          //分配taskId
            allocateTaskWithClientCandidates(taskId, clients.keySet(), true);
        }
    }
		
    private void allocateTaskWithClientCandidates(final TaskId taskId, final Set<ID> clientsWithin, final boolean active) {
      //在clientWithin范围内找到合适的Client
        final ClientState client = findClient(taskId, clientsWithin, active);
        taskPairs.addPairs(taskId, client.assignedTasks());
      //将task分配给该Client
        client.assign(taskId, active);
    }

    private void assignTaskToClient(final Set<TaskId> assigned, final TaskId taskId, final ClientState client) {
        taskPairs.addPairs(taskId, client.assignedTasks());
        client.assign(taskId, true);
        assigned.add(taskId);
    }

    private Set<ID> findClientsWithoutAssignedTask(final TaskId taskId) {
        final Set<ID> clientIds = new HashSet<>();
        for (final Map.Entry<ID, ClientState> client : clients.entrySet()) {
            if (!client.getValue().hasAssignedTask(taskId)) {
                clientIds.add(client.getKey());
            }
        }
        return clientIds;
    }

private ClientState findClient(final TaskId taskId, final Set<ID> clientsWithin, boolean active) {

        //如果clientsWithin的大小为1，即只有1个Client，则直接选择该Client
        if (clientsWithin.size() == 1) {
            return clients.get(clientsWithin.iterator().next());
        }
				//找到之前分配该taskId流任务的Client
        final ClientState previous = findClientsWithPreviousAssignedTask(taskId, clientsWithin);
  			//如果之前 没有分配该taskId的流任务
        if (previous == null) {
          	//选择负载最小的Client进行分配
            return leastLoaded(taskId, clientsWithin, active);
        }
				//如果需要均衡负载
        if (shouldBalanceLoad(previous)) {
          	//先找到之前分配该taskId备份任务的Client
            final ClientState standby = findLeastLoadedClientWithPreviousStandByTask(taskId, clientsWithin);
          	//如果该taskId的备份任务没有分配，或者分配给standby需要平衡负载
            if (standby == null
                    || shouldBalanceLoad(standby)) {
              	//选择负责最低的Client进行分配
                return leastLoaded(taskId, clientsWithin, active);
            }
            return standby;
        }

        return previous;
    }

    private boolean shouldBalanceLoad(final ClientState client) {
        return client.reachedCapacity() && hasClientsWithMoreAvailableCapacity(client);
    }
		//判断是否有其他的Client比当前的Client有更多可分配的空间
    private boolean hasClientsWithMoreAvailableCapacity(final ClientState client) {
        for (ClientState clientState : clients.values()) {
            if (clientState.hasMoreAvailableCapacityThan(client)) {
                return true;
            }
        }
        return false;
    }

		//找到之前分配流任务的Client
    private ClientState findClientsWithPreviousAssignedTask(final TaskId taskId,
                                                                    final Set<ID> clientsWithin) {
        final ID previous = previousActiveTaskAssignment.get(taskId);
      	//如果之前分配该流任务的Client存在，并且在本次分配的范围之内，则返回该Client
        if (previous != null && clientsWithin.contains(previous)) {
            return clients.get(previous);
        }
      	//返回之前分配备份任务中，负载最小的Client
        return findLeastLoadedClientWithPreviousStandByTask(taskId, clientsWithin);
    }

    private ClientState findLeastLoadedClientWithPreviousStandByTask(final TaskId taskId, final Set<ID> clientsWithin) {
      	//获取之前分配备份任务的Client的集合
        final Set<ID> ids = previousStandbyTaskAssignment.get(taskId);
        if (ids == null) {
            return null;
        }
        final HashSet<ID> constrainTo = new HashSet<>(ids);
        constrainTo.retainAll(clientsWithin);
      	//返回负载最小的Client
        return leastLoaded(taskId, constrainTo, false);
    }

		//获取负载最小的Client
    private ClientState leastLoaded(final TaskId taskId, final Set<ID> clientIds, final boolean active) {
      	//先保证每个Client不分配同一组的task的情况下寻找负载最小的Client
        final ClientState leastLoaded = findLeastLoaded(taskId, clientIds, true, active);
      	//如果没有找到，再不保证Client不分配同一组task的情况下寻找负载最小的Client
        if (leastLoaded == null) {
            return findLeastLoaded(taskId, clientIds, false, active);
        }
        return leastLoaded;
    }

    private ClientState findLeastLoaded(final TaskId taskId,
                                        final Set<ID> clientIds,
                                        final boolean checkTaskPairs,
                                        final boolean active) {
        ClientState leastLoaded = null;
        for (final ID id : clientIds) {
            final ClientState client = clients.get(id);
            if (client.assignedTaskCount() == 0) {
                return client;
            }

            if (leastLoaded == null || client.hasMoreAvailableCapacityThan(leastLoaded)) {
                if (!checkTaskPairs) {
                    leastLoaded = client;
                } else if (taskPairs.hasNewPair(taskId, client.assignedTasks(), active)) {
                    leastLoaded = client;
                }
            }

        }
        return leastLoaded;

    }

```











