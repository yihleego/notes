# Kafka

## Kafka 基础架构与核心概念

### 请详细解释 Kafka 的核心组件（Broker、Topic、Partition、Replica、Zookeeper/KRaft）的作用和内部关系。

#### Topic（主题）

- 是什么：逻辑上的“数据类别/流”。例如 orders、user-events。
- 作用：把同一类消息归到一起，供生产者写入、消费者订阅。
- 内部要点：
    - Topic 本身不存数据，数据真正存放在 Partition（分区）里。
    - 可以配置保留策略（按时间/大小）、压缩、清理策略（delete/compact）等。

#### Partition（分区）

- 是什么：Topic 的物理分片，是 append-only 日志文件 的序列，每条消息在分区内有单调递增的 offset（偏移量）。
- 作用：提供并行度与扩展性（多个分区可并发读写），并提供分区内的强顺序（同一分区内消息按写入顺序读取）。
- 内部要点：
    - Leader/Follower：每个分区在某个 Broker 上有一个 Leader 副本，客户端（生产/消费）默认与 Leader 交互；其余副本是 Follower，从 Leader 同步数据。
    - 高水位（High Watermark, HW）：Follower 成功同步到某位置后，Leader 才把这位置之前的数据对消费者“可见”；这保证了副本一致性与可恢复性。
    - 消费位点：消费者在每个分区上有各自的消费 offset（通常以消费者组为粒度保存在内部主题 __consumer_offsets 中）。

#### Replica（副本）

- 是什么：同一分区在不同 Broker 上的拷贝。副本数由 replication.factor 指定，例如 3 表示 1 个 Leader + 2 个 Follower。
- 作用：提供容错与高可用。任一 Broker 宕机，控制层可以把该分区的 Leader 角色切到其他同步副本。
- 内部要点：
    - ISR（In-Sync Replicas，同步副本集合）：与 Leader 跟进足够快的副本集合。只有在 ISR 中的副本才有资格被选为新 Leader。
    - acks & min.insync.replicas：生产者 acks=all + 主题/集群设置 min.insync.replicas 能保证写入需要至少 N 个副本确认，增强一致性（代价是更高写延迟与更可能的写拒绝，在故障时保护数据）。

#### Broker（代理/服务器）

- 是什么：Kafka 集群中的单个服务器进程。一个集群通常由多个 Broker 组成（编号如 broker.id=1,2,3…）。
- 作用：真正持久化分区数据、处理生产/消费请求、执行保留/清理、管理副本同步等。
- 可能承担的角色（逻辑角色，非硬绑定）：
    - 分区 Leader/Follower 的承载者：具体到每个分区，哪个 Broker 承载它的 Leader/Follower。
    - Group Coordinator：为消费者组服务，负责成员加入/离开、再均衡（rebalance）、提交 offset 等。
    - Transaction Coordinator：为事务性生产者管理事务状态（内部主题 __transaction_state）。
    - Controller（控制器）：负责分区 Leader 选举、分配、故障转移、元数据变更广播等。
        - 在 ZooKeeper 架构下：某个 Broker 会被选为集群 Controller；
        - 在 KRaft 架构下：Controller 由 **控制器节点（组成一个 Raft 仲裁组）**承担，详见下一节。
- 分区/副本分布：控制层根据策略（如 rack awareness、均衡目标）把分区的副本分散到不同 Broker，避免单点与同机架风险。

#### ZooKeeper vs. KRaft（控制/元数据平面）

Kafka 有两代“控制/元数据”架构，用来维护集群元数据（有哪些 Broker、有哪些 Topic/Partition、谁是 Leader、ACL/配置等）并驱动控制流程（如选主、再均衡）。

5.1 ZooKeeper（旧架构）

- 结构：外部的 ZooKeeper 集群（奇数台）存储 Kafka 的元数据和临时会话信息；Kafka 集群中的一个 Broker 当选为 Controller，通过 ZooKeeper 完成 Leader 选举与元数据更新，并把变更广播给其他 Broker。
- 特点：
    - 控制面依赖外部系统（ZooKeeper），运维上需要部署/监控两套组件。
    - 元数据一致性由 ZooKeeper 的 ZAB 协议保障。

5.2 KRaft（Kafka Raft 模式，现行架构）

- 结构：Kafka 不再依赖 ZooKeeper，而是在 Kafka 进程内部引入 Raft 共识。集群有一组 Controller 节点（可以与 Broker 复用进程或分离部署），它们组成 Raft 仲裁组，维护一份元数据日志（metadata log）。
- 角色划分：
    - Controller 进程/节点：运行 Raft，提交并复制“元数据变更”（创建 Topic、分配分区、Leader 切换、ACL/配置变更等）。
    - Broker：订阅并缓存这份元数据，按照指令承载分区及其副本。
- 特点：
    - 去掉外部依赖，更容易部署、扩容、自动化。
    - 元数据强一致与事件化变更（顺序写入的 metadata log），提升大规模集群的可伸缩性与稳定性。
    - 故障时由 Raft 选主新的 Controller Leader，继续对外提供控制平面服务。

> 简记：
> ZooKeeper 时代 = “ZK 存元数据 + 某个 Broker 充当 Controller”；
> KRaft 时代 = “内置 Raft 控制层（Controller 仲裁组） + Broker 专注数据面”。
> 两者的目标都是为 Broker 层的数据副本与客户端 I/O 提供稳定的一致的元数据与控制面。

#### 组件之间的关系（一图流思维）

```
[Producer/Consumer]
        │  通过元数据发现 & 读写路由（客户端向集群查询分区的Leader在哪）
        ▼
   ┌────────── 控制/元数据平面 ──────────┐
   │  ZooKeeper（旧） 或 KRaft Controller（新） │
   │  - 维护Topic/Partition/副本/ACL/配置       │
   │  - 进行Leader选举、再均衡、故障转移        │
   └──────────────────────────────────┘
        │  将分配与变更下发给 Broker
        ▼
   ┌──────────── 数据平面 ────────────┐
   │   多个 Broker（broker.id=1,2,3…） │
   │   - 承载分区的Leader/Follower     │
   │   - 处理 Produce/Fetch 请求       │
   │   - 同步副本(Replica, 组成ISR)    │
   └──────────────────────────────────┘
        │
        ▼
 [Disk/Filesystem]（每个Broker本地顺序写日志段）
```

#### 常见“串联问题”与关键配置

- 吞吐 vs. 一致性：
    - acks=0/1/all（生产端）与 min.insync.replicas（服务端）一起决定写入确认策略；
    - acks=all + min.insync.replicas>=2 提高可靠性，但在故障期可能拒绝写（保护数据）。
- 可用性：
    - 副本数 replication.factor>=3（生产常见）+ 机架感知（rack-aware） 副本分布，容忍单机/单机架故障。
    - 通过 Controller 做分区迁移/再均衡，保持负载与容量均衡。
- 顺序性：
    - 仅在“分区内”保证顺序。需要全局顺序就减少分区数或使用单分区（牺牲并行度）。
    - 生产端用**键（key）**做分区路由，可保证同 key 的事件总落到同一分区。
- 存储与清理：
    - retention.ms / retention.bytes 控制保留；cleanup.policy=delete|compact 控制删除或压缩（按 key 保留最新值）。
- 消费者组协调：
    - Group Coordinator 在某个 Broker 上负责该组的再均衡，把分区分配给各消费者；消费者把位点提交到 __consumer_offsets。

#### 一句话总览（速查表）

| 概念                | 作用      | 关键点                                  |
   |-------------------|---------|--------------------------------------|
| Topic             | 逻辑数据流分组 | 上层概念，真正数据在分区                         |
| Partition         | 物理日志分片  | 分区内顺序；决定并行度；Leader/Follower          |
| Replica           | 分区副本    | 容错；ISR；与 acks/min.insync.replicas 配合 |
| Broker            | 服务器节点   | 承载分区与副本；处理 I/O；可能担任协调/事务/控制等逻辑角色     |
| ZooKeeper / KRaft | 控制/元数据层 | 维护元数据、做选主与再均衡；KRaft 内置 Raft、无外部 ZK   |

### Leader—Follower 副本同步机制如何工作？ISR 的维护策略是什么？

### Kafka 为什么选择顺序写？为什么能达到高吞吐？

#### 1. Partition 的物理结构

Kafka 的 Topic 被划分为多个 Partition，每个 Partition 对应磁盘上的一个目录。这个目录下保存若干个 Segment 文件：

- 每个 Segment 包含一个 .log 数据文件（存放消息内容）；
- 以及对应的 .index 索引文件（记录 offset 与物理位置的映射）。

这种结构带来两个好处：

- 并行性高：不同 Partition 可分布在不同 Broker / 磁盘上，实现水平扩展；
- 单分区顺序写：同一 Partition 内的消息严格有序，只需顺序追加写文件尾部。

#### 2. 顺序写带来的高性能

传统消息队列或数据库往往执行随机写（需要在文件中间插入或更新记录），而 Kafka 只追加数据：

- OS 层面顺序写非常快（顺序写速度可达随机写的几十倍）；
- 充分利用 OS Page Cache：数据写入后会缓存在内存中，由内核异步 flush；
- 减少寻道时间，特别适合机械硬盘。

这使得 Kafka 的写入几乎是“线性增长”的，写吞吐可以轻松达到数百 MB/s。

### Kafka 零拷贝（Zero Copy）与 sendfile

#### 1. 普通数据传输流程

传统的数据从磁盘到网络传输通常需要 4 次用户态/内核态切换、4 次数据拷贝：

```
Disk -> Kernel buffer -> User buffer -> Kernel socket buffer -> NIC
```

这种频繁的内存拷贝和上下文切换非常耗 CPU。

#### 2. Kafka 的 sendfile 优化

Kafka 采用了 Linux sendfile 系统调用 来实现零拷贝传输：

- 调用 sendfile(fd_out, fd_in, offset, len)；
- 数据直接从磁盘页缓存传输到 socket buffer；
- 避免进入用户态，不需要中间缓冲区拷贝。

数据流变为：

```
Disk (page cache) -> Kernel socket buffer -> NIC
```

这样：

- 降低了 CPU 消耗；
- 减少了上下文切换；
- 提升了传输吞吐量和延迟性能。

Kafka 的文件格式与网络协议设计得非常契合，使得消息批次（MessageBatch）可直接映射为磁盘连续区域，从而支持 sendfile 的直接发送。

### Page Cache 在 Kafka 中的作用是什么？

#### 一、什么是 Page Cache

Page Cache 是操作系统（如 Linux）用来缓存磁盘文件内容的内存区域。
当应用程序（如 Kafka broker）从磁盘读写文件时，实际上操作系统会把这些数据先放到内存中的 Page Cache 中，以减少真正的磁盘 I/O 操作。

- 读缓存：如果数据在 Page Cache 中，读取就不需要访问磁盘（称为“命中缓存”）。
- 写缓存：写入操作通常会先写入 Page Cache，然后异步地刷入磁盘。

#### 二、Kafka 与 Page Cache 的交互方式

Kafka 的设计充分利用了操作系统的 Page Cache，而不是自己实现一套缓存系统。 具体表现为：

1. Kafka 的日志文件（segment files）是顺序写入磁盘的。
    - 当生产者发送消息时，Kafka 只是调用文件系统的 write()，数据被写入 Page Cache。
    - 操作系统稍后会异步地将数据刷到磁盘（fsync() 控制刷盘时机）。
2. 消费者读取消息时，Kafka 直接通过文件 I/O 读取 segment 文件。
    - 如果数据仍然在 Page Cache 中（比如刚写入的消息），读操作可以直接从内存返回，速度极快。
    - 如果不在缓存中，才会触发真实的磁盘读取。

Kafka 不做额外的磁盘缓存，而是完全依赖 Linux 的 Page Cache：
这种“顺序写 + 操作系统缓存”策略带来了极高的 IO 性能，同时简化了 Kafka 的代码复杂度。

### Kafka 批量处理与压缩

Kafka 通过 批量写入 / 读取 和 消息压缩（如 Snappy、LZ4） 进一步减少系统调用次数与网络开销：

- 多条消息聚合为一个 batch；
- 一次写磁盘、一条索引；
- 网络一次发送多条消息。

结果是：高吞吐 + 低延迟 + 优秀的 CPU 利用率。

### Kafka 的存储结构（Segment、Index、TimeIndex、Log）如何设计？

## 生产者（Producer）机制与优化

### Producer 发送消息的完整流程是什么？

#### 1. 初始化 Producer

应用创建一个 KafkaProducer 实例，加载配置（如 bootstrap.servers, acks, retries, key.serializer, value.serializer 等）。
Producer 会：

- 建立与 Kafka Broker 的连接；
- 启动 I/O 线程池；
- 维护元数据缓存（Topic、分区信息）。

#### 2. 序列化消息

当应用调用 send() 方法发送消息时，Producer 会：

- 调用 key/value 序列化器；
- 将对象转换为字节数组；
- 可选：执行分区选择逻辑（默认使用 key 的哈希确定分区）。

#### 3. 压缩与分区

Producer 会将同一分区的消息临时存入 RecordAccumulator（内存缓冲区）。
当达到以下任一条件时，会触发批量发送：

- 批次大小 (batch.size) 满；
- 等待时间 (linger.ms) 到期；
- 主线程主动调用 flush()。

同时可以对批次启用压缩（compression.type=gzip/snappy/lz4/zstd）。

#### 4. 发送请求到 Broker

Sender 线程异步从 RecordAccumulator 中取出批次：

- 根据元数据确定目标 Broker；
- 构造 ProduceRequest；
- 通过 Socket 发送到对应 Broker 的网络通道；
- 若 acks=all，则等待所有副本确认；
- 若 acks=1，仅等待 Leader 副本确认；
- 若 acks=0，不等待确认直接返回。

#### 5. Broker 处理请求

Broker 接收到 ProduceRequest 后：

- 写入日志文件（LogSegment）；
- 更新偏移量；
- 将消息复制到 Follower 副本；
- 返回 ProduceResponse 给 Producer。

#### 6. Producer 收到响应

Producer 根据响应结果：

- 成功则触发回调（Callback.onCompletion()）；
- 失败（网络、超时、Leader 不可用）则重试；
- 达到最大重试次数后，将错误返回给调用方。

#### 7. 可选的幂等性与事务保证

若开启：

- enable.idempotence=true：Producer 有唯一 PID，可防止消息重复；
- 事务（transactional.id）：保证跨分区的原子性提交。

#### 流程总结：

```
应用层 -> Producer API -> 序列化 -> 缓冲区 -> Sender 线程 -> Broker -> 确认响应 -> 回调
```

### Kafka 中的消息路由（Partitioner）机制如何决定消息写入哪个分区？

情况 1：指定了 partition（显式指定）

如果 ProducerRecord 明确指定了 partition 值，那么消息直接写入该分区。

```java
ProducerRecord<String, String> record =
        new ProducerRecord<>("topicA", 2, "key1", "value1");
```

情况 2：指定了 key，但未指定 partition

默认分区器会根据 key 的哈希值 进行分区：

```
partition = hash(key) % numPartitions
```

> Kafka 默认使用 Murmur2 哈希算法，它在分布和性能上较优。

情况 3：未指定 key 和 partition

Kafka 会使用一种**轮询（Round-Robin）**机制，但带有缓存优化：

- Kafka Producer 内部维护了上次使用的分区索引；
- 对同一批次（Batch）内的消息，轮询选择下一个可用分区；
- 可以较好地在分区间分摊负载。

### acks=0/1/all 的差异是什么？常见的可靠性配置组合有哪些？

#### acks 的三种取值

| acks 值          | 含义                                                | 数据可靠性 | 延迟    | 丢数据风险                  |
|-----------------|---------------------------------------------------|-------|-------|------------------------|
| **0**           | 生产者**不等待**任何服务器确认消息是否成功写入。                        | ❌ 极低  | ✅ 最低  | ✅ 极高                   |
| **1**           | 生产者等待 **leader 分区副本**写入成功确认，但**不等待 follower 同步**。 | ⚠️ 中等 | ⚠️ 较低 | ⚠️ 可能丢数据（leader 宕机未同步） |
| **all**（或 `-1`） | 生产者等待 **所有 ISR（in-sync replicas）副本**都确认写入成功。      | ✅ 最高  | ❌ 较高  | ❌ 最低（除非所有副本都挂掉）        |

#### 可靠性机制的补充配置

Kafka 的可靠性不仅由 acks 决定，还与以下参数组合使用：

| 配置项                                         | 含义                  | 推荐设置（高可靠场景）             |
|---------------------------------------------|---------------------|-------------------------|
| **`retries`**                               | 发送失败的重试次数           | 较高（如 3～10）              |
| **`enable.idempotence`**                    | 启用幂等生产者，避免重复消息      | `true`                  |
| **`max.in.flight.requests.per.connection`** | 每个连接上允许未确认请求的最大数量   | ≤5（启用幂等时 Kafka 会自动限制）   |
| **`min.insync.replicas`**                   | ISR 中最少要有多少副本确认才算成功 | 通常设为 `2`（配合 `acks=all`） |
| **`replication.factor`**                    | 分区副本数               | ≥3（生产环境标准）              |

#### 常见配置组合与使用场景

| 场景               | 示例配置                                                       | 可靠性   | 性能     | 说明                            |
|------------------|------------------------------------------------------------|-------|--------|-------------------------------|
| **低延迟日志收集**      | `acks=0`                                                   | ❌ 最低  | ✅ 极高   | 适用于丢部分数据可接受的场景（如 clickstream） |
| **一般业务日志**       | `acks=1, retries=3`                                        | ⚠️ 中等 | ⚠️ 较好  | 常见默认配置；可能在 leader 崩溃时丢数据      |
| **关键业务数据（推荐）**   | `acks=all, min.insync.replicas=2, enable.idempotence=true` | ✅ 最高  | ❌ 较高延迟 | 保证消息不丢不重（强一致性）                |
| **高可靠流处理/交易类数据** | `acks=all, retries>3, replication.factor≥3`                | ✅✅ 极高 | ⚠️ 较慢  | 企业级高可用配置                      |

### 如何处理 Producer 幂等性？幂等模式下仍有哪些异常场景？

#### 背景问题

在 Kafka 中，如果生产者发送消息后没有收到确认（ack），可能会自动重试。这会导致：

- 消息被多次写入同一分区；
- 消费者可能读到重复的数据。

这在金融交易、订单系统等对“唯一性”要求高的场景中是不可接受的。

#### 实现原理

Kafka 从 0.11 版本开始引入幂等生产者功能。其核心机制包括：

1. Producer ID（PID）
   每个生产者实例启动时会被分配一个唯一的 PID。
2. Sequence Number（序列号）
   每个消息在发送时带有一个递增的序列号。
3. Broker 去重逻辑
   Broker（Kafka 服务器端）会根据 (PID, Partition, SequenceNumber) 判断消息是否重复。
   如果发现相同的组合出现多次，则丢弃重复的消息。

#### 异常场景

1. 仅在单个分区内生效
   幂等机制的 (PID, Partition, SequenceNumber) 唯一性只在单分区内判断。
   如果同一条逻辑消息被发送到不同分区（或不同主题），Kafka 并不会认为它们重复。
2. 生产者重启后 PID 失效
   每个生产者实例重启后，会分配新的 Producer ID (PID)。
   旧的 PID 对应的幂等状态就丢失了，Broker 认为是一个新的生产者。
3. 消息可能丢失（幂等不保证 Exactly Once）
   幂等机制防止“多写”，但不保证消息“必达”：
   如果 acks=0 或 acks=1，Broker 宕机可能导致数据丢失；
   如果启用了幂等但没有合适的重试策略，仍可能丢数据。
4. 乱序问题
   虽然幂等机制要求 max.in.flight.requests.per.connection <= 5，但当启用多线程或多个分区发送时，仍可能出现消息顺序错乱。

### Producer 批量发送（batch.size、linger.ms）如何影响吞吐？

### 如何优化 Producer 吞吐？给出至少五种方式。

### 压缩策略（gzip、snappy、lz4、zstd）如何选择？

### Producer 端消息重试机制是怎样的？会导致消息重复吗？

## 消费者（Consumer）机制与优化

### Consumer Group 的协调协议如何工作？

### Rebalance 发生的常见原因是什么？如何减少 Rebalance？

Kafka 的 Rebalance（再均衡） 是消费者组（Consumer Group）在成员或分区分配发生变化时的一种自动协调行为。它会导致分区的重新分配，从而影响消息消费的连续性。下面是常见触发 Rebalance 的原因及解释：

#### 一、Rebalance 发生的常见原因

1. 成员变化
   这是最常见的触发原因：
    - 消费者组成员变化：新消费者加入或老消费者离开。
    - 节点故障或恢复：节点宕机、重启、网络闪断导致组协调器检测到心跳丢失。
    - 滚动升级：应用或 Broker 重启过程中，消费者临时离组。
   
2. 元数据变化
   当集群拓扑或主题配置变更时，也可能触发 Rebalance：
    - 新建或删除 Topic。
    - Topic 的 分区数增加。
    - Broker 下线或上线。
    - Partition Leader 发生迁移。
    
3. 心跳或 Session 过期
   消费者或节点未在规定时间内向协调器发送心跳，协调器认为该成员失效，从而触发再平衡。
   Kafka 中通过 session.timeout.ms 和 heartbeat.interval.ms 控制检测机制。

4. 配置或协调器异常
    - Coordinator 重新选举。
    - 状态存储不一致或 Group Metadata 丢失。
    - 应用端频繁调用 poll() 不及时或阻塞导致的 Rebalance。

#### 二、Rebalance 带来的影响

- 消费中断：分配期间无法拉取消息，导致短暂停顿。
- 性能抖动：大量任务迁移、连接重建、缓存失效。
- 重复消费：位移提交点不一致时可能出现重复消息。

#### 三、减少 Rebalance 的方法

1. 优化心跳与会话参数
    - 增大 session.timeout.ms（允许消费者临时延迟心跳）
    - 合理配置 max.poll.interval.ms，避免应用端超时。
    - 保证 poll() 调用及时，防止阻塞。

2. 控制成员变动频率
    - 减少消费者频繁上下线（如避免动态创建短生命周期的消费者）。
    - 滚动升级时使用 分批重启，而非全量重启。
    - 使用 静态成员 ID（Static Membership），避免重启时被视为新成员。

3. 避免频繁修改 Topic 配置
    - 谨慎增加分区数或删除主题。
    - 避免在高峰期做 Broker 扩缩容。

4. 监控与调优
    - 监控 Group Rebalance 次数、持续时间。
    - 优化协调器稳定性，确保 ZooKeeper / Kafka Controller / Flink JobManager 稳定。
    - 对任务或分区进行 手动负载均衡（manual assignment），减少自动分配压力。

5. 使用 Incremental Rebalance（增量再均衡）

对于支持的系统（如 Kafka 2.4+），增量再均衡允许只调整受影响的分区，而不是全量重分配，大幅减少中断。

### Offset 提交机制（自动 vs 手动）有什么区别？为什么自动提交 offset 可能引发消息丢失？

1. 性能与易用性权衡
    - 自动提交的设计是为了让大部分简单消费者（比如日志分析、监控消费）可以**“开箱即用”**，不必处理提交逻辑。
    - 手动提交虽然安全，但需要在业务处理成功后再提交，否则容易漏提交或重复消费，对初学者来说负担较重。
    - 对很多场景（例如监控、ETL、日志聚合）来说，少量重复消费或丢失都可以容忍，因此 Kafka 默认选择了更高性能、易于使用的自动模式。

2. 自动提交的延迟较小，吞吐量更高
    - 自动提交是异步定时任务（默认每 5 秒提交一次），不阻塞消费线程。
    - 手动提交的 commitSync() 是阻塞调用，会等待服务端确认提交成功，这在高并发场景下是性能瓶颈。
    - 即使使用 commitAsync()，也仍然会引入一定的协调开销（尤其是高吞吐 topic）。

因此对于需要极高吞吐量的系统（如日志采集、指标流），自动提交更合适。

#### 一、基本概念：Offset 提交

- Offset：Kafka 中每条消息都有一个顺序编号（offset），标识它在分区中的位置。
- 提交（commit）offset：表示“我已经成功处理到这个 offset 之前的所有消息”，下次重启时从这个 offset 继续消费。

#### 二、自动提交（enable.auto.commit = true）

Kafka 提供自动提交机制：

- 工作机制：
    - 消费者会定期（默认每 5 秒）自动将当前 poll() 出来的最大 offset 提交到 Kafka。
    - 提交时机由配置项 auto.commit.interval.ms 决定。
- 优点：
    - 简单，无需额外代码。
- 缺点（⚠️ 可能导致消息丢失）：
    - 提交 offset 是“基于时间”的，而不是“基于处理成功”的。
    - 若消费者在提交前崩溃或重启，可能出现两种情况：
        1. 消息丢失：如果消费者在自动提交 offset 后，还没真正处理完消息就挂了，重启后会从新 offset 开始 → 未处理的消息被跳过。
        2. 消息重复：如果消费者挂掉前没来得及提交 offset，重启后会从旧 offset 重新消费 → 已处理的消息被重复消费。

#### 三、手动提交（enable.auto.commit = false）

- 工作机制：
    - 由开发者在业务逻辑中显式调用 commitSync() 或 commitAsync() 提交 offset。
    - 可灵活控制提交时机（如：每处理完一批消息后）。
- 优点：
    - 可以在“消息处理成功后”再提交 offset，从而保证 “至少一次”（at-least-once）语义。
    - 可与事务结合，实现 “精确一次”（exactly-once）语义。
- 缺点：
    - 代码复杂度更高。
    - 若忘记提交 offset，重启后会重复消费旧数据。

### 如何实现精确一次消费（exactly once）？

见下方

### 如何保证顺序消费？

Kafka 只保证每个分区内部的消息是有序的（FIFO：先写先读）。
这意味着：

- 同一个 Partition 内，Producer 发送的消息会被按顺序存储；
- Consumer 从该 Partition 读取时，也会按这个顺序消费；
- 不同 Partition 之间，Kafka 不保证全局顺序。

Kafka 仅保证分区内的顺序性，Consumer 端要做到顺序消费，需满足以下三点：

1. 同一分区只能被一个 Consumer 消费；
2. Consumer 对该分区的消息按顺序处理（单线程）；
3. 消息的业务顺序通过 Key 控制分区映射。

### 你如何处理 Consumer 延迟（lag）快速增长的问题？

## 可靠性、消息语义与事务

### Kafka 在哪些情况下可能出现消息丢失？如何避免？

#### 1. 生产者端丢失

场景：

- Producer 向 Broker 发送消息时发生网络中断；
- 未开启消息确认机制（acks=0 或 acks=1）；
- 异步发送（send()）后程序崩溃，消息未成功发送；
- 未开启重试机制或重试次数过少。

说明：

- acks=0：生产者不等待任何确认，极快但风险高；
- acks=1：只要 Leader 写入成功即返回确认，如果 Leader 崩溃，Follower 尚未同步，会丢消息；
- acks=all（或 acks=-1）：Leader 和所有 ISR 同步成功才确认，最安全。

解决方案：

- ✅ 设置 acks=all；
- ✅ 开启 retries，设置合理的重试次数（如 retries=Integer.MAX_VALUE）；
- ✅ 开启幂等性（enable.idempotence=true）；
- ✅ 使用同步发送（producer.send(record).get()）在关键场景下确保发送成功。

#### 2. Broker 端丢失

场景：

- Broker 崩溃后数据未及时同步到副本；
- 日志未落盘（log.flush.interval.messages 或 log.flush.interval.ms 太大）；
- 磁盘损坏或 Kafka 未正确启用 fsync。

说明：
Kafka 的可靠性取决于 副本机制（Replication） 和 ISR（In-Sync Replica）。
如果 Leader 崩溃，新的 Leader 未包含全部消息，则这些消息会丢失。

解决方案：

- ✅ 设置 min.insync.replicas >= 2；
- ✅ acks=all 时，确保有足够的 ISR；
- ✅ 监控 ISR 变化，防止过多副本掉队；
- ✅ 使用可靠的存储介质（如 SSD），定期检查磁盘健康。

#### 3. 消费者端丢失

场景：

- 消费者在处理消息后崩溃，但 offset 已提交；
- 消费者自动提交偏移量（enable.auto.commit=true）；
- 手动提交时未确保“处理完成后再提交”。

### Kafka 会不会出现消息重复？为什么？如何处理？

会，自动提交，改为手动提交或业务做幂等

### Kafka 的“至少一次”、“最多一次”、“精确一次”语义分别如何实现？

#### At-least-once（至少一次）

每条消息至少被处理一次，可能重复但不会丢失。

实现机制：

- Producer：
    - 设置 acks=all，确保消息写入所有副本后再确认。
    - 失败时重试发送（retries > 0）。
- Consumer：
    - 在处理完消息后再提交 offset。
    - 如果在处理后提交前崩溃，消费程序重启后会重新处理消息（导致重复）。

#### At-most-once（最多一次）

消息最多被处理一次，可能会丢失，但绝不会重复。

实现机制：

- Producer：
    - 不启用 acks 或设置 acks=0（即不等待 broker 的确认）。
    - 如果消息发送失败，不重试。
- Consumer：
    - 在处理消息之前立即提交 offset（自动提交或提前提交）。

#### Exactly-once（精确一次）

每条消息只被处理一次，不丢失、不重复。

实现机制（从 Kafka 0.11 起支持）：

Kafka 实现精确一次语义（EOS）通过以下机制：

1. 幂等 Producer
    - 启用参数：`enable.idempotence=true`
    - Kafka 会为每个 Producer 分配唯一 PID（Producer ID）；
    - 每个 (PID, partition) 对应的消息会带有单调递增的序列号；
    - Broker 会检测并丢弃重复的消息。
2. 事务性 Producer
    - 用于保证跨分区或跨 Topic 的原子性：`transactional.id=my-transactional-id`
    - 若出现异常，则执行 producer.abortTransaction()；
    - Kafka 保证：要么事务中的所有消息都可见，要么都不可见。
3. Consumer 与 Producer 的原子性（消费 → 生产）
    - 当一个应用既消费又生产（如流处理器）时，可以使用：`isolation.level=read_committed`
    - 配合 Kafka Streams 或使用事务性 producer，可以保证：“消费偏移量提交” 与 “生产新消息” 在同一个事务中完成。

Kafka 通过 “幂等生产者 + 事务性写入 + 原子提交 offset” 实现了端到端的精确一次语义（Exactly Once）。

### Kafka 事务（Transaction）如何实现？事务消息在内部是如何写入的？

Kafka 的事务主要解决两个问题：
1.	生产者的幂等性：确保同一条消息在重试时不会被重复写入。
2.	跨分区、跨 topic 的原子性：确保一组消息要么全部成功（commit），要么全部失败（abort）。

例如，生产者向多个分区发送消息，希望要么全部成功提交，要么都不生效，这就需要事务。

### 消息幂等性与事务的差异是什么？

## Broker 内部机制

### Kafka 的高水位（HW）、LEO、LSO 分别代表什么？

#### HW（High Watermark）

HW 表示“高水位线”，即 所有副本都同步到的最大 offset。
消费者（Consumer）只能消费到 HW 之前的消息（包括 HW 对应的 offset）。
举例：

- Leader 的日志写到了 offset 100（LEO = 100）
- Follower 只同步到 offset 95
- 那么 HW = 95

#### LEO（Log End Offset）

LEO 表示“日志末端偏移量”，即 下一条待写入消息的 offset。
换句话说，LEO 指向日志中“下一个将被写入的位置”，而不是最后一条消息的 offset。

举例：

- 如果当前分区中有 offset 为 0～99 的消息，
- 那么 LEO = 100（下一个要写的 offset）。

#### LSO（Last Stable Offset）

LSO 表示“最后稳定偏移量”，主要在 启用事务（transaction） 的场景下使用。
它指的是：所有事务性消息中，最后一个已经提交（committed）或中止（aborted） 的事务之前的最大 offset。

作用：

- 对事务性消费者（隔离级别为 read_committed）来说，只能读取到 LSO 之前的消息。
- 防止消费到未提交的事务数据

### Follower 如何同步 Leader

Kafka 的每个 Partition（分区） 都有一个 Leader 副本和若干个 Follower 副本。Leader 负责处理所有读写请求，而 Follower 仅从 Leader 拉取数据，保持与其同步。

1. Leader 写入数据
   Producer 把消息发送给 Leader 副本，Leader 把数据写入本地日志（log segment）。
2. Follower 拉取数据
   每个 Follower 都会周期性地向 Leader 发送 Fetch 请求（类似消费者的拉取），请求从上次同步的位置开始获取新的数据。
3. Follower 写入日志
   Follower 收到消息后，将其追加写入自己的本地日志文件中，并更新自己的 High Watermark（高水位）。
4. Leader 维护同步状态
   Leader 跟踪每个 Follower 的最新 offset（即同步到的位置）。只有当某条消息被所有 “活跃的、同步中的 Follower” 成功复制后，Leader 才会将这条消息标记为 “已提交”（committed）。

### 为什么需要 ISR（In-Sync Replica）

ISR（同步副本集合）是 Kafka 中用来保证 数据一致性 与 可用性 的机制。

1. ISR 定义

   ISR 是一个列表，包含了：
    - Leader 自身；
    - 所有与 Leader 同步状态的 Follower（延迟在可接受范围内）。

   换句话说，ISR 表示当前 “可靠同步”的副本集合。

2. ISR 的作用

   （1）保证数据一致性
   Kafka 的 “已提交消息” 只有当 ISR 中的所有副本都复制了该消息后，才会对消费者可见。
   这样即使 Leader 宕机，新的 Leader（从 ISR 中选出）也一定拥有全部已提交的数据，避免数据丢失。

   （2）控制副本滞后
   Follower 如果落后太多（超出 replica.lag.time.max.ms 配置时间），会被移出 ISR。
   这样可以防止因为慢副本导致的系统整体性能下降，也确保 ISR 中的副本足够“新”。

   （3）Leader 选举安全
   当 Leader 崩溃时，Kafka 只会从 ISR 中选出新的 Leader。
   这能确保新 Leader 至少包含所有已提交的数据，从而保证高一致性（类似于分布式系统的 “安全选举” 原则）。

### Replica Lag 较大时系统如何处理？

### Controller 的职责是什么？选举过程如何运作？

### Log Compaction 内部原理是什么？什么时候适用？

### 消息删除策略（Retention Policy）有哪些？如何实现？

### Kafka 的 Rack Awareness 如何保证高可用？

## Kafka 运维与调优

### Kafka 如何扩容 Topic Partition？扩容后的消息顺序如何保证？

1. 扩容方式
   通过命令 kafka-topics --alter --topic <topic> --partitions <new_count> 增加分区数量。Kafka 不会重新分配已有消息，只是增加新的空分区，供新写入数据使用。
2. 分区映射变化
   Producer 的分区选择逻辑通常是 partition = hash(key) % partition_count。扩容后 partition_count 变化，key 到分区的映射会改变，导致同一 key 的后续消息可能落入不同分区。
3. 消息顺序保证
   - 同分区内顺序仍然保证：Kafka 的顺序语义是“分区内有序”，这一点不变。
   - 跨分区顺序无法保证：若扩容后 key 映射变动，同一 key 的新旧消息可能分布在不同分区，从而丢失全局或 per-key 顺序。
   - 要保持顺序：必须使用稳定的 key 或自定义分区器，确保同一 key 始终映射到相同分区（可在应用层做 hash 映射缓存）。

### Kafka 扩容 Broker 时需要注意什么？可能会导致什么问题

#### 扩容过程中的注意事项

1. 增加 Broker 并不会自动分配数据
   - 新加入的 Broker 默认不会有任何分区或副本。
   - 必须通过 kafka-reassign-partitions 工具 或 自动负载均衡工具（如 Cruise Control） 来迁移分区。
2. 分区迁移会带来网络与磁盘负载
   - 在迁移过程中会大量的网络 I/O（Broker 间复制）与磁盘写入。
   - 若迁移过快，可能会影响线上生产消费性能。
   - 建议使用 --throttle 参数限制迁移带宽（如每 Broker 限速 50MB/s）。
3. 迁移过程中副本状态监控
   - 持续观察 UnderReplicatedPartitions、OfflinePartitionsCount 等指标。
   - 如果迁移中断（Broker 挂掉或网络闪断），要重新执行或恢复迁移任务。
4. Leader 分布优化
   - 迁移完成后，建议执行一次 Leader rebalancing（kafka-preferred-replica-election）以均衡负载。

#### 可能出现的问题

1. 数据与负载不均
   - 若分区未重新分配，新 Broker 长期空闲，老节点仍然高负载。
   - 可能导致部分节点磁盘满、延迟高。
2. 性能波动
   - 分区迁移期间，Broker 会进行大量数据复制，影响网络与磁盘性能。
   - 若无速率限制，可能导致生产延迟增加、消费者滞后。
3. Leader 过度集中
   - 若迁移后未重新选举 Leader，可能导致部分 Broker 承担过多请求。
4. 副本不同步风险
   - 磁盘或网络压力过大可能造成 ISR 收缩，影响高可用性。
5. 版本兼容问题
   - 若新增 Broker Kafka 版本与集群版本不一致，可能导致协议不兼容或控制器异常。

### 如何判断某个 partition 是热点分区？如何优化？

#### 查看各分区的消息流量差异

- 每秒消息生产速率 (BytesInPerSec, MessagesInPerSec)
- 每秒消息消费速率 (BytesOutPerSec, MessagesOutPerSec)

```
kafka-run-class kafka.tools.GetOffsetShell \
  --broker-list <broker_list> \
  --topic <topic_name> --time -1
```

#### 检查生产端（Producer）的分区分配逻辑

热点分区常由非均匀的分区策略引起，例如：

- 使用 key 进行分区，但 key 分布极不均匀；
- 使用 DefaultPartitioner 时某些 key 值过于集中；
- 或使用自定义 partitioner 时逻辑错误。

可以通过日志或埋点统计 key→partition 的映射分布。

#### 优化热点分区的策略

1. 优化分区策略（Producer 端）
    - 随机分区（不指定 key）：让 Kafka 自动均衡。
    - 哈希分区但注意 key 均匀性：确认 key 的取值分布是否接近 uniform。
    - 自定义分区器：如果 key 无法均匀分布，可在分区器中引入随机扰动，如：`partition = (key.hashCode() + ThreadLocalRandom.current().nextInt(n)) % numPartitions;`
2. 增加分区数量
   适用于热点集中在少量分区的情况
3. 重新分配分区（Rebalance）
   如果热点导致部分 broker 负载过重，可使用 kafka-reassign-partitions.sh

### Kafka 的常用监控指标有哪些？

### Kafka 吞吐下降的常见原因是什么？

### 如何处理磁盘使用率过高？

### Kafka checkpoint、snapshot 相关概念是什么？

### Log Cleaner 的工作机制和性能影响？

## Kafka Stream / Connect

### Kafka Streams 如何实现状态存储？

### Streams 的 Exactly Once 是如何实现的？

### KTable 与 KStream 的差异？

### Connect 中 Source Connector 和 Sink Connector 的内部机制？

### Connect 分布式模式下任务如何分配？

## 常见系统设计题

### 如何设计一个百万 TPS 的 Kafka 消息系统？

设计一个 百万 TPS（Transactions Per Second） 的 Kafka 消息系统，需要从 架构设计、硬件配置、Kafka 调优、网络设计、以及消息模型设计 等多个层面协同优化。下面是一个系统性思路。

#### 一、总体架构设计思路

1. **分布式集群架构**
    - Kafka 本身是水平可扩展的，通过 **Broker 分片 + Partition 分区** 实现高并发。
    - 设计时需根据目标吞吐量（100 万 TPS）反推集群规模。
    - 一般来说：
        - 1 台高配 Broker（64 核 CPU、256 GB 内存、NVMe SSD）可稳定支撑 10~20 万 TPS。
        - 100 万 TPS 需 **5~10 台 Broker** 起步，视消息大小而定。

2. **多主题分区**
    - Kafka 的吞吐能力来自 **分区（Partition）** 并行。
    - 每个 Partition 通常只能由单个 consumer thread 消费，因此要足够多的分区。
    - 经验值：每个分区可支撑 10k ~ 50k TPS。
    - → 为百万 TPS，需要约 **50~200 个分区**。

3. **高并发生产者（Producer Pool）**
    - 使用异步批量发送（`linger.ms`, `batch.size`）来聚合写入。
    - 通过多线程或多实例部署 Producer，打满分区的发送带宽。

4. **多消费者并行消费**
    - 消费端采用 **多 Consumer Group 或分区级别多线程**。
    - 避免单一消费者成为瓶颈。

#### 二、Kafka 集群配置与优化

1. 硬件层面

| 层级  | 推荐配置                       |
|-----|----------------------------|
| CPU | ≥ 32 核（高主频）                |
| 内存  | ≥ 128 GB（page cache 对性能关键） |
| 存储  | NVMe SSD（至少 2 GB/s 顺序写）    |
| 网络  | 25 Gbps 以太网（低延迟、高带宽）       |

2. Kafka Broker 参数

| 参数                            | 建议值           | 说明         |
|-------------------------------|---------------|------------|
| `num.network.threads`         | 8~16          | 网络 I/O 线程  |
| `num.io.threads`              | 16~32         | 磁盘 I/O 线程  |
| `socket.send.buffer.bytes`    | 1MB           | 发送缓冲区      |
| `socket.receive.buffer.bytes` | 1MB           | 接收缓冲区      |
| `socket.request.max.bytes`    | 100MB+        | 单请求最大字节数   |
| `num.replica.fetchers`        | 4+            | 副本同步线程数    |
| `log.segment.bytes`           | 1~2GB         | Segment 大小 |
| `log.retention.hours`         | 按业务需求调整       | 保留策略       |
| `log.cleaner.enable`          | false（写多读少场景） | 减少 GC 压力   |

3. JVM 调优

- `-Xms` = `-Xmx` = 16~32 GB，避免堆自动扩缩。
- `G1GC` 或 ZGC 垃圾回收器。
- 禁止 swap：`vm.swappiness=0`。

#### 三、生产者（Producer）优化

| 参数                                      | 推荐配置             | 说明       |
|-----------------------------------------|------------------|----------|
| `acks`                                  | 1 或 0            | 减少确认等待   |
| `batch.size`                            | 64KB~512KB       | 批量聚合发送   |
| `linger.ms`                             | 5~10ms           | 等待批次聚合   |
| `compression.type`                      | `lz4` 或 `snappy` | 提升吞吐降低带宽 |
| `max.in.flight.requests.per.connection` | 5+               | 并发请求数    |

并使用 **异步发送 + 回调机制**，避免阻塞主线程。

#### 四、消费者（Consumer）优化

- 使用 **多线程或多实例** 并行消费。
- `fetch.min.bytes`、`fetch.max.wait.ms` 调大，增加批量效率。
- `max.poll.records` 调高，提升单次拉取量。
- 如果消费端逻辑较重，建议：
    - 使用异步处理队列；
    - 消费与业务处理解耦（可配合异步任务队列）。

#### 五、消息模型与序列化

1. **消息大小**
    - Kafka 吞吐量与消息大小强相关。
    - 建议单消息 ≤ 1 KB。
    - 如果消息较大，可通过批量压缩、分块写入。

2. **序列化方式**
    - 使用高性能序列化框架：**Avro / Protobuf / Kryo**。
    - 避免 JSON。

3. **压缩算法**
    - 生产端启用压缩：`compression.type=lz4`。
    - 效果：压缩比 3~5x，吞吐提升 2~3 倍。

#### 六、网络与操作系统优化

- 网络参数：
  ```
  net.core.somaxconn=1024
  net.ipv4.tcp_tw_reuse=1
  net.ipv4.tcp_fin_timeout=15
  net.ipv4.tcp_max_syn_backlog=4096
  ```
- 使用独立网卡分离 **生产流量 / 消费流量 / 副本同步流量**。
- 启用 `jumbo frames` (MTU=9000) 以减少协议开销。

#### 七、监控与扩展

- 使用 **Prometheus + Grafana** 监控：
    - 吞吐量（bytes/sec）
    - ISR（同步副本）状态
    - 磁盘利用率
    - 网络流量
    - 消费延迟（consumer lag）
- 自动扩容策略：
    - 分区可动态扩容；
    - 支持 Broker 动态加入。

#### 八、性能实测基准（经验数据）

| 环境                        | 配置             | 吞吐量（TPS） |
|---------------------------|----------------|----------|
| 3 台 Broker（64C256G, NVMe） | 3 主题 × 30 分区   | ~300k    |
| 6 台 Broker                | 6 主题 × 60 分区   | ~650k    |
| 10 台 Broker               | 10 主题 × 100 分区 | \>1M     |

> 注意：实际性能取决于消息大小、ack 模式、磁盘和网络性能。

### 如何保证消息跨数据中心同步的一致性？

### 如何设计业务的幂等消费端？

### Kafka 如何作为日志收集系统？架构如何设计？

### 如何让 Kafka 支撑大规模延时敏感写入（低延迟高吞吐）？
