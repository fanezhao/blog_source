---
title: RocketMQ3-RocketMQ是如何保证不丢失消息的
date: 2020-05-27 13:30:00
tags:
  - 消息队列
---

> 来聊聊如何保证 RocketMQ 不丢失消息的。

## 消息的发送流程

一条消息从生产到被消费，将会经历三个阶段：

{% qnimg rmq消息三阶段.jpg %}

- 生产阶段：Producer 新建消息，然后通过网络将消息投递给 MQ Broker。
- 存储阶段：消息将会存储在 Broker 端磁盘中。
- 消息阶段： Consumer 将会从 Broker 拉取消息。

以上任一阶段都可能会丢失消息，我们只要找到这三个阶段丢失消息原因，采用合理的办法避免丢失，就可以彻底解决消息丢失的问题。

## 一、生产阶段

生产者（Producer） 通过网络发送消息给 Broker，当 Broker 收到之后，将会返回确认响应信息给 Producer。所以生产者只要接收到返回的确认响应，就代表消息在生产阶段未丢失。

RocketMQ 发送消息示例代码如下：

```java
DefaultMQProducer mqProducer=new DefaultMQProducer("test");
// 设置 nameSpace 地址
mqProducer.setNamesrvAddr("namesrvAddr");
mqProducer.start();
Message msg = new Message("test_topic" /* Topic */,
        "Hello World".getBytes(RemotingHelper.DEFAULT_CHARSET) /* Message body */
);
// 发送消息到一个Broker
try {
    SendResult sendResult = mqProducer.send(msg);
} catch (RemotingException e) {
    e.printStackTrace();
} catch (MQBrokerException e) {
    e.printStackTrace();
} catch (InterruptedException e) {
    e.printStackTrace();
}
```

`send`方法是一个同步操作，只要这个方法不抛出任何异常，就代表消息已经**发送成功**。

消息发送成功仅代表消息已经到了 Broker 端，Broker 在不同配置下，可能会返回不同响应状态:

- SendStatus.SEND_OK
- SendStatus.FLUSH_DISK_TIMEOUT
- SendStatus.FLUSH_SLAVE_TIMEOUT
- SendStatus.SLAVE_NOT_AVAILABLE

引用官方状态说明：

{% qnimg rmq消息发送结果官方解释.jpeg %}

另外 RocketMQ 还提供异步的发送的方式，适合于链路耗时较长，对响应时间较为敏感的业务场景。

```java
DefaultMQProducer mqProducer = new DefaultMQProducer("test");
// 设置 nameSpace 地址
mqProducer.setNamesrvAddr("127.0.0.1:9876");
mqProducer.setRetryTimesWhenSendFailed(5);
mqProducer.start();
Message msg = new Message("test_topic" /* Topic */,
        "Hello World".getBytes(RemotingHelper.DEFAULT_CHARSET) /* Message body */
);

try {
    // 异步发送消息到，主线程不会被阻塞，立刻会返回
    mqProducer.send(msg, new SendCallback() {
        @Override
        public void onSuccess(SendResult sendResult) {
            // 消息发送成功，
        }

        @Override
        public void onException(Throwable e) {
            // 消息发送失败，可以持久化这条数据，后续进行补偿处理
        }
    });
} catch (RemotingException e) {
    e.printStackTrace();
} catch (InterruptedException e) {
    e.printStackTrace();
}
```

异步发送消息一定要**注意重写**回调方法，在回调方法中检查发送结果。

不管是同步还是异步的方式，都会碰到网络问题导致发送失败的情况。针对这种情况，我们可以设置合理的重试次数，当出现网络问题，可以自动重试。设置方式如下：

```java
// 同步发送消息重试次数，默认为 2
mqProducer.setRetryTimesWhenSendFailed(3);
// 异步发送消息重试次数，默认为 2
mqProducer.setRetryTimesWhenSendAsyncFailed(3);
```

## 插播知识点

在看存储阶段之前，我们先看两个RocketMQ的概念。

### 同步刷盘和异步刷盘

RocketMQ的消息是存储到磁盘上的，这样既能保证断电后恢复，又可以让存储的消息量超出内存的限制。RocketMQ为提高性能，会尽可能保证磁盘的顺序写。消息在通过Producer写入RocketMQ的时候，有两种写磁盘的方式。

- 异步刷盘方式：在返回写成功状态时，消息可能只是被写入了内存的PAGECHCHE，写操作的返回快，吞吐量大；当内存里的消息累积到一定程序的时候，统一触发写磁盘动作，快速写入。
- 同步刷盘方式：在返回写成功状态时，消息已经被写入磁盘。具体流程是，消息写入内存的PAGECHAHE后，立即通知刷盘线程刷盘，然后等待线程刷盘完成，完成之后唤醒等待的线程，返回消息写成功的状态。

同步刷盘还是异步刷盘，是通过Broker配置文件里的`flushDiskType`参数设置的，同步刷盘设置为`SYNC_FLUSH`，异步刷盘设置为`ASYNC_FLUSH`（默认）。

### 同步复制和异步复制

如果一个Broker组有Master和Slave，消息需要通过从Master复制到Slave上，有同步和异步两种复制方式。

- 异步复制方式是只要Master写成功即可反馈给客户端写成功状态。
- 同步复制方式是等Master和Slave均写成功后才反馈给客户端写成功状态。

这两种复制方式各有优劣：

- 在异步复制方式下，系统拥有较低的延迟和较高的吞吐量，但如果Master出了故障，有些数据因为没有被定进Slave，有可能会丢失。
- 在同步复制方式下，如果Master出了故障，Slave上有全部的备份数据，容易恢复，但是同步复制会增大数据写入延迟、降低系统吞吐量。

同步复制和异步复制也是通过Broker配置文件里的`brokerRole`参数进行设计的，对Master节点来说，同步复制应该设置成`SYNC_MASTER`，异步复制设置为`ASYNC_MASTER`（默认）；对于Slave节点来说，无论同步复制还是异步复制，`brokerRole`参数设置为`SLAVE`即可。

实际应用中要结合业务场景，合理设置刷盘方式和主从复制方式，尤其是`SYNC_FLUSH`方式，由于频繁的触发磁盘写动作，会明显降低性能。通常情况下，应该把Master和Slave配置成`ASYNC_FLUSH`的刷盘方式，主从之间配置成`SYNC_MASTER`的复制方式，这样即使有一台机器出现故障，仍然能保证数据不丢。是个不错的选择。

## 二、Broker存储阶段

默认情况下，消息只要到了 Broker 端，将会优先保存到内存中，然后立刻返回确认响应给生产者。随后 Broker 定期批量的将一组消息从内存异步刷入磁盘。

这种方式减少 I/O 次数，可以取得更好的性能，但是如果发生机器掉电，异常宕机等情况，消息还未及时刷入磁盘，就会出现丢失消息的情况。

若想保证 Broker 端不丢消息，保证消息的可靠性，我们需要将消息保存机制修改为同步刷盘方式，即消息**存储磁盘成功**，才会返回响应。

修改 Broker 端配置如下：

```properties
## 默认情况为 ASYNC_FLUSH 
flushDiskType = SYNC_FLUSH 
```

若 Broker 未在同步刷盘时间内（**默认为 5s**）完成刷盘，将会返回`SendStatus.FLUSH_DISK_TIMEOUT`状态给生产者。

### 集群部署

为了保证可用性，Broker 通常采用一主（**Master**）多从（**Slave**）部署方式。为了保证消息不丢失，消息还需要复制到 Slave 节点。

默认方式下，消息写入**Master**成功，就可以返回确认响应给生产者，接着消息将会异步复制到**Slave**节点。

此时若 master 突然**宕机且不可恢复**，那么还未复制到**Slave**的消息将会丢失。

为了进一步提高消息的可靠性，我们可以采用同步的复制方式，**Master** 节点将会同步等待**Slave**节点复制完成，才会返回确认响应。

Broker master 节点 同步复制配置如下：

```properties
## 默认为 ASYNC_MASTER 
brokerRole=SYNC_MASTER
```

如果 **Slave** 节点未在指定时间内同步返回响应，生产者将会收到`SendStatus.FLUSH_SLAVE_TIMEOUT`返回状态。

结合生产阶段与存储阶段，若需要**严格保证消息不丢失**，broker 需要采用如下配置：

```properties
## master 节点配置
flushDiskType = SYNC_FLUSH
brokerRole=SYNC_MASTER

## slave 节点配置
brokerRole=slave
flushDiskType = SYNC_FLUSH
```

同时这个过程我们还需要生产者配合，判断返回状态是否是`SendStatus.SEND_OK`。若是其他状态，就需要考虑补偿重试。

虽然上述配置提高消息的高可靠性，但是会**降低性能**，生产实践中需要综合选择。

## 三、消费阶段

消费者从Broker拉取消息，然后执行相应的业务逻辑。一旦执行成功，将会返回`ConsumeConcurrentlyStatus.CONSUME_SUCCESS`状态给 Broker。

如果Broker未收到消费确认响应或收到其他状态，消费者下次还会再次拉取到该条消息，进行重试。这样的方式有效避免了消费者消费过程发生异常，或者消息在网络传输中丢失的情况。

消息消费的代码如下：

```java
// 实例化消费者
DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("test_consumer");

// 设置NameServer的地址
consumer.setNamesrvAddr("namesrvAddr");

// 订阅一个或者多个Topic，以及Tag来过滤需要消费的消息
consumer.subscribe("test_topic", "*");
// 注册回调实现类来处理从broker拉取回来的消息
consumer.registerMessageListener(new MessageListenerConcurrently() {
    @Override
    public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> msgs, ConsumeConcurrentlyContext context) {
        // 执行业务逻辑
        // 标记该消息已经被成功消费
        return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
    }
});
// 启动消费者实例
consumer.start();
```

以上消费消息过程的，我们需要**注意返回消息状态**。只有当业务逻辑真正执行成功，我们才能返回
`ConsumeConcurrentlyStatus.CONSUME_SUCCESS`。否则我们需要返回`ConsumeConcurrentlyStatus.RECONSUME_LATER`，稍后再重试。

## 总结

本文通过一条消息生命周期的三个阶段来分析消息可能丢失的原因，并且跟根据原因做出一系列措施，理论上可以有效防止RocketMQ的消息丢失。

## 参考链接

[面试官再问我如何保证 RocketMQ 不丢失消息,这回我笑了！](https://www.toutiao.com/i6807935652416455171/?tt_from=weixin&utm_campaign=client_share&wxshare_count=1&timestamp=1590497337&app=news_article&utm_source=weixin&utm_medium=toutiao_ios&req_id=20200526204856010014047021117326D4&group_id=6807935652416455171)