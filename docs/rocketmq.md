# RocketMQ

RocketMQ 是一个基于 Java 开发的，分布式队列模型的，开源的消息中间件，具有高可用、高可靠、高实时、低延迟的特点。

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

Producer 是消息生产者，启动服务时，会选择一个 NameServer 节点保持长连接（无心跳），定时获取 Topic 路由信息。如果当前 NameServer 不可用时，Producer 会自动尝试连接下一个 NameServer，直到有可用连接为止。
默认情况下，Producer 每隔 30 秒从 NameServer 获取所有 Topic 路由信息，这意味着，如果某个 Broker 如果崩溃故障，Producer 最多要 30 秒才能感知。
在此期间，发往该 Broker 的消息发送失败。该时间由`DefaultMQProducer`的`pollNameServerInterval`参数决定，可通过配置修改。

Producer 会通过 Topic 路由信息，与所有关联的 Broker 建立长连接（由于 RocketMQ 设计只有主节点才可以接收消息，所以只会与主节点建立长连接）。
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

Consumer 是消息消费者，启动服务时，会选择一个 NameServer 节点保持长连接（无心跳），定时获取 Topic 路由信息。如果当前 NameServer 不可用时，Consumer 会自动尝试连接下一个 NameServer，直到有可用连接为止。
默认情况下，Consumer 每隔 30 秒从 NameServer 获取所有 Topic 路由信息，这意味着，如果某个 Broker 如果崩溃故障，Consumer 最多要 30 秒才能感知。
在此期间，发往该 Broker 的消息发送失败。该时间由`DefaultMQPushConsumer`的`pollNameServerInterval`参数决定，可通过配置修改。

Consumer 会通过 Topic 路由信息，与所有关联的 Broker 建立长连接（与主节点和从节点都会建立连接）。
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

## 高可用

### 多主模式

由多个主节点组成集群，单个主节点宕机或者重启对应用没有影响。

优点：所有模式中性能最高。在多主的架构体系下，一个 Topic 的分布在不同的主节点的，方便进行横向拓展。

缺点：单个主节点宕机期间，未被消费的消息在节点恢复之前不可用，消息的实时性就受到影响。

### 多主多从同步复制模式

由多个主节点组成集群，且每个主节点都有至少一个从节点。消费者可以通过从节点消费消息，但是不能接收生产者发送的消息。此外，还可以用于数据备份。

对数据要求较高的场景，主从同步复制方式，保存数据热备份，通过异步刷盘方式，保证 RocketMQ 高吞吐量。

优点：主从同步复制模式能保证数据不丢失。

缺点：发送单个消息响应时间会略长，性能相比异步复制低 10% 左右。

### 多主多从异步复制模式

类似于多主多从同步复制模式，性能略有提升，但是可能会发生消息丢失。

优点：主从异步复制模式性能和多主模式接近。

缺点：使用异步复制的同步方式有可能会有消息丢失的问题。

### DLedger 模式

RocketMQ 4.5 版本引入 DLedger，增加了一种全新的复制方式。在写入消息的时候，要求至少消息复制到半数以上的节点之后，才给客户端返回写入成功，并且支持通过选举来动态切换主节点的。

假如存在 1 个主节点和 2 个从节点，共 3 个节点，当主节点不可用时，2 个从节点会通过投票选出一个新的主节点来继续提供服务，相比主从的复制模式，解决了可用性的问题。
由于消息要至少复制到 2 个节点上才会返回写入成功，所以即使主节点宕机了，也至少有一个节点上的消息是和主节点一样的。
在选举时，总会把数据和主节点一样的从节点选为新的主节点，这样就保证了数据的一致性，既不会丢消息，还可以保证严格顺序。

优点：支持动态切换主节点，保证消息不会丢失。

缺点：最少需要 3 个节点才能保证数据一致性，由于至少要复制到半数以上的节点才返回写入成功，所以性能上不如多主多从复制模式。如果主节点崩溃故障，选举过程中不能提供服务。

## 事务消息

RocketMQ 4.3 版支持事务消息（Transactional Message），采用了两阶段提交的思想来实现了提交事务消息，同时增加一个补偿机制来处理二阶段超时或者失败的消息，如下图所示：

![rocketmq_transactionalmessage](images/rocketmq_transactionalmessage.png)

事务消息的发送及提交流程如下：

1. Producer 向 Broker 发送半消息。
2. Broker 将消息持久化之后，返回响应结果给 Producer。
3. 如果半消息发送成功，Producer 执行本地事务逻辑。
4. Producer 根据本地事务执行结果，向 Broker 提交二次确认。
    - 如果执行成功，发送 Commit 命令，Broker 将半事务消息标记为可投递，订阅方最终将收到该消息。
    - 如果执行失败，发送 Rollback 命令，Broker 删除半事务消息，订阅方将不会接收该消息。

在网络不稳定或者是应用重启的情况下，可能导致 Producer 发送的二次确认消息未能到达 Broker，此时会通过事务消息的补偿机制处理这种情况，流程如下：

5. 如果 Broker 长时间没有收到 Commit 或 Rollback 命令，会发送一条“回查”消息给 Producer。
6. Producer 收到“回查”消息，检查本地事务执行的最终结果。
7. Producer 再次向 Broker 提交二次确认给 Broker。

> 半消息是一种特殊的消息类型，该状态的消息暂时不能被 Consumer 消费。
> 当一条事务消息被成功投递到 Broker 上，且没有接收到 Producer 发出的二次确认时，该事务消息就处于"暂时不可被消费"状态，该状态的事务消息被称为半消息。

## 消息重复消费

消息队列无法保证消息不被重复消费，需要通过业务代码实现幂等。

## 消息可靠投递

### 从 Broker 端考虑

1. 消息只要持久化到 CommitLog 日志文件中，即使 Broker 宕机，未消费的消息也能重新恢复再消费。
2. 采用同步刷盘策略，可以保证消息持久化到磁盘之后，再返回响应给生产者。
3. 采用多主多从同步复制或 DLedger 部署方案。

### 从 Producer 端考虑

1. 采用同步发送方式，发送一条消息并等待返回响应结果，如果返回结果为失败，则可以重试或抛出业务异常。
    - 发送成功，说明消息已经投递
    - 发送失败，可以重试或抛出异常
    - 发送超时，可以通过通过查询日志判断是否已经投递
2. 采用事务消息的投递方式。

### 从 Consumer 端考虑

1. Consumer 维护可持久化的 offset，用于标记已经完成消费的消息为止。
2. 保证业务处理完成后再返回 ACK，如果消费者在处理过程中发生崩溃故障，RocketMQ 会将消息分配给别的消费者去处理。

## 消息顺序消费

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

## 延迟队列

通过源码可以发现 RocketMQ 默认只支持 18 个延迟级别：

```java
public class MessageStoreConfig {
    private String messageDelayLevel = "1s 5s 10s 30s 1m 2m 3m 4m 5m 6m 7m 8m 9m 10m 20m 30m 1h 2h";
}
```

> 可以通过配置文件修改`messageDelayLevel`延迟级别，修改完成后重启 RocketMQ 即可。

这样做的原因：RocketMQ 会把消息按照延迟时间段发送到指定的队列中，然后定时对这些队列进行轮询。如果发现消息到期，就把该消息发送到指定 Topic 的队列中。
因为同一延迟队列中消息的延迟时间是一致的，且消息是按照消息到期时间升序排序的，所以可以保证消息消费的有序性。

在 Broker 启动的时候，默认情况下，会根据 18 个延迟级别分别创建 18 个 DeliverDelayedMessageTimerTask 定时任务，在任务执行的时候，调度任务会计算距离下一次的延迟时间，然后创建一个新的定时任务。

由于 Timer 是单线程的，在延迟级别数量多的情况下可能会出现一系列问题，所以在 RocketMQ 4.9.2 之后的版本，使用了 ScheduledExecutorService 代替 Timer 执行定时任务，~~通过 scheduleWithFixedDelay 方法代替创建创建大量非必要对象，降低 GC 压力~~。

## 消息积压

消息积压一般是由于消费者的消费速度远小于生产者发送消息的速度，导致消息队列中存在大量消息。

1. 消费者代码实现存在缺陷。解决方案：修改代码。
2. 受限于数据库的性能瓶颈。解决方案：优化数据库操作。
3. 受限于其他服务的性能瓶颈。解决方案：待上游服务解决。
4. 突发场景，如：优惠活动导致订单暴增，热点事件。解决方案：消费者临时扩容。
