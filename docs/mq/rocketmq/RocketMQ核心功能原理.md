# 初探门径

## 整体介绍

- 一款分布式、队列模型的消息中间件

- 支持集群模型、负载均衡、**水平扩展能力**
- 亿级消息堆积能力
- 采用零拷贝、顺序写盘、随机读
- 丰富的API
- 底层通信框架采用 Netty NIO 框架
- NameServer代替Zookeeper
- 强调集群无单点、可扩展，任意一点高可用，水平可扩展
- 消息失败重试机制、消息可查询
- 开源社区活跃、成熟度高（经过双十一考验）



## 概念模型

- Producer：消息生产者，负责生产消息，一般由业务系统负责产生消息
- Consumer：消息消费者，负责消费消息，一般由后台系统负责**异步消费**
  - Push Consumer：向Consumer对象注册监听
  - Pull Consumer：主动请求Broker拉取消息
- Producer Group：生产者集合，一般用于发送一类消息
  - 多个生产者并行发送消息，提高效率
  - 事务消息，broker会回调确认本地事务状态，如果一个生产者挂了，可以访问另外一个

- Consumer Group：消费者集合，一般用于接收一类消息进行消费
- Broker：MQ消息服务，中转角色，用于消息存储与生产消费转发



## 源码结构

- rocketmq-broker 主要的业务逻辑，消息收发，主从同步，pagecache
- rocketmq-client 客户端接口，如生产者和消费者
- rocketmq-example 示例
- rocketmq-common 公用数据结构
- rocketmq-distribution 编译模块，编译输出
- rocketmq-filter 进行Broker过滤不感兴趣的消息传输，减小带宽压力
- rocketmq-logappender 日志相关
- rocketmq-loggin 日志相关
- rocketmq-namesrv Namesrv服务，用于服务协调
- rocketmq-openmessaging 对外提供服务
- rocketmq-remoting 远程调用接口，封装Netty底层通信
- rocketmq-srvutil 提供一些公用的工具方法，如解析命令行参数
- rocketmq-store 消息存储
- rocketmq-test
- rocketmq-tools 管理工具，如mqadmin工具



# 主要组件

## NameServer

NameServer 集群，Topic 的路由注册中心，为客户端根据 Topic 提供路由服务，从而引导客户端向 Broker 发送消息。**NameServer 之间的节点不通信**。路由信息在 NameServer 集群中数据一致性采取的最终一致性。



## Broker

消息存储服务器，分为两种角色：Master 与 Slave，在 RocketMQ 中，**主服务承担读写操作，从服务器作为一个备份**，当主服务器存在压力时，从服务器可以承担读服务（消息消费）。所有 Broker，包含 Slave 服务器每隔 **30s** 会向 NameServer 发送心跳包，心跳包中会包含存在在 Broker 上所有的 Topic 的路由信息。

- 在 RocketMQ 4.5.0 版本后引入了多副本机制，即一个复制组（m-s）可以演变为基于 Raft 协议的复制组，复制组内部使用 Raft 协议保证 Broker 节点数据的强一致性，该部署架构在金融行业用的比较多。
- NameServer 是在内存中存储 Topic 的路由信息，持久化 Topic 路由信息的地方是在 Broker 中，即`${ROCKETMQ_HOME}/store/config/topics.json`



## Client

消息客户端，包括 Producer（消息发送者）和 Consumer（消息消费者）。客户端在同一时间只会连接一台 NameServer，只有在连接出现异常时才会向尝试连接另外一台。客户端每隔 **30s** 向 NameServer 发起 Topic 的路由信息查询。



# 消息生产者

## 核心参数

- `producerGroup` 发送者所属组，开源版本的 RocketMQ，发送者所属组主要的用途是事务消息。一个应用中唯一
- `createTopicKey` 创建topic（生产禁止）
- `defaultTopicQueueNums = 4` 通过生产者创建 Topic 时默认的队列数量
- `sendMsgTimeout = 3000` 消息发送默认超时时间，单位为毫秒。RocketMQ 4.3.0 版本进行了优化，设置的超时时间为总的超时时间，即如果超时时间设置 3s，重试次数设置为 10 次，可能不会重试 10 次，例如在重试到第 5 次的时候，已经超过 3s 了，试图尝试第 6 次重试时会退出，抛出超时异常，停止重试
- `compressMsgBodyOverHowmuch` 压缩的阈值，默认为 4k，即当消息的消息体超过 4k，则会使用 zip 对消息体进行压缩，会增加 Broker 端的 CPU 消耗，但能提高网络方面的开销
- `retryTimesWhenSendFailed` 同步消息发送重试次数。RocketMQ 客户端内部在消息发送失败时默认会重试 2 次。该参数与 sendMsgTimeout 会联合起来生效
- `retryTimesWhenSendAsyncFailed` 异步消息发送重试次数，默认为 2，即重试 2 次，通常情况下有三次机会
- `retryAnotherBrokerWhenNotStoreOK` 该参数的本意是如果客户端收到的结果不是 SEND_OK，应该是不问源由的继续向另外一个 Broker 重试，但根据代码分析，目前这个参数并不能按预期运作，应该是一个 Bug
- `maxMessageSize` 允许发送的最大消息体，默认为 4M，服务端（Broker）也有 maxMessageSize 这个参数的设置，故客户端的设置不能超过服务端的配置，最佳实践为客户端的配置小于服务端的配置
- `sendLatencyFaultEnable` 是否开启失败延迟规避机制。RocketMQ 客户端内部在重试时会规避上一次发送失败的 Broker，如果开启延迟失败规避，则在未来的某一段时间内不向该 Broker 发送消息。默认为 false，不开启
- `notAvailableDuration` 不可用的延迟数组，默认值为 {0L, 0L, 30000L, 60000L, 120000L, 180000L, 600000L}，即每次触发 Broker 的延迟时间是一个阶梯的，会根据每次消息发送的延迟时间来选择在未来多久内不向该 Broker 发送消息
- `latencyMax` 设置消息发送的最大延迟级别，默认值为 {50L, 100L, 550L, 1000L, 2000L, 3000L, 15000L}，个数与 notAvailableDuration 对应



## 消息发送方式

RocketMQ 支持同步、异步、Oneway 三种发送方式。

- 同步：客户端发起一次消息发送后会同步等待服务器的响应结果。
- 异步：客户端发起一下消息发起请求后不等待服务器响应结果而是立即返回，这样不会阻塞客户端子线程，当客户端收到服务端（Broker）的响应结果后会自动调用回调函数。
  - 每一个消息发送者实例（DefaultMQProducer）内部会创建一个异步消息发送线程池，默认线程数量为 CPU 核数，线程池内部持有一个有界队列，默认长度为 5W，并且会控制异步调用的最大并发度，默认为 65536，其可以通过参数 clientAsyncSemaphoreValue 来配置。
  - 客户端使用线程池将消息发送到服务端，服务端处理完成后，返回结果并根据是否发生异常调用 SendCallback 回调函数。
- Oneway：客户端发起消息发送请求后并不会等待服务器的响应结果，也不会调用回调函数，即不关心消息的最终发送结果。
  - Oneway 方式通常用于发送一些不太重要的消息，例如操作日志，偶然出现消息丢失对业务无影响。



## 消息发送状态

SendStatus

发送成功

- SEND_OK

消息发送成功，但有问题，需要考虑补偿重发消息

- FLUSH_DISK_TIMEOUT 刷盘超时
- FLUSH_SLAVE_TIMEOUT 主从同步超时
- SLAVE_NOT_AVAILABLE 从节点不可用



## 延迟消息

**消息发送到Broker后**，要特定时间才会被Consumer消费。

目前只支持固定精度的定时消息。

MessageStoreConfig配置精度（store）；ScheduleMessageService定时任务处理

在Message上设置，setDelayTimeLevel，默认延迟等级 : 1s 5s 10s 30s 1m 2m 3m 4m 5m 6m 7m 8m 9m 10m 20m 30m 1h 2h， 传入1代表1s, 2代表5s, 以此类推



## 顺序消费

- 生产者通过实现MessageQueueSelector方法，指定消息到指定的队列，**实现多条消息的顺序消费**
  - 消息发送时如果使用了 MessageQueueSelector，那消息发送的重试机制将失效，即 RocketMQ 客户端并不会重试，消息发送的高可用需要由业务方来保证，典型的就是消息发送失败后存在数据库中，然后定时调度，最终将消息发送到 MQ

- 消费者需要注册MessageListenerOrderly



## 消息标识

- RocketMQ 提供了丰富的消息查询机制，例如使用消息偏移量、消息全局唯一 MsgId（MQ生成的全局唯一ID，用于唯一标识一个消息）、消息 Key（业务相关，可以通过key批量查询）

- RocketMQ 在消息发送的时候，可以为一条消息设置索引建key，可以通过该索引 Key 进行查询（唯一或多条）消息；**如果要为消息指定多个 Key，用空格分开即可**（包含多条查询的业务key，以及包含业务全局唯一ID）

- **MsgId**是消息发送者在消息发送时会首先**在客户端生成**，全局唯一，在 RocketMQ 中该 ID 还有另外的一个叫法——uniqId。其组成说明如下：

  - 客户端发送 IP，支持 IPV4 和 IPV6
  - 进程 PID（2 字节）
  - 类加载器的 hashcode（4 字节）
  - 当前系统时间戳与启动时间戳的差值（4 字节）
  - 自增序列（2 字节）

- **OffsetMsgId**消息所在 **Broker** 的物理偏移量，即在 commitlog 文件中的偏移量，其组成如下两部分组成：

  - Broker 的 IP 与端口号
  - commitlog 中的物理偏移量

  可以根据 offsetMsgId 即可以定位到具体的消息，无需知道该消息的 Topic 等其他一切信息。

  ==MsgId和OffsetMsgId这两个消息 ID 有时候在排查问题的时候，特别是项目能提供 msgID，但是在消息集群中无法查询到时可以通过解码这两个消息 ID，从而得知消息发送者的 IP 或 消息存储 Broker 的 IP。其中 MsgId 可以通过 MessageClientIDSetter 的 getIPStrFromID 方法获取 IP，而 OffsetMsgId 可以通过 MessageDecoder 的 decodeMessageId 方法解码。==

- tag标签

  - 可以为 Topic 设置 Tag（标签），这样消费端可以对 Topic 中的消息基于 Tag 进行过滤，即选择性的对 Topic 中的消息进行处理
  - 不符合订阅的 Tag，其消费状态显示为 CONSUMED_BUT_FILTERED（消费但被过滤掉）



## 事务消息

两阶段提交＋定时轮循（Broker）

事务消息能保证**本地事务**与**消息发送**这两个操作的强一致性。



## 故障规避机制

RocketMQ Topic 路由注册中心 NameServer 采用的是最终一致性模型，而且客户端是定时向 NameServer 更新 Topic 的路由信息，即客户端（Producer、Consumer）是无法实时感知 Broker 宕机的，这样消息发送者会继续向已宕机的 Broker 发送消息，造成消息发送异常。

RocketMQ 是如何保证消息发送的高可用性呢？在消息重试的时候，会尽量规避上一次发送的 Broker，当消息发往 broker-a q1 队列时返回发送失败，那重试的时候，会先排除 broker-a 中所有队列，即这次会选择 broker-b q1 队列，增大消息发送的成功率。上述机制自动生效。

RocketMQ 提供了两种规避策略，该参数由 **sendLatencyFaultEnable** 控制，用户可干预，表示是否开启延迟规避机制，默认为不开启。

- sendLatencyFaultEnable 设置为 false：默认值，不开启，**延迟规避策略只在重试时生效**，例如在一次消息发送过程中如果遇到消息发送失败，规避 broekr-a，但是在下一次消息发送时，即再次调用 DefaultMQProducer 的 send 方法发送消息时，还是会选择 broker-a 的消息进行发送，只要继续发送失败后，重试时再次规避 broker-a。
- sendLatencyFaultEnable 设置为 true：开启延迟规避机制，一旦消息发送失败会将 broker-a “悲观”地认为在接下来的一段时间内该 Broker 不可用，在为未来某一段时间内所有的客户端不会向该 Broker 发送消息。这个延迟时间就是通过 notAvailableDuration、latencyMax 共同计算的，首先先计算本次消息发送失败所耗的时延，然后对应 latencyMax 中哪个区间，即计算在 latencyMax 的下标，然后返回 notAvailableDuration 同一个下标对应的延迟值。

如果所有的 Broker 都触发了故障规避，并且 Broker 只是那一瞬间压力大，那岂不是明明存在可用的 Broker，但经过规避，反倒是没有 Broker 可用，那岂不是更糟糕了？针对这个问题，会退化到队列轮循机制，即不考虑故障规避这个因素，按自然顺序进行选择进行兜底。

RocketMQ Broker 的繁忙基本都是瞬时的，而且通常与系统 PageCache 内核的管理相关，很快就能恢复，故不建议开启延迟机制。因为一旦开启延迟机制，例如 5 分钟内不会向一个 Broker 发送消息，这样会导致消息在其他 Broker 激增，从而会导致部分消费端无法消费到消息，增大其他消费者的处理压力，导致整体消费性能的下降。



# 消息消费者

## 核心参数

- consumerGroup

  消费组的名称，在 RocketMQ 中，对于消费者来说，一个消费组就是一个独立的隔离单位，例如多个消费组订阅同一个主题，其消息进度（消息处理的进展）是相互独立的，两者不会有任何的干扰。

- messageModel

  消息组消息消费模式，在 RocketMQ 中支持集群模式、广播模式。集群模式是一个消费组内多个消费者共同消费一个 Topic 中的消息，即一条消息只会被集群内的某一个消费者处理；而广播模式是指一个消费组内的每一个消费者负责 Topic 中的所有消息。

- consumeTimestamp

  指定从什么时间戳开始消费，其格式为 yyyyMMddHHmmss，默认值为 30 分钟之前，该参数只在 consumeFromWhere 为 CONSUME_FROM_TIMESTAMP 时生效。

- consumeFromWhere

  一个消费者**初次启动**时（即消费进度管理器中无法查询到该消费组的进度）从哪个位置开始消费的策略，可选值如下所示：

  - CONSUME_FROM_LAST_OFFSET：**首次**从队列最后位置开始消费。**后续**从上一次消费的位置开始消费。
  - CONSUME_FROM_FIRST_OFFSET：首次从队列开始位置开始消费。
  - CONSUME_FROM_TIMESTAMP：从指定的时间戳开始消费，这里的实现思路是从 Broker 服务器寻找消息的存储时间小于或等于指定时间戳中最大的消息偏移量的消息，从这条消息开始消费。

- allocateMessageQueueStrategy

  消息队列负载算法。主要解决的问题是消息消费队列在各个消费者之间的负载均衡策略，例如一个 Topic 有８个队列，一个消费组中有３个消费者，那这三个消费者各自去消费哪些队列。

  RocketMQ 默认提供了如下负载均衡算法：

  - AllocateMessageQueueAveragely：平均连续分配算法。
  - AllocateMessageQueueAveragelyByCircle：平均轮流分配算法。
  - AllocateMachineRoomNearby：机房内优先就近分配。
  - AllocateMessageQueueByConfig：手动指定，这个通常需要配合配置中心，在消费者启动时，首先先创建 AllocateMessageQueueByConfig 对象，然后根据配置中心的配置，再根据当前的队列信息，进行分配，即该方法不具备队列的自动负载，在 Broker 端进行队列扩容时，无法自动感知，需要手动变更配置。
  - AllocateMessageQueueByMachineRoom：消费指定机房中的队列，该分配算法首先需要调用该策略的 `setConsumeridcs(Set<String> consumerIdCs)` 方法，用于设置需要消费的机房，将筛选出来的消息按平均连续分配算法进行队列负载。
  - AllocateMessageQueueConsistentHash 一致性 Hash 算法。

- subscribe 指定订阅主题，和订阅表达式，如tag1 || tag2 || tag3，或*（全部订阅）

  - 集群模式下，一个消费者组的不同消费者如果订阅不同的tag，可能会丢失消息。消费者消费的队列恰好收到了过滤的消息，就会丢失（被过滤）。

- offsetStore

  消息进度存储管理器，该属性为私有属性，不能通过 API 进行修改，该参数主要是根据消费模式在内部自动创建，RocketMQ 在广播消息、集群消费两种模式下消息消费进度的存储策略会有所不同。

  - 集群模式：RocketMQ 会将消息消费进度存储在 Broker 服务器，存储路径为 `${ROCKET_HOME}/store/config/ consumerOffset.json` 文件中。

    ```shell
    # 消息消费进度，首先使用 topic@consumerGroup 为键，其值是一个 Map，键为 Topic 的队列序列，值为当前的消息消费位点。
    {
            "offsetTable":{
                    "my_topic@c_g_push":{0:0,1:2,2:0,3:0},
                    "%RETRY%c_g_push@c_g_push":{0:0}
            }
    }
    ```

  - 广播模式：RocketMQ 会将消息消费进度存储在消费端所在的机器上，存储路径为 `${user.home}/.rocketmq_offsets` 中。

- consumeThreadMin

  消费者最小线程数量，默认为 20。在 RocketMQ 消费者中，会为每一个消费者创建一个独立的线程池。

  - <font color=red>当配合MessageListenerConcurrently使用时，一个消费者内部的多个消费线程并行消费队列中的任务，注意此种消费方式无法保证顺序消费</font>。

- consumeThreadMax

  消费者最大线程数量，在当前的 RocketMQ 版本中，该参数通常与 consumeThreadMin 保持一致，大于没有意义，因为 RocketMQ 创建的线程池内部创建的队列为一个无界队列。

- consumeConcurrentlyMaxSpan

  并发消息消费时处理队列中最大偏移量与最小偏移量的差值的阈值，如差值超过该值，触发消费端限流。限流的具体做法是不再向 Broker 拉取该消息队列中的消息，默认值为 2000。

- pullThresholdForQueue

  允许消费端单队列积压的消息数量，如果处理队列中超过该值，会触发消息消费端的限流。默认值为 1000，不建议修改该值。

- pullThresholdSizeForQueue

  允许消费端单队列中积压的消息体大小，默认为 100MB。

- pullThresholdForTopic

  按 Topic 级别进行消息数量限流，默认不开启，为 -1，如果设置该值，会使用该值除以分配给当前消费者的队列数，得到每个消息消费队列的消息阈值，从而改变 pullThresholdForQueue。

- pullThresholdSizeForTopic

  按 Topic 级别进行消息消息体大小进行限流，默认不开启，其最终通过改变 pullThresholdSizeForQueue 达到限流效果。

- long pullInterval = 0

  消息拉取的间隔，默认 0 表示，消息客户端在拉取一批消息提交到线程池后立即向服务端拉取下一批，PUSH 模式不建议修改该值。

- int pullBatchSize = 32

  一次消息拉取请求最多从 Broker 返回的消息条数，默认为 32。

- consumeMessageBatchMaxSize 

  消息消费一次最大消费的消息条数。默认是1。

- maxReconsumeTimes 

  消息消费重试次数，并发消费模式下默认重试 16 次后进入到死信队列，如果是顺序消费，重试次数为 Integer.MAX_VALUE

- suspendCurrentQueueTimeMillis

  消费模式为顺序消费时设置每一次重试的间隔时间，提高重试成功率。

- consumeTimeout

  消息消费超时时间，默认为 15 分钟。



## 消费模式

Clustering 集群模式（默认）

- 消费端的负载均衡；groupName用于把多个consumer组织到一起



Broadcasting 广播模式

- 同一个consumerGroup里的所有consumer可以消费订阅topic内的全部消息；即一条消息会被每一个consumer消费



## 偏移量 Offset

消息消费进度的核心；Offset指某个topic下的一条消息在某个MessageQueue里的位置（和Consumer解耦）。

Offset存储实现分为远程文件类型和本地文件类型

- RemoteBrokerOffsetStore 集群模式，采用远程文件类型；由Broker控制Offset的值。
- LocalFileOffsetStore 广播模式，记录在本地Consumer端。由于每个Consumer都会收到消息且消费。各个Consumer之间没有任何干扰，独立线程消费。

<font color=red>Push模式不需要关注Offset的存储位置，由RocketMQ实现；Pull模式需要自己手动维护Offset</font>。



## PushConsumer长轮询模式分析

主流消息获取模式

- Push推送
  - Broker主动推送消息给Consumer，但会增加Broker负担，影响Broker性能；Broker不知道Consumer的消息处理能力，不知道给Consumer多少条消息；
- Pull拉取
  - 注意时效性，以及拉取的频率



**RocketMQ出于提高Broker性能的哲学，多生产者，多消费者，Push实现为长轮询，Broker的工作尽量交由客户端去做。（broker.longpolling；Consumer到Broker取消息，当有消息时，Broker立即返回；否则，Broker线程阻塞channel，5S*3）**

- 经过队列负载机制后，会分配给当前消费者一些队列，注意一个消费组可以订阅多个主题。
- 轮流从 pullRequestQueue 中取出一个 PullRequest 对象，根据该对象中的拉取偏移量向 Broker 发起拉取请求，默认拉取 32 条，可通过 pullBatchSize 参数进行改变，该方法不仅会返回消息列表，还会返更改 PullRequest 对象中的下一次拉取的偏移量。
- 接收到 Broker 返回的消息后，会首先放入 ProccessQueue（处理队列），该队列的内部结构为 TreeMap，key 存放的是消息在消息消费队列（consumequeue）中的偏移量，而 value 为具体的消息对象。
- 然后将拉取到的消息提交到消费组内部的线程池，并立即返回，并将 PullRequest 对象放入到 pullRequestQueue 中，然后取出下一个 PullRequest 对象继续重复消息拉取的流程，从这里可以看出，**消息拉取与消息消费是不同的线程**。
- 消息消费组线程池处理完一条消息后，会将消息从 ProccessQueue 中删除，然后会向 Broker 汇报消息消费进度，以便下次重启时能从上一次消费的位置开始消费。

- 消息消费进度提交，不保证重复消费（向 Broker 汇报消息消费进度时是取 ProceeQueue 中最小的偏移量为消息消费进度）



## PullConsumer使用

- 获取MessageQueue并遍历
- 手动维护Offset（磁盘/数据库）
- 根据不同的消息状态做不同的处理



# 核心原理

## 消息的存储结构

Commit log 存储真实数据

- 被Broker的所有Consumer Queue共享
- 顺序写，随机读（利用pageCache和index提高性能）

Consumer Queue 存储消息在 Commit log 中的位置信息

- 每一个Topic的Queue对应一个Consumer Queue



## 同步刷盘与异步刷盘

消息存储：内存+磁盘

异步刷盘

- Broker写到内存pagecache中，立即返回写成功给producer
- 吞吐量高
- 内存积压到一定程度后，触发一次顺序写

同步刷盘

- Broker写到内存pagecache中，落盘后，再返回写成功给producer



## 同步复制与异步复制

同一组Broker的Master和Slave相互配合

同步复制

- Producer->BrokerMaster->BrokerSlave->BrokerMaster->Producer

异步复制

- 异步线程复制

实际可采用**同步双写（主从同步复制），异步刷盘**。保证Master和Slave的内存中都有数据，只要不是一起宕机，就可以保证数据的高可用，同时又兼顾了性能。



## 高可用机制

Master/Slave高可用

- 通过BrokerId区分Master/Slave
- Master读写；Slave只读
- 当Master繁忙或不可用时，可以自动切换到Slave读取消息



## NameServer协调者

NameServer是整个集群的状态服务器。

- 相互独立部署；客户端会向所有NameServer上报状态（热备份）
- 简单，易维护



# 集群架构

## 单点模式



## 主从模式

- 保障消息的即时性和可靠性
- 投递一条消息，关闭主节点，<font color=red>从节点可以继续提供消费者数据消费，但不能接收消息</font>
- 主节点上线后进行消费进度offset同步



### 主从同步解析

- Master / Slave 主从同步
- 同步信息包括：数据内容 + 元数据信息
  - 元数据信息同步
    - Broker识别角色，为Slave则启动定时同步任务，同步配置项（topicConfig、consumerOffset...）
    - BrokerController（关注定时任务，60S），基于netty（broker）
    - ==Broker没有配置主从的地址，只配置了namesrv，slave同步时如何知道master在哪里？==
      - BrokerName+BrokerId 界定是否是一组Broker，以及主节点或从节点；Broker启动后会定时向NameSrv注册，同时定时上报路由信息。之后，从NameSrv上可以获取这组BrokerName的Master
      - Slave配置中配置了 **haMasterAddress**
  - 数据内容消息同步
    - 实时同步commitlog
    - HAService、HAConnection、WaitNotifyObject，基于socket（store）



## 双主模式



## 双主双从、多主多从模式

