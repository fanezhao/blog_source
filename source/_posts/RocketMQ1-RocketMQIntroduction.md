---
title: RocketMQ1-RocketMQ入门
date: 2020-05-26 20:30:00
tags:
  - 消息队列
---

## 什么是消息队列？

简单来说，消息队列就是基础数据结构中“先进先出”的一种数据结构。现在互联网“微服务”架构的兴起，原有大型集中式的IT服务因为各种弊端，通常被拆分成多个细粒度的多个“微服务”，这些微服务可以在一个局域网内，也可能跨机房部署。一方面对服务之间松耦合的要求越来越高，另一方面，服务之间的联系却越来越紧密，对通信的质量也越来越高。分布式消息队列可以提供应用解耦、流量削峰、消息分发等功能，已经成为大型互联网服务架构里标配的中间件。

## RocketMQ是什么？

{% qnimg mq.png %}

上图是一个典型的消息中间件收发消息的模型，RocketMQ也是这样的设计，简单说来，RocketMQ具有以下特点：

- 是一个队列模型的消息中间件，具有高性能、高可靠、高实时、分布式特点。

- `Producer`、`Consumer`、队列都可以分布式。
- Producer向一些队列轮流发送消息，队列集合称为`Topic`，`Consumer`如果做广播消费，则一个`Consumer`实例消费这个Topic对应的所有队列，如果做集群消费，则多个`Consumer`实例平均消费这个topic对应的队列集合。
- 能够保证严格的消息顺序。
- 提供丰富的消息拉取模式。
- 高效的订阅者水平扩展能力。
- 实时的消息订阅机制。
- 亿级消息堆积能力。
- 较少的依赖。

### RocketMQ实现架构

{% qnimg rmq-basic.png %}

Rocket的架构由四个部分组成：`NameServer`，`Broker`，`Producer`和`Consumer`。它们中的每一个都可以水平扩展，而没有单个故障点。

- **NameServer**：`NameServer`提供轻量级的服务发现和路由。每个`NameServer`记录完整的路由信息，提供相应的读写服务，并支持快速的存储扩展。

- **Broker**：`Broker`通过提供轻量级的`TOPIC`和`QUEUE`机制来提供消息存储。它们支持`Push`和`Pull`模型，包含容错机制（2个副本或3个副本），并提供强大的峰值填充功能和按原始时间顺序累积数千亿条消息的能力。此外，`Broker`提供灾难恢复，丰富的指标统计信息和警报机制，而这是传统消息传递系统所没有的。
- **Producer**：`Producer`支持分布式部署。分布式`Producer`通过多种负载平衡模式将消息发送到`Broker`集群。发送过程支持快速失败并且延迟低。
- **Consumer**：`Consumer`也支持`Push`和`Pull`模型中和分布式部署。它还支持集群使用和消息广播。它提供了实时消息订阅机制，可以满足大多数消费者的需求。

了解了上面四种角色之后，再介绍一下`Topic`和`Message Queue`。一般来说，每个业务都有不同类型的消息要投递，就可以把这些不同类型的消息以不同的`Topic`做区分，所以发送和接收消息之前，先创建`Topic`。

有了`Topic`之后，还要解决性能问题。如果一个`Topic`要发送和接收的数据量非常大，需要能支持增加并行处理的机器来提高处理速度，这时候一个`Topic`可以根据需要设置一个或多个`Message Queue`，`Message Queue`类型分区或`Partition`。`Topic`有了多个`Message Queue`之后，消息就可以并行的向各个`Message Queue`发送，消费者也可以并行地从多个`Message Queue`读取消息并消费。

### RocketMQ 物理部署结构

{% qnimg rocketmq物理部署.png %}

如上图所示， RocketMQ的部署结构有以下特点：

- `NameServer`是一个几乎无状态节点，可集群部署，节点之间无任何信息同步。

- `Broker`部署相对复杂，`Broker`分为Master与Slave，一个Master可以对应多个Slave，但是一个Slave只能对应一个Master，Master与Slave的对应关系通过指定相同的BrokerName，不同的BrokerId来定义，BrokerId为0表示Master，非0表示Slave。**Master也可以部署多个。每个Broker与Name Server集群中的所有节点建立长连接，定时注册Topic信息到所有Name Server**。
- `Producer`与`NameServer`集群中的其中一个节点（随机选择）建立长连接，定期从`NameServer`取`Topic`路由信息，并向提供`Topic`服务的Master建立长连接，且定时向Master发送心跳。`Producer`完全无状态，可集群部署。
- `Consumer`与`NameServer`集群中的其中一个节点（随机选择）建立长连接，定期从`NameServer`取`Topic`路由信息，并向提供`Topic`服务的Master、Slave建立长连接，且定时向Master、Slave发送心跳。`Consumer`既可以从Master订阅消息，也可以从Slave订阅消息，订阅规则由Broker配置决定。

### 如何存储队列位置信息

实际运行中的系统，难免会遇到重新消费某条消息、跳过一段时间内的消息等情况。这些异常情况的处理，都和`Offset`有关。RocketMQ中，一种类型的消息会放到一个`Topic`，为了能够并行，一般一个`Topic`会有多个`Message Queue`（也可以设置成一个），`Offset`是指某个`Topic`下的一条消息在某个`Message Queue`里的位置，通过`Offset`可以定位到这条消息，或者指示`Consumer`从这条消息开始向后继续处理。

## NameServer的功能

对于一个消息队列集群来说，系统由很多台机器组成，每个机器的角色、IP地址都不相同，而且这些信息是变动的。这种情况下，如果一个新的`Producer`或`Consumer`加入，怎么配置连接信息呢？`NameServer`的存在主要是解决这类问题的。

`NameServer`是整个消息队列中的状态服务器，集群的各个组件通过它来了解全局信息。同时各个角色的机器都要定期向`NameServer`上报自己状态，超时不上报的话，`NameServer`就认为这个机器不可用了，其它组件会将这个机器从可用列表移除。

`NameServer`可以部署多个，相互之间独立，其它角色同时向多个`NameServer`机器上报状态信息，从而达到热备份的目的。`NameServer`本身是无状态的，也就是说`NameServer`中的`Broder`、`Topic`等状态信息不会持久存储，都是由各个角色定时上报并存储到内存中（`NameServer`支持配置参数的持久化，不过一般用不到）。

### 存储结构

- 1、topicQueueTable

  ```java
  private final HashMap<String/* topic */, List<QueueData>> topicQueueTable;
  ```

  这个结构的`Key`是`Topic`的名称，它存储了所有`Topic`的属性信息。Value是一个QueueData队列，队列的长度等于这个`Topic`数据存储的Master Broker的个数，QueueData里存储着Broker的名称、读写queue的数量、同步标识等。

- 2、brokerAddrTable

  ```java
  private final HashMap<String/* brokerName */, BrokerData> brokerAddrTable;
  ```

  这个是以BrokerName为索引，相同名称的Broker可能存在多台机器，一个Master和Slave。这个结构存储着一个BrokerName对应的属性信息，包括所属的Cluster名称，一个Master Broker和多个Slave Broker的地址信息。

- 3、clusterAddrTable

  ```java
  private final HashMap<String/* clusterName */, Set<String/* brokerName */>> clusterAddrTable;
  ```

  clusterAddrTable 存储的集群中的Cluster的信息，结果很简单，就是一个Cluster对应一个由BrokerName的组合。

- 4、brokerLiveTable

  ```java
  private final HashMap<String/* brokerAddr */, BrokerLiveInfo> brokerLiveTable;
  ```

  这个结构和brokerAddrTable有关系，但是内容完全不同，它的Key是BrokerAddr。也就是对应着一台机器，BrokerAddrTable中的Key是BrokerName，多个机器的BrokerName可以相同。

  brokerLiveTable存储的内容是这台`Broker`机器的实时状态，包括上次更新状态的时间戳，`NameServer`会定期检查这个时间戳，超时（2分钟）没有更新就认为这个`Broker`无效，将其从Broker列表中删除。

- 5、filterServerTable

  ```java
  private final HashMap<String/* brokerAddr */, List<String>/* Filter Server */> filterServerTable;
  ```

  Filter Server是过滤服务器，是RocketMQ的一种服务端过滤方式，一个Broker可以有一个或多个Filter Server。这个结构的Key是`Broker`的地址，Value是和这个`Broker`关联的多个Filter Server的地址。

### 状态维护

RocketMQ架构中其它角色会向`NameServer`上报状态，`NameServer`的主要逻辑在`DefaultRequestProcessor`类中，根据上报消息的请求码，更新存储的对应信息。

此外，连接断开的事件也会触发状态更新，具体逻辑在`org.apache.rocketmq.namesrv.routeinfo.BrokerHousekeepingService`中，代码清单如下：

```java
    @Override
    public void onChannelClose(String remoteAddr, Channel channel) {
        this.namesrvController.getRouteInfoManager().onChannelDestroy(remoteAddr, channel);
    }

    @Override
    public void onChannelException(String remoteAddr, Channel channel) {
        this.namesrvController.getRouteInfoManager().onChannelDestroy(remoteAddr, channel);
    }

    @Override
    public void onChannelIdle(String remoteAddr, Channel channel) {
        this.namesrvController.getRouteInfoManager().onChannelDestroy(remoteAddr, channel);
    }
```

当`NameServer`和`Broker`的长连接断掉之后，`onChannelDestroy`函数会被调用，把这个Broker的信息清理出去。

`NameServer`还有定时检查时间时间戳的逻辑，`Broker`向`NameServer`发送的心跳会更新时间戳，当`NameServer`检查时间戳长时间没更新后，便会触发清理逻辑。

```java
this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {
    @Override
    public void run() {
        NamesrvController.this.routeInfoManager.scanNotActiveBroker();
    }
}, 5, 10, TimeUnit.SECONDS);
```

每10s检查一次，时间戳超时2mins则认为`Broker`已失效。

### 为何不用ZooKeeper?

ZK是Apache的一个开源软件，为分布式应用程序提供协调服务。那么为什么RocketMQ要自己造轮子呢？

因为ZK的功能很强大，包括自动Master选举等，RocketMQ的架构设计决定了它不需要进行Master选举，用不到这些复杂的功能，只需要一个轻量级的元数据服务器就足够了。

而且中间件对稳定性要求很高，RocketMQ的NameServer只有很少的代码，容易维护，所以不需要再依赖另一个中间件，从而减少维护成本。

## 通信协议

RocketMQ自己定义了一个通信协议，使得模块间传输的二进制消息和有意义的内容之间互相转换。

{% qnimg rmq通信协议.jpg %}

- 第一部份是4个字节整数，值等于第二、三、四部分长度的总和。
- 第二部份是4个字节整数，值等于第三部分长度的总和。
- 第三部分是通过Json序列化的数据。
- 第四部份是通过应用自定义二进制序列化的数据。

## 对事务的支持

RocketMQ的事务消息，是指发送消息事件和其它事件需要同时成功或同时失败。比如银行转账，A银行的某账户需要转1W到B银行的某账户。A银行发送“B银行账户增加1W”这个消息，要和“从A银行账户扣除1W”这个操作同时成功或同时失败。

RocketMQ采用两阶段提交的方式实现事务消息。`TransactionMQProducer`处理上面情况的流程是，先发一个“准备从B银行账户增加1W”的消息，发送成功之后做从A银行扣除1W的操作，根据操作结果是否成功，确定之前的“准备从B银行账户增加1W”的消息是做`commit`还是`rollback`。具体流程如下：

- 1、发送方向RocketMQ发送“待确认”消息。

- 2、RocketMQ将收到的“待确认”消息持久化成功之后，向发送方回复消息已经发送成功，此时第一阶段消息发送完成。
- 3、发送方开始执行本地逻辑事件。
- 4、发送方根据本地逻辑事件执行结果向RocketMQ发送二次确认（`commit`或是`rollback`）消息，RocketMQ收到`commit`状态则将第一阶段消息标记为可投递，订阅方将能收到该消息；收到`rollback`状态则删除第一阶段的消息，则订阅方不会收到该消息。
- 5、如果出现异常情况，步骤4提交的二次确认最终未到达RocketMQ，服务器在固定时间段后将对“待确认”消息发起回查请求。
- 6、发送方收到消息回查请求后（如果发送第一阶段消息的`Producer`不能工作，回查请求被发送到和`Producer`在同一个Group里的其他`Producer`），通过检查对应消息的本地事件执行结果返回`commit`或`rollback`状态。
- 7、RocketMQ收到回查请求后，按照步骤4的逻辑处理。

## 参考链接

[RocketMQ官网](http://rocketmq.apache.org/docs/rmq-arc/)

[十分钟入门RocketMQ](http://jm.taobao.org/2017/01/12/rocketmq-quick-start-in-10-minutes/)

