# AMQP

advanced message queuing protocol 高级消息队列协议。具有现代特征的二进制协议。是一个提供统一消息服务的应用层标准高级消息队列协议，是应用层协议的一个开放标准，为面向消息的中间件设计。

## 核心概念

- Server

  又称为Broker，接收客户端连接，实现AMQP实体服务

- Connection

  连接，应用程序与Broker的网络连接

- Channel

  网络信道，几乎所有的操作都在Channel中进行，Channel是进行消息读写的通道。客户端可建立多个Channel，每个Channel代表一个会话任务

- Message

  消息，服务器和应用程序之间传送的数据，由Properties和Body组成。Properties可以对消息进行修饰，比如消息的优先级，延迟等高级特性；Body是消息体内容

- Virtual host

  虚拟地址，用于进行**逻辑隔离**，最上层的消息路由。一个Virtual host可以有若干个Exchange和Queue，同一个Virtual host里不能有相同名称的Exchange或Queue

- Exchange

  交换机，接收消息，根据路由键转发消息到绑定的队列；可以绑定多个Queue，转发到哪个Queue由binding中的Routing key决定

- Binding

  Exchange和Queue之间的虚拟连接，binding中包含routing key

- Routing key

  一个路由规则，虚拟机可用它来确定如何路由一个特定消息

- Queue

  又称为Message Queue，消息队列，保存消息并将它们转发给消费者



# RabbitMQ

- 生产者关心Exchange和RoutingKey
- 消费者关心Queue
- Exchange和Queue通过RoutingKey关联起来
- Exchange是Topic模式，RoutingKey是通配符形式，Queue关联通配符形式的RoutingKey，即可以接收一类满足通配符条件的生产者发送的消息
- Exchange是Fanout模式，Exchange不关联RoutingKey，会广播发送到所有关联的Queue
- Exchange是Direct模式，一个队列可以绑定多个RoutingKey（用逗号分隔）；多个队列也可以绑定一个RoutingKey（类似Fanout模式）



## 核心实现

### Exchange 交换机

#### Name

交换机名称

#### Type

交换机类型

- Direct

  所有发送到Direct Exchange的消息被转发到RouteKey中指定的Queue。**RouteKey必须完全匹配才会被队列接收**，否则该消息会被抛弃。

  <font color=red>注意</font>，Direct模式可以使用RabbitMQ自带的Default Exchange，不需要将Exchange同队列进行任何绑定操作，消息传递时，RouteKey必须完全匹配Queue name才会被队列接收。

- Topic

  所有发送到Topic Exchange的消息被转发到所有关心RouteKey中指定的Topic的Queue。Exchange将RouteKey和某个Topic进行模糊匹配，此时队列需要绑定一个Topic。

  <font color=red>注意</font>，可以使用通配符进行模糊匹配：

  - `#` 匹配一个或多个词（**单词间用 . 分隔**）
  - `*` 匹配不多不少一个词

  如：

  `log.#` 能够匹配到 log.info.oa

  `log.*` 只会匹配到 log.erro

- Fanout

  不处理路由键，只需要简单的将队列绑定到交换机上，发送到交换机上的消息都会被转发到与该交换机绑定的所有队列上。Fanout交换机转发消息是最快的

- Headers

  不常用，通过消息头路由。

#### Durability

是否持久化，true为持久化

#### AutoDelete

当最后一个绑定到Exchange上的队列删除后，自动删除这个Exchange

#### Internal

当前Exchange是否用于RabbitMQ内部使用，默认为false

#### Arguments

扩展参数，用于扩展AMQP协议定制化使用



### Binding 绑定

Exchange和Queue之间的连接关系。Binding中可以包含RoutingKey或者参数。



### Queue 消息队列

实际存储消息数据。

- Durability：是否持久化。Durable 是，Transient 否
- Auto delete：yes 表示最后一个监听被移除之后，该Queue会自动被删除



### Message 消息

服务器和应用程序之间传送的数据。本质就是一段数据，由Properties和Payload（Body）组成。

- delivery mode
- headers 自定义属性
- content_type、content_encoding、priority
- correlation_id、reply_to、expiration、message_id
- timestamp、type、user_id、app_id、cluster_id



### Virtual host 虚拟主机

虚拟地址，用于进行逻辑隔离，最上层的消息路由。一个Virtual host里可以有若干个Exchange和Queue，同一个Virtual host不能有相同名称的Exchange或Queue



## 高级特性

### 消息如何保证100%投递成功

#### 可靠性投递

- 保证消息成功发出
- 保证MQ节点成功接收
- 发送端收到MQ节点确认应答（有状态）
  - Confirm 确认消息模式
- 完善的消息补偿机制

#### 消息补偿解决方案

- 生产端消息落库，对消息状态进行标记；没有成功的，轮询发送
  - 生产端需要写业务库和消息库（**高并发场景不适合**）
  - 【分布式定时任务】抓取未返回的消息，重新发送，设置重试次数阈值
- 消息的延迟投递，做二次确认
  - 生产端只写业务库，然后向MQ发送消息（非核心业务，对时间要求不大，如5分钟后发短息，影响不大），主业务链路完成（**少写一次数据库**）；然后再向MQ发送一条延迟投递消息，如延迟5分钟（【消息回调服务】消费）
  - 消费端正常收到消息后，发送确认消息给MQ，【消息回调服务】消费确认消息，然后落库；接着，从MQ消费到生产端发来的延迟投递消息，查询消费端消费成功（已落库），结束；否则，【消息回调服务】调用生产端重新发送消息



### 消费端实现幂等性消费

解决方案

- 唯一ID+指纹码（内部、外部业务规则）机制，利用数据库主键去重
  - 先用唯一ID+指纹码查询，查到了，失败；查不到，则插入；并发情况下，如果查询都成功，但因为是主键原因，只能有一个成功。（有写入瓶颈；可分库分表，分散压力）
- 利用redis的原子性
  - 需要考虑，如果redis成功，是否写库，如果写库的话，两个事务的问题；如果不写库，如何保证redis数据不丢失的问题



### Confirm 确认消息

消息确认，指生产端投递消息后，如果Broker收到消息，则会给生产者一个应答。生产者接收应答，用来确定这条消息是否正常发送到Broker，这种方式也是消息的可靠性投递的核心保障。

- channel.confirmSelect() 指定消息的确认模式
- channel.addConfirmListener(ConfirmListener) 添加确认监听
- <font color=red>生产端发送消息，收到Broker的确认信息通知，通过**CorrelationData**关联，此参数Id生产端提供并保证唯一性</font>



### Return 不可达消息

Return Listener 处理一些不可路由的消息。如果在发送消息的时候，当前的Exchange不存在，或这指定的Routingkey路由不到队列，Return Listener监听不可达的消息。

- 生产端发送消息，mandatory 参数 如果为true，则监听器接收到路由不可达的消息，做后续处理；如果为false，那么Broker会自动删除该消息
- channel.addReturnListener(ReturnListener) 添加不可达消息监听



### 消费端限流

假设RabbitMQ积压大量消息，打开一个消费端，巨量消息推送给消费端，导致消费端无法处理巨量消息。RabbitMQ提供了一种QOS服务质量保证功能，即在**非自动确认**消息前提下，如果一定数目的消息未被确认前，MQ不会推送新的消息给消费端。

- 消费端消费消息，autoAck 参数设置为false
  - channel.basicAck(deliveryTag, multiple) 手动签收
    - multiple false 不支持批量签收
- channel.basicQos(prefetchSize, prefetchCount, global)
  - 0 大小不作限制
  - 1 单条处理
  - false true是channel级别，false是consumer级别



### 消费端ACK与重回队列

<font color=red>消费端手动确认，通过MQ消息头参数 `amqp_deliveryTag` 手动确认提交，此参数是MQ生成确保唯一性。</font>

消费端手动ACK和NACK（仍回队列尾部）

- 消费端进行消费的时候，如果由于业务异常，可以进行日志的记录，然后进行补偿
- 如果由于服务器宕机等严重问题，需要手工进行ACK保障消费端消费成功

消费端的重回队列

为了对没有处理成功的消息，把消息重新回递给Broker；一般在实际应用中，都会关闭重回队列，设置为false



### TTL队列/消息

TTL Time To Live 生存时间

- RabbitMQ支持消息的过期时间，在消息发送时可以指定
- RabbitMQ支持队列的过期时间，从消息入队开始计算，只要超过了队列的超时时间，消息会自动清除



### 死信队列

DLX Dead Letter Exchange 利用DLX，当消息在一个队列中变成死信之后，它能被publish到另一个Exchange，这个Exchange就是DLX。DLX也是一个正常的Exchange，和一般的Exchange没有区别，能在任何队列上被指定，实际上就是设置某个队列的属性。

消息变成死信

- 消息被拒绝(basic.reject / basic.nack) 并且requeue=false
- 消息TTL过期
- 队列达到最大长度

死信队列

- 死信队列设置
  - Exchange：dlx.exchange
  - Queue：dlx.queue
  - RoutingKey：#
- 普通队列设置
  - 队列上加一个参数：arguments.put("x-dead-letter-exchange", "dlx.exchange")



## 集群架构

### 主备模式

实现RabbitMQ的高可用集群，一般在并发和数据量不高的情况下，这种模式非常的好用且简单。主备模式称为Warren模式。

- 主备方案：如果主节点挂了，从节点提供服务；主节点运行时，备节点对外不提供服务
- 利用Haproxy实现



### 远程模式

可以实现双活的一种模式，称为Shovel模式。可以把消息进行不同数据中心的复制，可以让两个MQ集群跨地域互联。



### 镜像模式

Mirror镜像模式，保证100%数据不丢失，在实际工作中使用最多。并且实现集群非常简单。为了保证RabbitMQ数据的高可靠性，主要实现就是数据同步，一般是3个节点实现数据同步。

- 由Haproxy负责负载均衡
- Haproxy部署Keepalived实现Haproxy的HA



### 多活模式

**实现异地数据复制的主流模式**，需要依赖RabbitMQ的federation插件，可以实现持续可靠的AMQP数据通信，多活模式在实际配置与应用非常简单。采用双中心/多中心各部署一套RabbitMQ集群，各中心RabbitMQ服务除了需要为业务提供正常的消息服务外，中心之间还需要实现部分队列消息共享。

不需要构建Cluster，可以在Brokers或Cluster之间传输消息，连接的双方可以使用不同的users和virtual hosts，可以使用不同版本的RabbitMQ和Erlang，因为federation插件使用AMQP协议通讯，可以接受不连续的传输。



## 插件

### 延迟插件

消息的延迟推送，定时消息执行。包括消息重试策略，以及用于削峰限流、降级的异步延迟消息机制，都是延迟队列的使用场景。
