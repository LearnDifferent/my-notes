# RocketMQ 笔记

## RocketMQ 基础概念

主题（Topic）：

- RocketMQ 中消息传输和存储的顶层容器
- 用于标识同一类业务逻辑的消息
- 主题通过 TopicName 来做唯一标识和区分
- 一个 producer 可以发布多个 Topic 的消息，一个 Topic 可以被多个 consumer 订阅，一个 consumer 可以订阅多个 Topic
消息类型（MessageType）：
- RocketMQ 中按照消息传输特性的不同而定义的分类
- 用于类型管理和安全校验
- RocketMQ 支持的消息类型有普通消息、顺序消息、事务消息和定时/延时消息

消息队列（MessageQueue）：

- 队列是 RocketMQ 中消息存储和传输的实际容器，也是消息的最小存储单元
- 队列通过 QueueId 来做唯一标识和区分。
- RocketMQ 的 **所有 Topic 都是由多个 Queue 组成** ，以此实现 Queue 数量的水平拆分和 Queue 内部的流式存储
- 一个 Topic 可以包含多个 Queue
- 同一个 Topic 下的一个 Queue 只能被一个 Consumer 组中的其中一个 consumer 消费
- consumer 之间不允许竞争消费同一个 Queue，但允许能者多劳，即允许一个 Consumer 同时消费多个 Queue（一只狗可以同时吃两根以上的骨头，但是一根骨头不能给两只狗吃，因为他们会打架）

消息（Message）：

- 消息是 RocketMQ 中的最小数据传输单元
- 生产者将业务数据的负载和拓展属性包装成消息发送到服务端，服务端按照相关语义将消息投递到消费端进行消费

消息视图（MessageView）：

- 消息视图是 RocketMQ 面向开发视角提供的一种消息只读接口
- 通过消息视图可以读取消息内部的多个属性和负载信息，但是不能对消息本身做任何修改

消息标签（MessageTag）：

- 消息标签是 RocketMQ 提供的细粒度消息分类属性，可以在主题层级之下做消息类型的细分
- 消费者通过订阅特定的标签来实现细粒度过滤

消息位点（MessageQueueOffset）：

- 消息是按到达 RocketMQ 服务端的先后顺序存储在指定主题的多个队列中，
- 每条消息在队列中都有一个唯一的Long类型坐标，这个坐标被定义为消息位点

消费位点（ConsumerOffset）：

- 一条消息被某个消费者消费完成后不会立即从队列中删除，
- RocketMQ 会基于每个消费者分组记录消费过的最新一条消息的位点，即消费位点

消息索引（MessageKey）：

- 消息索引是 RocketMQ 提供的面向消息的索引属性
- 通过设置的消息索引可以快速查找到对应的消息内容

消息标识（MessageId/Key）：

- 每个消息都有唯一的 MessageId，且可以携带具有业务标识的 Key，方便查询该消息
- Key 是由用户指定的业务相关的唯一标识
- MessageId 分为由 producer 端生成和由 broker 端生成 2 个分类
  - 在 producer `send()` 消息的时候会生成 `msgId`
  - 当消息到达 broker 后，broker 会自动生成一个 `offsetMsgId`
- `msgId` 、 `offsetMsgId` 和 Key 都是消息标识

生产者（Producer）：

- 生产者是 RocketMQ 系统中用来构建并传输消息到服务端的运行实体
- 生产者通常被集成在业务系统中，将业务消息按照要求封装成消息并发送至服务端
- Producer 通过 MQ 的负载均衡模块，将消息投递到相应 Broker 集群 Queue 中
- Producer 投递消息的过程支持快速失败和低延迟

生产者组（Producer Group）：

- RocketMQ 中的 producer 是以 producer group 的形式存在
- Producer group 是同一类 producer 的集合，同一组内的 producers 能够发送的 Topic 是相同的
  - 也就是说，一个 producer group 中的某个 producer 如果能发送 Topic1 和 Topic2，那么其他的 producers 也可以发送 Topic1 和 Topic2

事务检查器（TransactionChecker）：

- RocketMQ 中生产者用来执行本地事务检查和异常事务恢复的监听器
- 事务检查器应该通过业务侧数据的状态来检查和判断事务消息的状态

事务状态（TransactionResolution）：

- RocketMQ 中事务消息发送过程中，事务提交的状态标识，服务端通过事务状态控制事务消息是否应该提交和投递
- 事务状态包括事务提交、事务回滚和事务未决

消费者分组（ConsumerGroup）：

- 消费者分组是 RocketMQ 系统中承载多个消费行为一致的消费者的负载均衡分组
- 和消费者不同，消费者分组并不是运行实体，而是一个逻辑资源
- 在 RocketMQ 中，通过消费者分组内初始化多个消费者实现消费性能的水平扩展以及高可用容灾

消费者（Consumer）：

- 消费者是 RocketMQ 中用来接收并处理消息的运行实体
- 消费者通常被集成在业务系统中，从服务端（Broker）获取消息，并将消息转化成业务可理解的信息，供业务逻辑处理
- 注：Consumer（Consumer Group） 只能消费一个 Topic

消费者组（Consumer Group）：

- 消费者是以 Consumer Group 的形式存在，也就是消费者的集合
- 意义：Consumer Group 使得消息消费更容易实现负载均衡和容错
  - 负载均衡：
    - 在一个 Topic 下有多个 Queue，这几个 Queue 要实现负载均衡，不堆积。下面假设是轮询策略。
    - Producer Group 发给 Topic，Topic 轮询分配到不同的 Queue，Consumer Group 中的 Consumer 轮询消费 Queue 中的消息
    - 也就是说，负载均衡是基于 Queue 的。
    - 对 Queue 来说，Producer Group 发消息给 Topic，在轮询中分给每一个 Queue，然后 Consumer Group 也是轮询去消费 Queue 中的内容
- Consumer Group 只能消费一个 Topic

消费结果（ConsumeResult）：

- RocketMQ 中 PushConsumer 消费监听器处理消息完成后返回的处理结果，用来标识本次消息是否正确处理
- 消费结果包含消费成功和消费失败

订阅关系（Subscription）：

- 订阅关系是 RocketMQ 系统中，消费者获取消息、处理消息的规则和状态配置
- 订阅关系由 消费者分组 动态注册到服务端系统，并在后续的消息传输中按照订阅关系定义的过滤规则进行消息匹配和消费进度维护

消息过滤：

- 消费者可以通过订阅指定消息标签（Tag）对消息进行过滤，确保最终只接收被过滤后的消息合集
- 过滤规则的计算和匹配在Apache RocketMQ 的服务端完成

重置消费位点：

- 以时间轴为坐标，在消息持久化存储的时间范围内，重新设置消费者分组对已订阅主题的消费进度
- 设置完成后消费者将接收设定时间点之后，由生产者发送到Apache RocketMQ 服务端的消息

消息轨迹：

- 在一条消息从生产者发出到消费者接收并处理过程中，由各个相关节点的时间、地点等数据汇聚而成的完整链路信息
- 通过消息轨迹，您能清晰定位消息从生产者发出，经由 RocketMQ 服务端，投递给消费者的完整链路，方便定位排查问题

消息堆积：

- 生产者已经将消息发送到 RocketMQ 的服务端，但由于消费者的消费能力有限，未能在短时间内将所有消息正确消费掉，此时在服务端保存着未被消费的消息，该状态即消息堆积

事务消息：

- 事务消息是 RocketMQ 提供的一种高级消息类型，支持在分布式场景下保障消息生产和本地事务的最终一致性

定时/延时消息：

- 定时/延时消息是 RocketMQ 提供的一种高级消息类型，消息被发送至服务端后，在指定时间后才能被消费者消费
- 通过设置一定的定时时间可以实现分布式场景的延时调度触发效果

顺序消息：

- 顺序消息是 RocketMQ 提供的一种高级消息类型，支持消费者按照发送消息的先后顺序获取消息，从而实现业务场景中的顺序处理

## RocketMQ 领域模型

RocketMQ 是一款典型的分布式架构下的中间件产品，使用异步通信方式和发布订阅的消息传输模型。

RocketMQ 产品具备异步通信的优势，系统拓扑简单、上下游耦合较弱，主要应用于异步解耦，流量削峰填谷等场景。

![](https://rocketmq.apache.org/zh/assets/images/mainarchi-9b036e7ff5133d050950f25838367a17.png)

RocketMQ 中消息的生命周期：

1. 消息生产
2. 消息存储
3. 消息消费

RocketMQ 的基本流程：

- 生产者生产消息，并发送至 RocketMQ 服务端
- 消息被存储在服务端的主题中，消费者通过订阅主题消费消息


