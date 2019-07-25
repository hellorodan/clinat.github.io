---
layout:     post
title:      "ZooKeeper概述"
subtitle:   " \"ZooKeeper相关介绍和客户端使用\""
date:       2019-07-25 15:00:00
author:     "Clinat"
header-img: "img/post-bg-infinity.jpg"
catalog: true
tags:
    - ZooKeeper
---

> “ZooKeeper的相关介绍，安装过程以及客户端的使用”


## ZooKeeper概述

雅虎创建，基于 Google chubby

是一个开源的分布式协调框架，并不是用来存储数据的，通过监控数据状态的变化，达到基于数据的集群管理

zookeeper序列化使用的是Jute

### 使用场景

订阅发布：watcher机制，统一配置管理（disconf）

分布式锁：redis，zookeeper，数据库

负载均衡，ID生成器，分布式队列，统一命名服务，master选举

### 原则

**单一视图：**无论客户端连接到哪一个服务器，所看到的模型都是一样的

**原子性：**所有的事务请求的处理结果在整个集群中的所有机器上的应用情况是一致的。（所有机器上的分布式事务要么同时成功，要么同时失败）

**可靠性：**一旦服务器成功应用了某一个事务的数据，并且对客户端做了成功的 响应，那么这个数据在整个集群中一定是同步并且保留下来

**实时性：**一旦一个事务被成功应用，客户端就能够立即从服务器端读取到事务变更后的最新数据；（zookeeper仅仅保证在一定时间内实时，近实时）



## 安装（linux系统）

### 单机环境安装

- 下载zookeeper安装包：http://apache.fayea.com/zookeeper/stable/
- 解压安装包：`tar -zxvf zookeeper-3.4.12.tar.gz`
- cd 到 `zookeeper/conf`，（复制一份zoo.cfg）:`cp zoo_sample.cfg zoo.cfg`，zookeeper加载的配置文件为 `zoo.cfg`
- `sh zkServer.sh xxx`:`start|start-foreground|stop|restart|status|upgrade|print-cmd`执行服务器命令
- `sh zkCli.sh -server ip:port`：与zookeeper服务器建立连接，如果添加 `-server ip:port`内容，默认连接到本机的zookeeper节点

### 集群环境

zookeeper集群，包括三种角色：leader/follower/observer

observer 是一种特殊的zookeeper节点，可以帮助解决zookeeper的扩展性（如果大量客户端访问zookeeper集群，需要增加集群机器数量，从而增加zookeeper服务器性能，导致zookeeper写性能下降，因为，zookeeper的数据变更需要半数以上服务器投票通过，造成网络消耗增加投票成本）

- observer不参与投票，只接受投票结果。
- 不属于zookeeper的关键部位。

配置observer：在zoo.cfg 里面增加

​	peerType=observer

​	server.id=host:port:port:observer

安装：

- 修改配置文件 `zoo.cfg`（配置所有机器）

  server.id=host:port:port

  id的范围为1~255，用id表示该机器在集群中的机器序号

  第一个port是follow而与leader交换信息的端口号，第二个port是leader节点挂掉了，需要通过这个端口号进行leader选举。

  2181是zookeeper与客户端交换信息的端口号 ，所以前面两个端口号不能使用2181

- 创建myid（dataDir定义在 `zoo.cfg`文件中）

  在每一个服务器的dataDir目录下创建一个myid文件，数据内容是每台服务器对应的Server.id中id的值，该值配置在 `zoo.cfg`文件中

- 启动zookeeper，通过调用命令 `sh zkServer.sh start`启动服务器（zookeeper集群只有当两台及以上服务器启动后才会工作，这时候使用命令 `sh zkServer.sh status`可以查看当前服务器在zookeeper集群中的角色）



## zoo.cfg配置文件分析

- `tickTime=2000`：zookeeper中最小的事件单位，默认是2000毫秒。
- `initLimit=10`：follower启动之后，与leader服务器完成数据同步的时间。
- `syncLimit=5`：leader节点与follower节点进行心跳检测的最大延迟时间。
- `dataDir=/tmp/zookeeper`：表示zookeeper服务器存储快照文件的目录。
- `dataLogDir`：表示配置zookeeper事务日志的存储路径，默认指定在dataDir路径下。
- `clientPort=2181`：表示客户端与服务器端建立连接的端口号：2181。



## 相关概念

### 数据模型

zookeeper的数据模型和文件系统类似，每一个节点称为 znode，是zookeeper中最小的数据单元。每个znode上都可以保存数据和挂载子节点

​	持久化节点（`CreateMode.PERSISTENT`） ：节点创建后会一直存在zookeeper服务器上，直到主动删除

​	持久化有序节点（`CreateMode.PERSISTENT_SEQUENTIAL`）：节点会为有序子节点维护一个顺序

​	临时节点（`CreateMode.EPHEMERAL`）：临时节点的生命周期和客户端的会话保持一致，当客户端的会话失效，该节点自动清理 （临时节点不能拥有子节点）

​	临时有序节点（`CreateMode.EPHEMERAL_SEQUENTIAL`）：在临时节点上多一个顺序性特性（临时节点不能拥有子节点）

### 数据存储

zookeeper有三种日志：

zookeeper.out 运行日志

快照，存储某一时刻的全量数据

事务日志，事务操作的日志记录

### watcher

zookeeper提供了分布式数据发布/订阅，zookeeper允许客户端向服务器端注册一个watcher监听。当服务器端的节点触发指定事件的时候会触发watcher。服务端会向客户端发送一个事件通知

watcher的通知是一次性的，一旦触发一次通知后，该watcher就失效

注意在注册watcher之后，如果还需要继续监听，应该重新注册watcher，因为watcher是一次性有效，zookeeper中可以用来注册的方法有 `getData()`,`exists()`,`setData()`,`getChildren()` 方法，

##### 操作与产生的事件类型的对应关系

|                              | "/parent"的事件     | "/parent/child"的事件 |
| ---------------------------- | ------------------- | --------------------- |
| **create("/parent")**        | NodeCreated         |                       |
| **delete("/parent")**        | NodeDeleted         |                       |
| **setData("/parent")**       | NodeDataChanged     |                       |
| **create("/parent/child")**  | NodeChildrenChanged | NodeCreated           |
| **delete("/parent/child")**  | NodeChildrenChanged | NodeDeleted           |
| **setData("/parent/child")** | ——                  | NodeDataChanged       |

##### 事件类型与设置watcher的方法的对应关系

（watcher并不是监视所有类型的事件，其中default watcher是指创建ZooKeeper时注册的watcher）!

|                         | Default Watcher | exists("/path") | getData("/path") | getChildren("/path") |
| :---------------------- | --------------- | :-------------- | ---------------- | -------------------- |
| **None**                | Y               | Y               | Y                | Y                    |
| **NodeCreated**         |                 | Y               | Y                |                      |
| **NodeDeleted**         |                 | Y               | Y                |                      |
| **NodeDataChanged**     |                 | Y               | Y                |                      |
| **NodeChildrenChanged** |                 |                 |                  | Y                    |

### ACL

zookeeper提供了控制节点访问权限的功能，用于有效的保证zookeeper中的数据安全性，避免误操作而导致系统出现重大事故

CREATE/READ/WRITE/DELETE/ADMIN

- create：表示创建权限
- read：表示读权限
- write：表示写权限
- delete：表示删除权限
- admin：表示管理员权限

#### 权限控制模式

schema：授权对象

ip：192.168.1.1

Digest：username：password

world：开放式的权限控制模式，数据节点的访问权限对所有用户开放。world：anyone

super：超级用户，可以对zookeeper上的数据节点进行操作



## Server命令操作

- `create [-s] [-e] path data acl`：-s 表示节点是否有序，-e表示是否为临时节点，acl表示权限

- `get path [watch]`：获得指定path数据（信息）

- `set path data [version]`：修改节点对应的data

  数据库中有一个version字段去控制数据行的版本号，在执行该指令时，如果使用该字段，则version必须与当前节点的`dataversion`相同，如果不同，则设置数据失败。-1表示默认不使用version字段

- `delete path [version]`：删除指定path的节点，注意必须按照创建顺序的相反顺序删除，即先删除子节点，再删除父节点，其中version字段与 `set`指令的作用相同



## 节点state信息（get命令）

- `cversion = 0` ：子节点的版本号
- `dataVersion = 0 `：表示当前节点数据版本号
- `aclVersion = 0`：表示acl的版本号，修改节点权限
- `cZxid = 0x10000000a`：节点被创建时的事务ID
- `mZxid = 0x10000000d`：节点最后一次被更新的事务ID
- `pZxid = 0x10000000a`：当前节点下的子节点最后一次被修改时的事务ID
- `ctime = Fri Dec 21 23:57:12 PST 2018`：创建时间
- `mtime = Sat Dec 22 00:00:38 PST 2018`：修改时间
- `ephemeralOwner = 0x0`:创建临时节点的时候，会有一个sessionId，该值就是存储的sessionId
- `dataLength = 3`：数据长度
- `numChildren = 0`：子节点数



## zookeeper集群中的角色

leader

​	leader是zookeeper集群的核心

​	1.事务请求的唯一调度和处理者，保证集群事务处理的顺序性

​	2.集群内部各个服务器的调度者

follower

​	1.处理客户端非事务请求，以及转发事务请求给leader服务器

​	2.参与事务请求提议（proposal）的投票（客户端的一个事务请求，需要半数服务器投票通过以后才能通知leader commit；leader会发起一个提案，要follower投票）

​	3.参与leader选举的投票

observer

​	1.观察zookeeper集群中最新状态的变化并将这些状态同步到observer服务器上

​	2.增加observer不影响集群的事务处理能力，同时还能增加集群的非事务处理能力



## ZAB协议

zab协议为分布式协调服务zookeeper专门设计的一种支持崩溃回复和原子广播的协议（Zookeeper atomic broadcast）

### 消息广播

zookeeper使用主备模式的系统架构来保持数据在个副本之间的一致性。

当客户端向zookeeper集群写数据时，数据均是写入到leader服务器，然后由leader将数据复制到follower服务中，其中，在复制的过程中，只需要follower服务器中由半数以上的返回ack信息，zab协议就可以提交。

步骤：

- leader接受客户端的事务请求
- leader将数据复制到follower
- follower返回ack
- 如果数量超过一半，leader则执行commit（确认修改），同时提交自己

注意：当follower接收到客户端的写请求时，会将该请求转发到leader服务器

### 崩溃回复

当leader崩溃后，zookeeper集群会选择所有服务器中拥有最大事务id（zxid）的服务器作为新的leader，这样可以确保所有已经提交的数据能够同步到集群中所有的服务器中。



## java API的使用

### 原生API

首先添加 zookeeper 的相关依赖：

```xml
<dependency>
    <groupId>org.apache.zookeeper</groupId>
    <artifactId>zookeeper</artifactId>
    <version>3.4.13</version>
    <type>pom</type>
</dependency>
```

首先需要定义连接zookeeper集群中所有服务器的ip地址以及对应的端口号，zookeeper服务器默认语客户端连接的端口号为2181，这里定义了三台服务器，如 `CONNECTSTRING`。

`CountDownLatch`是一个计数器，当watcher监听到连接状态变化为已连接之后会执行减一操作，这时`main()`方法便可以继续执行，主要原因是防止zookeeper在连接的过程中，`main()`方法继续执行导致执行结果与预期不一致的问题。

```java
public class Demo{
    private final static String CONNECTSTRING="192.168.149.128:2181,192.168.149.130:2181,192.168.149.129:2181";
	private static CountDownLatch countDownLatch = new CountDownLatch(1);
	private static ZooKeeper zooKeeper;

	public static void main(String[] args) throws Exception{
    	zooKeeper = new ZooKeeper(CONNECTSTRING, 5000, new MyWatcher());
        countDownLatch.await();

        String result = zooKeeper.create("/clq","123".getBytes(), ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT);

        zooKeeper.setData("/clq","234".getBytes(),-1);

        zooKeeper.delete("/clq",-1);

        zooKeeper.getChildren(path,true);
	}
    
    class MyWatcher implements Watcher{

        @Override
        public void process(WatchedEvent watchedEvent) {
            if (watchedEvent.getState() == Event.KeeperState.SyncConnected) {
                countDownLatch.countDown();
                System.out.println(watchedEvent.getState() + "-->" + watchedEvent.getType());
            }
            if (watchedEvent.getType() == Event.EventType.NodeDataChanged) {
                ......
            } else if (watchedEvent.getType() == Event.EventType.NodeChildrenChanged) {
                ......
            } else if (watchedEvent.getType() == Event.EventType.NodeCreated) {
                ......
            } else if (watchedEvent.getType() == Event.EventType.NodeDeleted) {
                ......
            }
        }
    }
}
```

### zkClient

需要添加依赖：

```xml
<dependency>
    <groupId>com.101tec</groupId>
    <artifactId>zkclient</artifactId>
    <version>0.10</version>
</dependency>
```

首先同样需要定义所有zookeeper服务器的IP地址和连接端口，如 `CONNECTSTRING`。

与原生API不同的是，zkClient在创建节点的时候，能够实现迭代创建节点，不需要创建父节点之后，在创建子节点这样繁琐的过程，删除节点的时候同样也支持迭代删除节点。

注册watcher的过程也与原生API不同，调用 `zkClient.subscribeDataChanges()`方法，并且传入 `IZkDataListener`实例，并且实现其相应的处理方法，同时实现了持续监听，不需要每次响应之后重新注册监听。

```java
public class ZkClientApiOperatorDemo {

    private final static String CONNECTSTRING="192.168.149.128:2181,192.168.149.130:2181,192.168.149.129:2181";

    private static ZkClient getInstance(){
        return new ZkClient(CONNECTSTRING,4000);
    }

    public static void main(String[] args) throws Exception{
        ZkClient zkClient = ZkClientApiOperatorDemo.getInstance();
        zkClient.createPersistent("/clq/clq1/clq1-1/clq1-1-1",true);

        List<String> list = zkClient.getChildren("/clq");
        System.out.println(list);

        zkClient.subscribeDataChanges("/clq", new IZkDataListener() {
            @Override
            public void handleDataChange(String s, Object o) throws Exception {
                System.out.println("节点名称："+s+"-> 节点的值："+o);
            }

            @Override
            public void handleDataDeleted(String s) throws Exception {

            }
        });
        zkClient.writeData("/clq","clq");
        TimeUnit.SECONDS.sleep(2);
        System.out.println("success");


    }
}
```

### curator

需要添加依赖：

```xml
<dependency>
    <groupId>org.apache.curator</groupId>
    <artifactId>curator-framework</artifactId>
    <version>2.11.0</version>
</dependency>
<dependency>
    <groupId>org.apache.curator</groupId>
    <artifactId>curator-recipes</artifactId>
    <version>2.11.0</version>
</dependency>
```

本身是Netfix公司开源的zookeeper客户端；curator提供了各种应用场景的封装

curator-framework：提供了fluent风格的api

curator-recipes：提供了实现封装

首先同样需要定义所有zookeeper服务器的IP地址和连接端口，如 `CONNECTSTRING`。

创建zookeeper连接有两种方式，一种是传统的创建方式，另一种是fluent风格的创建方式。

```java
public class CuratorOperatorDemo {

    private final static String CONNECTSTRING="192.168.149.128:2181,192.168.149.130:2181,192.168.149.129:2181";

    public static CuratorFramework getInstance(){
        //传统创建方式
        CuratorFramework curatorFramework = CuratorFrameworkFactory
                .newClient(CONNECTSTRING,5000,5000
                        ,new ExponentialBackoffRetry(1000,3));
        curatorFramework.start();
        //fluent风格的创建方式
        CuratorFramework curatorFramework1 = CuratorFrameworkFactory
                .newClient(CONNECTSTRING,5000,5000
                        ,new ExponentialBackoffRetry(1000,3));
        curatorFramework1.start();
        return curatorFramework1;
    }

    public static void main(String[] args) {
        CuratorFramework curatorFramework = getInstance();
        System.out.println("conncet success ......");
        try {
            //创建节点，支持迭代创建，fluent风格
            curatorFramework.create().creatingParentContainersIfNeeded()
              .withMode(CreateMode.PERSISTENT).forPath("/curator/curator1/curator1-1","123".getBytes());
            //删除节点
         	curatorFramework.delete().deletingChildrenIfNeeded()
             .forPath("/clq");
            //stat会保存操作后的节点信息
	     	Stat stat = new Stat();
            //获取节点的信息，信息会保存到stat中
            byte[] bytes = curatorFramework.getData().storingStatIn(stat)
                .forPath("/clq");
            System.out.println(new String(bytes)+ "-> stat"+stat);
            //设置节点数据
            Stat stat1 = curatorFramework.setData()
                .forPath("/clq","123".getBytes());
            System.out.println(stat1);
            //创建节点cache，使用该cache为节点注册监听
            NodeCache cache = new NodeCache(curatorFramework,"/curator",false);
        	//使该cache立刻加载该节点的数据
            cache.start(true);
			//为节点注册监听，curator已经在内部实现了持续监听，这里使用的lambda表达式，可以根			//据不同的需要进行响应的实现
        	cache.getListenable().addListener(()-> System.out.println("节点数据发生变化，变化后的结果："
                + new String(cache.getCurrentData().getData())));
            //这里可以定义连续操作，最后返回的结构保存到Collection中，可以遍历该Collection来				//获取所有的执行结果
            Collection<CuratorTransactionResult> resultCollection = curatorFramework.inTransaction().create()
                    .forPath("/ccc","3456".getBytes())
                  .and().setData().
                forPath("/ccc","111".getBytes()).and().commit();

            for(CuratorTransactionResult result : resultCollection){
                System.out.println(result.getForPath()+"->"+result.getType());
            }
        } catch (Exception e) {
            e.printStackTrace();
        }

    }
}
```

##  











