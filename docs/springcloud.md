# Spring Cloud

## 服务注册发现

服务注册中心本质上是为了解耦服务提供者和服务消费者。对于任何一个微服务，原则上都应存在或者支持多个提供者，这是由微服务的分布式属性决定的。
更进一步，为了支持弹性扩缩容特性，一个微服务的提供者的数量和分布往往是动态变化的，也是无法预先确定的。
因此，原本在单体应用阶段常用的静态负载均衡机制就不再适用了，需要引入额外的组件来管理微服务提供者的注册与发现，而这个组件就是服务注册中心。

### CAP 理论

- 一致性(Consistency)：所有节点在同一时间具有相同的数据
- 可用性(Availability)：保证每个请求不管成功或者失败都有响应
- 分区容忍性(Partition tolerance)：系统中任意信息的丢失或失败不会影响系统的继续运作

一般来说，分布式系统中，很难同时满足一致性、可用性和分区容忍。

- 如果需要保证一致性，那么会影响可用性的性能，因为要数据同步，否则请求结果会有差异，但是数据同步会消耗时间，期间可用性就会降低。
- 如果需要保证可用性，那么只要存在任意一个服务，就能正常接受请求，但是对与返回结果不能保证，因为在分布式系统中，数据同步需要时间。
- 如果需要同时保证一致性和可用性，那么分区容错就很难保证了，通常单机模式下才具备。所以分区容忍性是分布式系统的基本核心。

|                 | Nacos                    | Eureka | Consul   | Zookeeper | 
|:----------------|:-------------------------|:-------|:---------|:----------|
| 一致性协议           | CP/AP                    | AP     | CP       | CP        |
| 负载均衡策略          | Weight/Metadata/Selector | Ribbon | Fabio    |           |
| 雪崩保护            | 有                        | 有      | 无        | 无         |
| 访问协议            | HTTP/DNS                 | HTTP   | HTTP/DNS | TCP       |
| 监听支持            | 支持                       | 支持     | 支持       | 支持        |
| 多数据中心           | 支持                       | 支持     | 支持       | 不支持       |
| 跨注册中心同步         | 支持                       | 不支持    | 支持       | 不支持       |
| Spring Cloud 集成 | 支持                       | 支持     | 支持       | 支持        |
| Dubbo 集成        | 支持                       | 不支持    | 支持       | 支持        |
| K8S 集成          | 支持                       | 不支持    | 支持       | 不支持       |

### Alibaba Nacos

Nacos 支持基于 DNS 和基于 RPC 的服务发现。服务提供者使用原生SDK、OpenAPI、或一个独立的 Agent TODO 注册 Service 后，服务消费者可以使用 DNS TODO 或 HTTP API 查找和发现服务。

Nacos 提供对服务的实时的健康检查，阻止向不健康的主机或服务实例发送请求。Nacos 支持传输层 (PING 或 TCP)和应用层 (如 HTTP、MySQL、用户自定义）的健康检查。
对于复杂的云环境和网络拓扑环境中（如 VPC、边缘网络等）服务的健康检查，Nacos 提供了 agent 上报模式和服务端主动检测2种健康检查模式。Nacos 还提供了统一的健康检查仪表盘，帮助您根据健康状态管理服务的可用性及流量。

Nacos 提供动态配置服务可以让您以中心化、外部化和动态化的方式管理所有环境的应用配置和服务配置。动态配置消除了配置变更时重新部署应用和服务的需要，让配置管理变得更加高效和敏捷。
Nacos 提供了一个简洁易用的页面帮助您管理所有的服务和应用的配置。包括配置版本跟踪、金丝雀发布、一键回滚配置以及客户端配置更新状态跟踪在内的一系列开箱即用的配置管理特性，帮助您更安全地在生产环境中管理配置变更和降低配置变更带来的风险。

服务实例注册到注册中心时，会从注册中心获取服务列表，之后每隔一段时间会从注册中心获取服务列表。当某个实例状态发生变化时，会将消息推送到客户端。

Nacos 支持 AP 和 CP 模式的切换，默认为 AP 模式。

#### AP 模式 Distro 协议

Distro 协议的工作流程如下：

1. Nacos 启动时首先从其他远程节点同步全部数据。
2. Nacos 每个节点是平等的都可以处理写入请求，同时把新数据同步到其他节点。
3. 每个节点只负责部分数据，定时发送自己负责数据的校验值到其他节点来保持数据一致性。

#### CP 模式 Raft 协议

Raft 协议中有三种状态：跟随者、竞选者、领导者。

默认情况下每个节点都是跟随者，每个节点会随机生成一个选举的超时时间，在这个超时的时间范围类节点必须要等待。
超时时间过后，当前结点有跟随者变为竞选者，会给其他发出选举的投票通知，只要该竞选者有超过半数以上即可成为领导者。

Raft 故障重新选举：如果跟随者节点不能及时收到领导角色的消息，那么跟随者就会变为竞选者，给其他节点发送选举投票通知，有过半票数则称为领导者。

Raft 日志复制：所有的写请求都交给领导者，将请求操作写入日志，标记该状态为未提交状态。领导者会将日志以心跳形式发送给其他跟随者，只要满足过半的跟随者可以写入该数据，则直接通知其他节点同步该数据，这个过程称为日志复制。

### Netflix Eureka

Eureka Server 可以运行多个实例来构建集群，解决单点问题，但不同于 Nacos、ZooKeeper 的选举 Leader 的过程，Eureka Server 采用的是 Peer to Peer 对等通信。
这是一种去中心化的架构，在这种架构风格中，节点通过彼此互相注册来提高可用性，每个节点需要添加一个或多个有效的 serviceUrl 指向其他节点。每个节点都可被视为其他节点的副本。

在集群环境中如果某台 Eureka Server 宕机，Eureka Client 的请求会自动切换到新的 Eureka Server 节点上，当宕机的服务器重新恢复后，Eureka 会再次将其纳入到服务器集群管理之中。
当节点开始接受客户端请求时，所有的操作都会在节点间进行复制（Replicate To Peer）操作，将请求复制到该 Eureka Server 当前所知的其它所有节点中。

当一个新的 Eureka Server 节点启动后，会首先尝试从邻近节点获取所有注册列表信息，并完成初始化。Eureka Server 通过 getEurekaServiceUrls() 方法获取所有的节点，并且会通过心跳契约的方式定期更新。
只要有一个 Eureka Server 节点存在，就能保证注册服务可用（保证可用性），只不过查到的信息可能不是最新的（不保证强一致性）。

默认情况下，如果 Eureka Server 在一定时间内没有接收到某个服务实例的心跳（默认周期为30秒），Eureka Server 将会注销该实例（默认为90秒）。
当 Eureka Server 节点在短时间内丢失过多的心跳时，就认为客户端与注册中心出现了网络故障，那么这个节点就会进入自我保护模式，此时会出现以下几种情况：

1. Eureka 不再从注册表中移除因为长时间没有收到心跳而过期的服务。
2. Eureka 仍然能够接受新服务注册和查询请求，但是不会被同步到其它节点上（即保证当前节点依然可用）。
3. 当网络稳定时，当前实例新注册的信息会被同步到其它节点中。

因此，Eureka 可以很好的应对因网络故障导致部分节点失去联系的情况，而不会像 Zookeeper 那样使得整个注册服务瘫痪。

### HashiCorp Consul

Consul 是 HashiCorp 公司推出的开源工具，用于实现分布式系统的服务发现与配置。Consul 使用 Go 语言编写，因此具有天然可移植性（支持Linux、windows和Mac OS X）。

Consul 内置了服务注册与发现框架、分布一致性协议实现、健康检查、Key/Value 存储、多数据中心方案，不再需要依赖其他工具（比如 ZooKeeper 等），使用起来也较为简单。

Consul 保证了强一致性和分区容错性，且使用的是 Raft 协议。虽然保证了强一致性，但是可用性就相应下降了，因为 Consul 的 Raft 协议要求必须过半数的节点都写入成功才认为注册成功，所以服务注册的时间会稍长一些。如果 Leader 崩溃故障，在重新选举出新 Leader 之前 Consul 服务不可用。

### Apache Zookeeper

Apache Zookeeper 在设计时就紧遵 CP 原则，即任何时候对 Zookeeper 的访问请求能得到一致的数据结果，同时系统对网络分割具备容错性。但是 Zookeeper 采用 Zab 原子广播协议，所以不能保证每次服务请求都是可达的。

从 Zookeeper 的实际应用情况来看，在使用 Zookeeper 获取服务列表时，如果此时的 Zookeeper 集群中的 Leader 宕机了，该集群就要进行 Leader 的选举，又或者 Zookeeper 集群中半数以上服务器节点不可用，那么将无法处理该请求。

在大多数分布式环境中，尤其是涉及到数据存储的场景，数据一致性应该是首先被保证的，这也是 Zookeeper 设计紧遵 CP 原则的另一个原因。
但是对于服务发现来说，情况就不太一样了，针对同一个服务，即使注册中心的不同节点保存的服务提供者信息不尽相同，也并不会造成灾难性的后果。
对于服务消费者来说，能消费才是最重要的，消费者虽然拿到可能不正确的服务实例信息后尝试消费一下，也要胜过因为无法获取实例信息而不去消费，导致系统异常要好。

## 服务网关

网关的作用：

- 统一入口：未全部为服务提供一个唯一的入口，网关起到外部和内部隔离的作用，保障了后台服务的安全性。
- 鉴权校验：识别每个请求的权限，拒绝不符合要求的请求。
- 动态路由：动态的将请求路由到不同的后端集群中。
- 减少耦合：服务端服务可以独立发展，通过网关层来做映射。
- 洞察监控：在边缘跟踪有意义的数据和统计数据，以便为我们提供准确的生产视图。
- 请求限流：通过对并发请求进行限速来保护系统。

### Spring Cloud Gateway

TODO

### Netflix Zuul

TODO

## 远程调用

### Spring Cloud OpenFeign

Spring Cloud OpenFeign 最核心的作用是为 HTTP 形式的 Rest API 提供了非常简洁高效的 RPC 调用方式。 如果说 Spring Cloud 其他成员解决的是系统级别的可用性，扩展性问题， 那么 Spring Cloud OpenFeign 解决的则是与开发人员利益最为紧密的开发效率问题。

Spring Cloud OpenFeign 的核心工作原理简单的总结为：

1. 通过扫描`@EnableFeignClients`注解指定路径下的的`@FeignClient`修饰的接口。
2. 解析到`@FeignClient`修饰的接口后，通过扩展 BeanDefinition 的注册逻辑，注册`FeignClientFactoryBean`到 Spring 容器中。
3. 调用`FeignClientFactoryBean`对象的`getObject()`方法，会返回一个基于 JDK 动态代理生成的代理对象。
4. 调用`@FeignClient`修饰的接口实例的方法时，会被转发到`InvocationHandler`对，由该对象完成一系列操作。

### Netflix Feign

TODO

## 负载均衡

### Spring Cloud LoadBalancer

TODO

### Netflix Ribbon

Ribbon 是 Netflix 的开源项目，主要功能是提供客户端的负载均衡算法和服务调用。
Ribbon 默认提供了几种负载均衡算法，例如：轮询、随机、权重等。开发者也可以实现自定义的负载均衡算法。

Ribbon 的核心组件：

- Rule：用于从服务列表中选取服务的逻辑组件。

- Ping：在后台运行的确保服务可用性的组件。
- ServerList：服务列表，支持静态和动态。

#### Rule

- RoundRobinRule：轮询调度规则，默认规则，不断的轮询选择服务，将请求平摊到的每个服务。
- RandomRule：随机规则，将流量随机分散在各个服务上。
- BestAvailableRule：忽略不可用的服务，然后尽量找并发比较低的服务器来请求
- WeightedResponseTimeRule：每个服务器可以有权重，权重越高优先访问，如果某个服务器响应时间比较长，那么权重就会降低，减少访问
- AvailabilityFilteringRule：可用过滤策略，先过滤出故障的或并发请求大于阈值的一部分服务实例，然后再以线性轮询的方式从过滤后的实例清单中选出一个
- ZoneAvoidanceRule：从最佳区域实例集合中选择一个最优性能的服务
- RetryRule：服务请求失败时，重新选择一个服务

#### ServerList

Ribbon 使用`ServerList`接口抽象服务实例列表，Ribbon 获取服务实例有两种方法：配置文件和注册中心。

##### 从配置文件获取 ServerList

在没有使用注册中心的情况下，Ribbon 可以通过`ConfigurationBasedServerList`类获取配置文件列举服务实例的地址，例如：

```properties
{service-name}.ribbon.listOfServers=http://localhost:8001,http://localhost:8002
```

##### 从注册中心获取 ServerList

服务实例上下线在微服务系统中是一个非常常见的场景，Ribbon 使用`DiscoveryEnabledNIWSServerList`类维护注册中心的服务。

当 Ribbon 从注册中心获取了服务实例列表之后，会动态更新服务实例列表，更新的方式有两种，一种是通过定时任务定时拉取服务实例列表，另一种是通过注册中心服务事件通知的方式。

## 熔断降级

### Spring Cloud Circuit Breaker

#### Resilience4J

TODO

#### Spring Retry

TODO

### Netflix Hystrix

Hystrix 是 Netflix 的开源项目，是断路器的一种实现，主要功能是提供客户端的服务隔离、熔断、降级，用于高微服务架构的可用性，是防止服务出现雪崩的利器。

在通过网络依赖服务出现高延迟或者失败时，为系统提供保护和控制 可以进行快速失败，缩短延迟等待时间和快速恢复。
当异常的依赖回复正常后，失败的请求所占用的线程会被快速清理，不需要额外等待提供失败回退（Fallback）和相对优雅的服务降级机制提供有效的服务容错监控、报警和运维控制手段。

#### 隔离策略

Hystrix 提供了两种模式隔离策略：信号量隔离模式、线程池隔离模式。（默认使用线程池模式）

|        | 线程池隔离                      | 信号量隔离                                            |
|:-------|:---------------------------|:-------------------------------------------------|
| 是否支持超时 | 支持，超时直接返回                  | 不支持，如果阻塞，只能通过协议超时                                |
| 是否支持熔断 | 支持，当线程池到达maxSize后，再请求会触发熔断 | 支持，当信号量达到maxConcurrentRequests后，再请求会触发熔断         |
| 是否支持异步 | 支持                         | 不支持                                              |
| 隔离原理   | 每个服务单独用线程池                 | 通过信号量的计数器                                        |
| 资源消耗情况 | 大，需要频繁进行线程上下文切换            | 小，计数器                                            |

如果要使用信号量模式，需要配置参数`execution.isolation.strategy=ExecutionIsolationStrategy.SEMAPHORE`。