---
layout:     post
title:      "Kafka Streams Process API"
subtitle:   " \"Kafka Streams 低级API的使用方式\""
date:       2019-06-21 17:00:00
author:     "Clinat"
header-img: "img/post-bg-alitrip.jpg"
catalog: true
tags:
    - Kafka

---

> “Kafka Streams Process API”

## 处理器API使用示例

处理器API的使用流程：先创建拓扑对象topology，然后调用addSource添加源节点，addProcessor创建处理器节点，addSink创建接收器节点，然后创建流实例，启动流实例。

```java
public class Test {

    public static void main(String[] args) throws Exception{
        Properties properties = new Properties();
        properties.put(StreamsConfig.APPLICATION_ID_CONFIG,"streams-pipe");
        properties.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG,"10.20.51.195:9092");
        properties.put(StreamsConfig.DEFAULT_KEY_SERDE_CLASS_CONFIG, Serdes.String().getClass());
        properties.put(StreamsConfig.DEFAULT_VALUE_SERDE_CLASS_CONFIG,Serdes.String().getClass());

        Topology topology = new Topology();
        topology.addSource(Topology.AutoOffsetReset.LATEST,"Source",
                new UsePreviousTimeOnInvalidTimestamp(),
                Serdes.String().deserializer(),Serdes.String().deserializer(),
                "stream-input1");
        topology.addProcessor("Process1",new MyProcessorSupplier(),"Source");
        topology.addProcessor("Process2",() -> new MyProcessor(),"Source");
        topology.addSink("Sink","streams-output1",
                Serdes.String().serializer(),Serdes.String().serializer(),"Process");

        KafkaStreams streams = new KafkaStreams(topology,properties);
        streams.start();
    }
}
class MyProcessorSupplier implements ProcessorSupplier<String,String>{
    @Override
    public Processor<String, String> get() {
        return null;
    }
}

class MyProcessor extends AbstractProcessor<String,String>{
    @Override
    public void process(String key, String value) {

    }
}
```

### 添加源节点

调用addSource方法添加源节点。

```java
topology.addSource(Topology.AutoOffsetReset.LATEST,"Source",
                new UsePreviousTimeOnInvalidTimestamp(),
                Serdes.String().deserializer(),Serdes.String().deserializer(),
                "stream-input1");
```

各个参数依次为：位移重置策略，源节点名称，时间戳提取器，键反序列化器，值反序列化器，输入主题名称。

### 添加处理器

调用addProcessor方法添加处理器节点。

```java
topology.addProcessor("Process1",new MyProcessorSupplier(),"Source");
topology.addProcessor("Process2",() -> new MyProcessor(),"Source");
```

各个参数依次为：处理器节点名称，MyProcessorSupplier对象，父节点名称。

MyProcessorSupplier对象中只有一个get方法返回处理器对象。

### 添加接收器

调用addSink方法添加接收器节点。

```java
topology.addSink("Sink","streams-output1",
                Serdes.String().serializer(),Serdes.String().serializer(),"Process");
```

各个参数依次为：接收器节点名称，输出主题名称，键序列化器，值序列化器，父节点名称。



## 处理器构建

### init方法

通常情况下，可以使用AbstractProcessor类中的init方法，但是如果需要使用状态存储或者需要实现定时任务则需要重写init方法。

```java
class MyProcessor extends AbstractProcessor<String,String>{

    private KeyValueStore<String, String> keyValueStore;

    @Override
    public void init(ProcessorContext processorContext){
        super.init(processorContext);
        keyValueStore = (KeyValueStore) context().getStateStore("store-name");
        MyPunctuator punctuator = new MyPunctuator();
        context().schedule(10000, PunctuationType.WALL_CLOCK_TIME, punctuator);
    }

    @Override
    public void process(String key, String value) {
				context().forward(key,value);
    }
}

class MyPunctuator implements Punctuator{

    @Override
    public void punctuate(long timestamp) {

    }
}

```

对于定时任务的实现，首先需要实现Punctuator接口，并且调用context().schedule()方法，该方法会定时调用Punctuator实例的punctuate方法。

其中需要传入枚举类型PunctuationType.WALL_CLOCK_TIME，或者PunctuationType.STREAM_TIME，两者的区别是前者是根据具体的时间，后者只有在消息到达之后，提取消息时间戳，判断是否应该执行定时方法。即一个是时间驱动，一个是数据驱动。

### process方法

该方法是数据处理的具体位置。

可以调用context().forward()方法向下游节点传递数据。

## 组合处理器

所谓的组合处理器不是一种特殊的处理器，而是使用之前的方法实现高级API中连接操作的实现方式。

```java
KeyValueBytesStoreSupplier storeSupplier = Stores.persistentKeyValueStore(TUPLE_STORE_NAME);
StoreBuilder<KeyValueStore<String, Tuple<List<ClickEvent>, List<StockTransaction>>>> storeBuilder =
				Stores.keyValueStoreBuilder(storeSupplier,
                            Serdes.String(),
                            eventPerformanceTuple).withLoggingEnabled(changeLogConfigs);

topology.addSource("Txn-Source", stringDeserializer, stockTransactionDeserializer, "stock-transactions")
				.addSource("Events-Source", stringDeserializer, clickEventDeserializer, "events")
				.addProcessor("Txn-Processor", StockTransactionProcessor::new, "Txn-Source")
				.addProcessor("Events-Processor", ClickEventProcessor::new, "Events-Source")
				.addProcessor("CoGrouping-Processor", CogroupingProcessor::new, "Txn-Processor", "Events-Processor")
				.addStateStore(storeBuilder, "CoGrouping-Processor")
				.addSink("Tuple-Sink", "cogrouped-results", stringSerializer, tupleSerializer, "CoGrouping-Processor");
```

首先定义两个处理器，分别负责将两个输入主题中的数据转发给组合处理器，其中组合处理器需要用到本地存储，所以需要先创建本地存储。

```java
public class ClickEventProcessor extends AbstractProcessor<String, ClickEvent> {
    @Override
    public void process(String key, ClickEvent clickEvent) {
        if (key != null) {
            Tuple<ClickEvent, StockTransaction> tuple = Tuple.of(clickEvent, null);
            context().forward(key, tuple);
        }
    }
}

public class StockTransactionProcessor extends AbstractProcessor<String, StockTransaction> {
    @Override
    public void process(String key, StockTransaction value) {
        if (key != null) {
            Tuple<ClickEvent,StockTransaction> tuple = Tuple.of(null, value);
            context().forward(key, tuple);
        }
    }
}
```

这两个转发处理器的实现非常简单，只是将接收到的数据存入Tuple中，并转发给组合处理器，该类是自定义的类（元组）。

```java
public class CogroupingProcessor extends AbstractProcessor<String, Tuple<ClickEvent,StockTransaction>> {

    private KeyValueStore<String, Tuple<List<ClickEvent>,List<StockTransaction>>> tupleStore;
    public static final  String TUPLE_STORE_NAME = "tupleCoGroupStore";


    @Override
    @SuppressWarnings("unchecked")
    public void init(ProcessorContext context) {
        super.init(context);
        tupleStore = (KeyValueStore) context().getStateStore(TUPLE_STORE_NAME);
        CogroupingPunctuator punctuator = new CogroupingPunctuator(tupleStore, context());
        context().schedule(15000L, STREAM_TIME, punctuator);
    }

    @Override
    public void process(String key, Tuple<ClickEvent, StockTransaction> value) {

        Tuple<List<ClickEvent>, List<StockTransaction>> cogroupedTuple = tupleStore.get(key);
        if (cogroupedTuple == null) {
             cogroupedTuple = Tuple.of(new ArrayList<>(), new ArrayList<>());
        }
        if(value._1 != null) {
            cogroupedTuple._1.add(value._1);
        }
        if(value._2 != null) {
            cogroupedTuple._2.add(value._2);
        }
        tupleStore.put(key, cogroupedTuple);
    }
}

```

组合处理器接到数据之后，先根据key获取本地存储的数据，然后将接受到的数据根据key添加到指定的集合中。

```java
public class CogroupingPunctuator implements Punctuator {

    private final KeyValueStore<String, Tuple<List<ClickEvent>, List<StockTransaction>>> tupleStore;
    private final ProcessorContext context;

    public CogroupingPunctuator(KeyValueStore<String, Tuple<List<ClickEvent>, List<StockTransaction>>> tupleStore, ProcessorContext context) {
        this.tupleStore = tupleStore;
        this.context = context;
    }

    @Override
    public void punctuate(long timestamp) {
        KeyValueIterator<String, Tuple<List<ClickEvent>, List<StockTransaction>>> iterator = tupleStore.all();

        while (iterator.hasNext()) {
            KeyValue<String, Tuple<List<ClickEvent>, List<StockTransaction>>> cogrouped = iterator.next();
            // if either list contains values forward results
            if (cogrouped.value != null && (!cogrouped.value._1.isEmpty() || !cogrouped.value._2.isEmpty())) {
                List<ClickEvent> clickEvents = new ArrayList<>(cogrouped.value._1);
                List<StockTransaction> stockTransactions = new ArrayList<>(cogrouped.value._2);

                context.forward(cogrouped.key, Tuple.of(clickEvents, stockTransactions));
                // empty out the current cogrouped results
                cogrouped.value._1.clear();
                cogrouped.value._2.clear();
                tupleStore.put(cogrouped.key, cogrouped.value);
            }
        }
        iterator.close();
    }
}
```

最后，周期调用定时方法，该方法会轮询本地存储的状态，判断是否有数据，如果有则将两个输入主题输入的数据组合成元组，并发送给下游节点。