# RocketMQ

RocketMQ 是一个基于 Java 实现的，分布式队列模型的消息中间件，具有高可用、高可靠、高实时、低延迟的特点。

## 主要角色

### NameServer

NameServer 是基于 Netty 实现的路由管理、服务注册、服务发现的注册中心。与 ZooKeeper 不同的是，NameServer 集群实际上是伪集群，每个结点之间相互独立，彼此之间不会进行信息通信。
NameServer 被设计为无状态的，运行时在内存中维护了 Broker、Producer、Consumer、Topic、Group 等信息，但不会持久化这些数据。

NameServer 的主要功能：

- 为 Broker 提供服务注册发现
- 提供 Topic 路由信息
- 可以作为控制台的操作入口

Broker 在启动时会向 NameServer 注册，保持长连接并定时进行心跳连接，定时同步 Topic 信息到 NameServer。
Producer 在发送消息时从 NameServer 中获取 Topic 的路由信息，然后将消息发送到对应的 Broker。
Consumer 也会定时从 NameServer 获取 Topic 的路由信息。

### Broker

Broker 主要负责消息存储、消息分发和消息投递，其内部维护了一个或多个 Message Queue，用于存储消息的索引，通过 CommitLog 日志文件储存消息。
每个 Broker 会与所有 Nameserver 保持长连接，定时同步 Topic 信息到 NameServer。

- 消息接收：Broker 接收到 Producer 的消息后，通过`SendMessageProcessor`类将消息写入到 CommitLog 日志文件。
- 消息分发：Broker 会通过`ReputMessageService`类启动一个线程，将 CommitLog 日志文件中的消息分到对应的 ConsumeQueue 文件以及 IndexFile 文件中。
- 消息投递：Broker 接收到 Consumer 发起获取消息的请求后，调用`PullMessageProcessor`类将 ConsumeQueue 文件中的消息返回给 Consumer。

### Producer

Producer 会选择一个 NameServer 节点保持长连接（无心跳），定时获取 Topic 路由信息。如果当前 NameServer 不可用时，Producer 会自动尝试连接下一个 NameServer，直到有可用连接为止。
默认情况下，Producer 每隔 30 秒从 NameServer 获取所有 Topic 路由信息，这意味着，如果某个 Broker 如果崩溃故障，Producer 最多要 30 秒才能感知。
在此期间，发往该 Broker 的消息发送失败。该时间由`DefaultMQProducer`的`pollNameServerInterval`参数决定，可通过配置修改。

Producer 会通过 Topic 路由信息，与所有关联的 Broker 建立长连接（由于 RocketMQ 设计只有 master 节点才可以发送消息，所以这里只会与 master 节点建立长连接）。
默认情况下，Producer 每隔 30 秒向已连接的 Broker 发送心跳，该时间由`DefaultMQProducer`的`heartbeatBrokerInterval`参数决定，可通过配置修改。
Broker 每隔 10 秒钟（此时间无法更改）扫描已连接的 Producer，若某个连接超过 2 分钟（此时间无法更改）没有发送心跳数据，则主动关闭连接。

#### 负载均衡

Producer 发送消息时，默认会轮询目标 Topic 下的所有 Message Queue，并采用递增取模的方式往不同的 Message Queue 上发送消息，以达到让消息平均落在不同的队列的目的。
由于 Message Queue 是分布在不同的 Broker 上的，所以消息也会发送到不同的 Broker 上。

#### 发送策略

Producer 发送消息有三种策略：

- 同步：发出消息后，必须等待接收方发回响应，才能执行下一个发送消息操作。一般用于重要的消息通知，如重要的通知邮件或者营销短信等。
- 异步：发出消息后，通过异步回调接收方发回响应，不阻塞下一个发送消息操作。一般用于可能链路耗时较长而对响应时间比较敏感的场景，如视频上传后通知启动转码服务。
- 单向：只负责发送消息，而不等待接收方发送响应，且没有异步回调。适合那些耗时比较短且对可靠性要求不高的场景，例如日志收集。

### Consumer

Consumer 会选择一个 NameServer 节点保持长连接（无心跳），定时获取 Topic 路由信息。如果当前 NameServer 不可用时，Consumer 会自动尝试连接下一个 NameServer，直到有可用连接为止。
默认情况下，Consumer 每隔 30 秒从 NameServer 获取所有 Topic 路由信息，这意味着，如果某个 Broker 如果崩溃故障，Consumer 最多要 30 秒才能感知。
在此期间，发往该 Broker 的消息发送失败。该时间由`DefaultMQPushConsumer`的`pollNameServerInterval`参数决定，可通过配置修改。

Consumer 会通过 Topic 路由信息，与所有关联的 Broker 建立长连接（这里会与 master 和 slave 都建立连接）。
默认情况下，Consumer 每隔 30 秒向已连接的 Broker 发送心跳，该时间由`DefaultMQPushConsumer`的`heartbeatBrokerInterval`参数决定，可通过配置修改。
Broker 每隔 10 秒钟（此时间无法更改）扫描已连接的 Consumer，若某个连接超过 2 分钟（此时间无法更改）没有发送心跳数据，则主动关闭连接。
并向该 Consumer Group 的所有 Consumer 发出通知，重新分配队列继续消费。

#### 消费获取方式

- PULL：主动从 Broker 中拉取消息进行消费，该方式需要用户自己实现，通过 Topic 获取 Message Queue 集合，然后遍历集合批量获取消息进行消费，最后记录该队列的 offset。
- PUSH：实现 MessageListener 监听器，并将监听器注册到 Broker，当消息达到 Broker 时，会触发监听器，消费者再去拉取消息进行消费。

#### 消费模式

- 集群消费：一条消息会发送给订阅某个 Topic 的每个 Consumer Group 中的一个消费者实例进行消费。如果某消费者实例不可用时，分组中的其他消费者会接替它进行消费。
- 广播消费：一条消息会发送给订阅某个 Topic 的每个 Consumer Group 中的所有消费者实例进行消费。

默认是集群消费模式。

#### 顺序消费

普通顺序消费模式下，消费者通过同一个消息队列（即一个 Topic 分区）收到的消息是有顺序的，不同消息队列收到的消息则可能是无顺序的。
所以只要生产者将消息确保投递到同一个消息队列中即可。假设，存在一个场景需要保证订单消息能被顺序消费，示例如下：

首先，实现`MessageQueueSelector`接口的`select`方法，通过传入订单`id`通过取模，返回对应的消息队列。

```java
class OrderedMessageQueueSelector implements MessageQueueSelector {
    @Override
    public MessageQueue select(List<MessageQueue> list, Message message, Object o) {
        long id = (long) o;
        return list.get((int) (id % list.size()));
    }
}
```

然后，在发送消息时候，指定该选择器，并传入订单`id`值。

```java
producer.send(msg, selector, id);
```

如此一来，同一个订单`id`，就会被放到同一个消息队列中，然后消费者通过实现`MessageListenerOrderly`接口进行消费即可。

```java
consumer.registerMessageListener(new MessageListenerOrderly() {
    @Override
    public ConsumeOrderlyStatus consumeMessage(List<MessageExt> list, ConsumeOrderlyContext context) {
       context.setAutoCommit(true);
       for (MessageExt msg : list) {
            // 通过日志可以发现每个消息队列都有唯一一个消费者线程
            log.debug("Thread: {}, Queue: {}, Message: {}", Thread.currentThread().getName(), msg.getQueueId(), msg.getBody());
       }
       return ConsumeOrderlyStatus.SUCCESS;
    }
});
```

#### 负载均衡

集群消费模式下，消费者集群多个实例共同消费一个 Topic 的多个队列，一个队列只会被分配给一个消费者实例进行消费。如果某个消费者实例不可用时，分组中的其他消费者会接替它进行消费。
每当有新的消费者加入到分组中时，会重新分配消息队列给各个消费者。

## 核心组件

### Message

Message 是消息载体，发送或者消费消息的时候必须指定 Topic，也可以指定一个可选的 Tag 用于过滤消息，还可以添加额外的键值对。

### Topic

Topic 是消息的主题，表示一类消息的集合，是 RocketMQ 进行消息订阅的基本单位。
例如，订单消息、交易消息、物流消息等。

### Tag

Topic 是消息的标签，用于同一主题下区分不同类型的消息。来自同一业务单元的消息，可以根据不同业务目的在同一主题下设置不同标签。
标签能够有效地保持代码的清晰度和连贯性，并优化 RocketMQ 提供的查询系统。消费者可以根据Tag实现对不同子主题的不同消费逻辑，实现更好的扩展性。
例如，订单创建消息、订单取消消息、交易成功消息、交易失败消息、物理已发货消息等、物流已送达消息。

### Producer Group

表示同一类 Producer 的集合，这类 Producer 发送同一类消息且发送逻辑一致。如果发送的是事务消息且原始生产者在发送之后崩溃，则 Broker 服务器会联系同一生产者组的其他生产者实例以提交或回溯消费。

### Consumer Group

表示同一类 Consumer 的集合，这类 Consumer 通常消费同一类消息且消费逻辑一致。消费者组使得在消息消费方面，实现负载均衡和容错的目标变得非常容易。
要注意的是，消费者组的消费者实例必须订阅完全相同的 Topic。

### Message Queue

一个 Topic 可以划分成多个消息队列。Topic 只是个逻辑上的概念，消息队列是消息的物理管理单位，当发送消息的时候，Broker 会轮询包含该 Topic 的所有消息队列，然后将消息发出去。
通过消息队列，可以使得消息的存储可以分布式集群化，具有了水平的扩展能力。
