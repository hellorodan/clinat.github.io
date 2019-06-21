---
layout:     post
title:      "Kafka Streams DSL"
subtitle:   " \"Kafka Streams 高级API的使用方式\""
date:       2019-06-21 14:30:00
author:     "Clinat"
header-img: "img/post-bg-unix-linux.jpg"
catalog: true
tags:
    - Kafka

---

> Kafka Streams DSL

## Kafka Streams DSL使用示例

特点：允许使用java8的Lambda表达式，内置了很多流处理的算子操作，区分记录流和变更流，支持交互式查询，能够查询本地状态和全局状态，支持连接操作和窗口操作。

而本质上底层调用的还是低级的处理器API。

```java
public class Pipe {

    public static void main(String[] args) throws Exception{

        Properties properties = new Properties();
        properties.put(StreamsConfig.APPLICATION_ID_CONFIG,"streams-pipe");
        properties.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG,"10.20.51.195:9092");
        properties.put(StreamsConfig.DEFAULT_KEY_SERDE_CLASS_CONFIG, Serdes.String().getClass());
     properties.put(StreamsConfig.DEFAULT_VALUE_SERDE_CLASS_CONFIG,Serdes.String().getClass());

        final StreamsBuilder builder = new StreamsBuilder();
        KStream<String ,String> source = builder.stream("streams-plaintext-input");
        KStream<String,String> st = source.mapValues(s -> s.toUpperCase());
        st.to("test");
        final Topology topology = builder.build();
        final KafkaStreams streams = new KafkaStreams(topology,properties);
        streams.start();

    }
}
```

### Kafka Streams配置

Kafka Streams可以配置非常多的参数，但是在启动Kafka Streams程序时有两个参数时必须配置的：

1.APPLICATION_ID_CONFIG：用于标识Kafka Streams应用程序，在一个集群中改配置必须唯一，用来管理从同一个主题消费的一组消费者成员。

2.BOOTSTRAP_SERVERS_CONFUG：可以是单个 主机名:端口 对，也可以是多个（逗号隔开），用于指向Kafka集群。

```java
Properties properties = new Properties();
        properties.put(StreamsConfig.APPLICATION_ID_CONFIG,"streams-pipe");
        properties.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG,"10.20.51.195:9092");
        properties.put(StreamsConfig.DEFAULT_KEY_SERDE_CLASS_CONFIG, Serdes.String().getClass());
        properties.put(StreamsConfig.DEFAULT_VALUE_SERDE_CLASS_CONFIG,Serdes.String().getClass());
```

### 拓扑构建

整个流处理的拓扑结构主要由源处理器，处理节点，接收器处理器组成。

1.源处理器：主要用于从输入主题中消费数据，并将数据转发给下游节点。

2.处理节点：用户自定义数据处理逻辑，并将数据转发给下游节点。

3.接收器处理器：将接收到的数据发送给只指定的接收主题。

```java
final StreamsBuilder builder = new StreamsBuilder();
        KStream<String ,String> source = builder.stream("streams-plaintext-input");
        KStream<String,String> st = source.mapValues(s -> s.toUpperCase());
        st.to("test");
        final Topology topology = builder.build();
        final KafkaStreams streams = new KafkaStreams(topology,properties);
        streams.start();
```

拓扑结构的构建提供了fluent编程风格，但是每次操作返回的不是原本的对象，而是修改之后的新的对象。 

```java
KStream<String, String> source = builder.stream("streams-plaintext-input");
        source.flatMapValues(value -> Arrays.asList(value.toLowerCase(Locale.getDefault()).split("\\W+")))
              .groupBy((key, value) -> value)
              .count(Materialized.<String, Long, KeyValueStore<Bytes, byte[]>>as("counts-store"))
              .toStream()
              .to("streams-wordcount-output", Produced.with(Serdes.String(), Serdes.Long());
```

### 创建Serde

该对象是用来对消息进行序列化和反序列化操作。Kafka Streams中，Serdes类提供了创建Serde对象的便捷方法。

```java
Serdes<String> stringSerde = Serdes.String();
KStream<String,String> source = builder.stream("src-topic",Consumed.with(stringSerde,stringSerde));
source.to("output-topic",Produced.with(stringSerde,stringSerde));
```

创建自定义Serde：

首先自定义序列化器和反序列化器，分别实现Serializer接口和Deserializer接口，然后调用Serdes类的方法创建自定义Serde。

```java
class JsonSerializer<T> implements Serializer<T>{
		......
}
class JsonDeserialier<T> implements Deserializer<T> {
    ......
}

JsonSerializer<Object> serializer = new JsonSerializer<Object>();
JsonDeserialier<Object> deserialier = new JsonDeserialier<Object>();
Serde<Object> serde = Serdes.serdeFrom(serializer,deserialier);
```



## 状态存储

流的某些操作需要根据之前数据的信息进行相应的操作，所以需要用到状态存储。

可以通过调用KStream类中提供的具有状态操作的方法进行有关状态的操作。

```java
public class MyTransformer implements ValueTransformer<Object,Object> {
    @Override
    public void init(ProcessorContext context) {
				//进行转化器的初始化操作
      	/*
        	可以通过
          stateStore = (KeyValueStore)context.getStateStore(storeName);
          获取本地的状态存储。
        */
    }

    @Override
    public Object transform(Object value) {
      	//具体数据转换操作的实现位置。
      	//状态是采用键值对的方式存储的，通过stateStore.get(key)获取状态，stateStore.put(key,value)保存状态。
        return null;
    }

    @Override
    public void close() {

    }
}
ValueTransformerSupplier<String,Object> s = () -> new MyTransformer();
KStream<String,Object> st1 = source.transformValues(s,"state-name");
```

本示例中，transformValues方法和mapValues方法的作用是相同的，只是后者没有提供本地状态存储和获取的功能。

### 重分区操作

如果生产者将消息发送到输入主题时没有指定消息的键，则按照Kafka默认的分区方式，会采用轮询的方式将消息均匀的发送的主题的各个分区中。由于每个流任务只能处理一个分区，也就说对于没有键值的数据，可能会需要查找多个状态存储。

可以调用KStream类的through()方法对数据进行重新分区。

```java
source.through("topic",Produced.with(stringSerde,stringSerde),new DefaultPartitioner());
```

Kafka Streams中提供了非常多的能够产生一个新键的方法（selectKey，map，transform等），一旦调用这些方法，内部会有一个boolean值会设置为true，之后调用join，reduce或者聚合操作时就会自动进行分区操作。

### 创建本地存储

注意，在使用本地存储之前，要在拓扑中创建本地存储，代码如下：

```java
final StreamsBuilder builder = new StreamsBuilder();

String storeName = "state-name";//本地存储的名称
//通过静态方法创建本地存储，这里有多种创建本地存储的方法，能够创建不同类型本地存储。
KeyValueBytesStoreSupplier ss = Stores.inMemoryKeyValueStore(storeName);
StoreBuilder<KeyValueStore<String, String>> storeBuilder = Stores.keyValueStoreBuilder(ss, Serdes.String(), Serdes.String());
builder.addStateStore(storeBuilder);//添加到拓扑结构中。

KStream<String ,String> source = builder.stream("streams-plaintext-input");
ValueTransformerSupplier<String,Object> s = () -> new MyTransformer();
KStream<String,Object> st1 = source.transformValues(s,storeName);
st1.to("test");

final Topology topology = builder.build();
```

### 连接操作

连接操作，首先需要创建ValueJoiner<V1,V2,R>实例，并实现该接口中apply()方法，该方法主要用于根据从两个流中获取的对象，生成另一个输出对象，这里需要指定窗口大小（只对一定时间范围内的数据进行连接操作），窗口会在后面介绍。

```java
public class MyJoiner implements ValueJoiner<String,String,Object>{

    @Override
    public Object apply(String value1, String value2) {
        return null;
    }
}

KStream<String ,String> stream1 = builder.stream("streams-plaintext-input1");
KStream<String ,String> stream2 = builder.stream("streams-plaintext-input2");
ValueJoiner<String,String,Object> joiner = new MyJoiner();
JoinWindows windows = JoinWindows.of(60*1000*20);
KStream<String,Object> joinedStream = 
      stream1.join(stream2,joiner,windows, Joined.with(stringSerde,stringSerde,objectSerde));
```

注意：连接操作需要保证，连接的两个输入主题的分区数是一样的，并且生产者在向这两个主题发送数据时采用的分区器是一样的，保证相同键的消息会分配到相同编号的分区中，因为连接操作是对键相同的消息进行操作的。

### 连接窗口JoinWindow

连接窗口本质上是滑动窗口，是基于记录的时间戳，不是具体时间。

![image-20190621151713059](/img_post/joinwindow_doc.png)

只有KStream和KStream进行连接操作时才需要窗口操作，因为KTable的数据集是有限的，所以不需要窗口。

|                      | 左连接 | 内连接 | 外连接 |
| :------------------- | :----- | :----- | :----- |
| KTable-KTable        | √      | √      | √      |
| KStream-KTable       | √      | √      |        |
| KStream-KStream      | √      | √      | √      |
| KStream-GlobalKTable | √      | √      |        |

###  时间戳

时间戳主要用于连接流，变更日志以及决定Processor.punctuate()方法何时触发。

Kafka Streams除了一些自带的时间戳提取器，还可以自定义时间戳提取器：

```java
public class MyTimestampExtractor implements TimestampExtractor {

    @Override
    public long extract(ConsumerRecord<Object, Object> record, long previousTimestamp) {
        return null;
    }
}

props.put(StreamsConfig.DEFAULT_TIMESTAMP_EXTRACTOR_CLASS_CONFIG, MyTimestampExtractor.class);
//Consumed.with(Serdes.String(),Serdes.String()).withTimestampExtractor(new MyTimestampExtractor());
```



## KTable

KStream是事件流，KTable是变更流，KTable在接到数据之后，会首先保存到本地缓存中，然后在缓存中删除具有相同键的数据。在接到数据之后并不会直接向下游节点转发数据，只有在到达设置的转发时间或超过缓存限制之后才会向下游节点转发数据。

```java
StreamsBuilder builder = new StreamsBuilder();
KTable<String,String> table = builder.table("topic");
```

也可以通过聚合操作生成变更流：

```java
StreamsBuilder builder = new StreamsBuilder();
KTable<String, ShareVolume> shareVolume = builder.stream(STOCK_TRANSACTIONS_TOPIC,
                                                         Consumed.with(stringSerde, stockTransactionSerde)
                                                         .withOffsetResetPolicy(EARLIEST))
				.mapValues(st -> ShareVolume.newBuilder(st).build())
				.groupBy((k, v) -> v.getSymbol(), Serialized.with(stringSerde, shareVolumeSerde))
				.reduce(ShareVolume::sum);
```

在构建拓扑结构的过程中，经常会用到KStream.groupByKey()方法和KStream.groupBy()方法，这两个方法都会返回KGroupedStream对象，该对象只是分组操作的中间对象，并不能直接使用，需要通过聚合操作生成KTable对象。类似，KTable.groupBy()同样生成KGroupedTable对象，需要通过聚合操作，生成KTable对象。

其中，groupByKey方法和groupBy的区别在于：

1.GroupByKey方法适用于KStream已经有非空的keys，更重要的是，它不会设置重新分区的flag。

2.GroupBy方法假定你已经修改了keys，因此重新分区的flag会设为true。在调用GroupBy、joins、聚合等方法时会导致自动重新分区。

3.一般来说，应该尽可能优先选择GroupByKey而不是GroupBy。

上面代码部分中.withOffsetResetPolicy(EARLIEST)表示设置KStream或KTable的位移重置策略，EARLIEST是一个枚举类型，还有一个值为LATEST。

### 窗口操作

窗口主要包括会话窗口，翻转窗口，滑动/跳跃窗口。

```java
KTable<Windowed<TransactionSummary>, Long> customerTransactionCounts =
                 builder.stream(STOCK_TRANSACTIONS_TOPIC, Consumed.with(stringSerde, transactionSerde).withOffsetResetPolicy(LATEST))
                .groupBy((noKey, transaction) -> TransactionSummary.from(transaction),
                        Serialized.with(transactionKeySerde, transactionSerde))
                .windowedBy(SessionWindows.with(twentySeconds).until(fifteenMinutes)).count();
```

开窗操作是通过windowBy方法完成的，上面的例子中创建了一个会话窗口，其中不活跃时间为20s，保留时间为15min。

各个窗口的创建方式：

**会话窗口：**`SessionWindows.with(twentySeconds).until(fifteenMinutes)`

**翻转窗口：**`TimeWindows.of(twentySeconds).until(fifteenMinutes)`

**滑动/跳跃窗口：**`TimeWindows.of(twentySeconds).advancedBy(fiveSeconds).until(fifteenMinutes)`

### GlobalKTable

全局变更流是唯一的，GlobalKTable订阅主题的所有分区的数据都会流经全局变更流。

因为在进行流连接操作中，可能会指定一个新的类型作为连接操作所使用的键，此时流会进行分区操作，分区操作是有必要的，但也是具有额外开销的。

所以，如果需要连接的数据量不是很大，可以采用全局变更流作为连接对象，进而省略重新分区的过程。

```java
StreamsBuilder builder = new StreamsBuilder();
GlobalKTable<String, String> publicCompanies = builder.globalTable("company");
```

