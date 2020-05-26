---
title: RocketMQ3-RocketMQ是如何保证不丢失消息的
date: 2020-05-26 20:58:51
tags:
  - 消息队列
---

> 来聊聊如何保证 RocketMQ 不丢失消息的。

## 消息的发送流程

一条消息从生产到被消费，将会经历三个阶段：

{% qnimg rmq-dontlostmsg.jpg %}

- 生产阶段，Producer 新建消息，然后通过网络将消息投递给 MQ Broker
- 存储阶段，消息将会存储在 Broker 端磁盘中
- 消息阶段， Consumer 将会从 Broker 拉取消息

以上任一阶段都可能会丢失消息，我们只要找到这三个阶段丢失消息原因，采用合理的办法避免丢失，就可以彻底解决消息丢失的问题。

## 生产阶段

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

send 方法是一个同步操作，只要这个方法不抛出任何异常，就代表消息已经**「发送成功」**。

消息发送成功仅代表消息已经到了 Broker 端，Broker 在不同配置下，可能会返回不同响应状态:

- SendStatus.SEND_OK
- SendStatus.FLUSH_DISK_TIMEOUT
- SendStatus.FLUSH_SLAVE_TIMEOUT
- SendStatus.SLAVE_NOT_AVAILABLE

引用官方状态说明：

{% qnimg rmq-status.jpeg %}

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

异步发送消息一定要**「注意重写」**回调方法，在回调方法中检查发送结果。

不管是同步还是异步的方式，都会碰到网络问题导致发送失败的情况。针对这种情况，我们可以设置合理的重试次数，当出现网络问题，可以自动重试。设置方式如下：

```java
// 同步发送消息重试次数，默认为 2
mqProducer.setRetryTimesWhenSendFailed(3);
// 异步发送消息重试次数，默认为 2
mqProducer.setRetryTimesWhenSendAsyncFailed(3);
```

## Broker 存储阶段

默认情况下，消息只要到了 Broker 端，将会优先保存到内存中，然后立刻返回确认响应给生产者。随后 Broker 定期批量的将一组消息从内存异步刷入磁盘。

这种方式减少 I/O 次数，可以取得更好的性能，但是如果发生机器掉电，异常宕机等情况，消息还未及时刷入磁盘，就会出现丢失消息的情况。

若想保证 Broker 端不丢消息，保证消息的可靠性，我们需要将消息保存机制修改为同步刷盘方式，即消息**「存储磁盘成功」**，才会返回响应。

修改 Broker 端配置如下：

```
## 默认情况为 ASYNC_FLUSH 
flushDiskType = SYNC_FLUSH 
```

若 Broker 未在同步刷盘时间内（**「默认为 5s」**）完成刷盘，将会返回`SendStatus.FLUSH_DISK_TIMEOUT`状态给生产者。

