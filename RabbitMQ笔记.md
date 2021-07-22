[toc]

# MQ 相关基础

## MQ 基础

> MQ: Message Queue / 消息队列

类似于寄快递，有个发件人和收件人的基本信息，然后把快递交到快递员（公司）手上，快递包裹就会送到目的地。

生产者相当于发件人，消费者相当于收件人，MQ 相当于快递员/快递公司，消息相当于快递包裹。

## MQ 产品

比较主流的消息队列中间件有 Kafka 和 RabbitMQ。

MySQL 和 Redis 等部分数据库也可以实现消息队列的功能。

## MQ、AMQP 和 JMS 的关系

MQ 是遵循了 AMQP 协议的具体实现产品。

AMQP：Advanced Message Queuing Protocol

JMS：Java Message Service

主流的 MQ 产品基本都支持这两种。

# 消息的分发策略

相关代码：

- [my-rabbitmq-java-exercises](https://github.com/LearnDifferent/my-rabbitmq-java-exercises)
- [spring-amqp-sample](https://github.com/LearnDifferent/spring-amqp-sample)

## 发布订阅

类似于订阅 Youtube 频道和微信公众号。

## 轮询分发

假设有 99 条消息，有 3 个消费者。中间件会按照每个消费者分配 33 条，来发送消息。

## 公平分发

相当于公会悬赏任务。

冒险者（消费者）手头没任务的时候，就去公会（中间件）领取任务（消费消息）。

只不过，会有每次领取任务（获取消息）的限额。假设限额为 1，那么冒险者（消费者）只有在确认完成了 1 个任务后，公会才会给其下 1 个任务。

这样的话，有能力的消费者可以获取到更多的消息。

## 重发

如果中间件交给消费者 A 一个消息，消费者 A 接收消息失败了。

那么，中间件就会将该消息发给消费者 B，让消费者 B 来消费。

这样可以保证可靠性。

## 消息拉取

拉取消息。

# TTL 和 Dead Letter Exchange

> 相关代码：[spring-amqp-sample](https://github.com/LearnDifferent/spring-amqp-sample)

## TTL 过期时间

TTL：

1. 队列内的消息设置过期时间：消息在队列中存活的时间，时间到了就失效
2. 具体某条消息的过期时间：某条消息在时间到了之后，自己失效

## Dead Letter Exchange

下文摘抄自[官方教程](https://www.rabbitmq.com/dlx.html)

Messages from a queue can be "dead-lettered"; that is, republished to an exchange when any of the following events occur:

- 消息被拒绝：The message is [negatively acknowledged](https://www.rabbitmq.com/confirms.html) by a consumer using basic.reject or basic.nack with requeue parameter set to false
- 消息过期（参数名为：x-message-ttl）：The message expires due to [per-message TTL](https://www.rabbitmq.com/ttl.html)
- 消息超过最大长度（参数名为：x-max-length）：The message is dropped because its queue exceeded a [length limit](https://www.rabbitmq.com/maxlength.html)

>查看有哪些 Arguments 及其名称，可以到 RabbitMQ 的 Management 工具的网页上 ，点击标签页「Queues」，点击下方的「Add a new queue」按钮展开页面 ，可以看到有一行「Arguments」及其下面的参数配置。
>
>这些参数配置（比如"Message TTL"）是可以点击的按钮，点击后，就会显示参数的名称（比如显示"x-message-ttl"）。

Note that expiration of a queue will not dead letter the messages in it.（因为队列失效的消息，不计入中）

Dead letter exchanges (DLXs) are normal exchanges. They can be any of the usual types and are declared as usual.

For any given queue, a DLX can be defined by clients using the [queue's arguments](https://www.rabbitmq.com/queues.html#optional-arguments), or in the server using [policies](https://www.rabbitmq.com/parameters.html#policies). In the case where both policy and arguments specify a DLX, the one specified in arguments overrules the one specified in policy.

等消息被转移到 Dead Letter Exchange 的死信队列后，可以像平时监听普通队列的消费者那样，使用 `@RabbitListener` 和 `@RabbitHandler` 去消费 Dead Letter 相关的消息。

# RabbitMQ 相关基础

## 安装 RabbitMQ 和 Management 插件

安装方法在 [官网](https://www.rabbitmq.com/download.html) 上查看：

- [Homebrew](https://www.rabbitmq.com/install-homebrew.html)
- [Docker](https://hub.docker.com/_/rabbitmq)

RabbitMQ 还有一个 Management Plugin，可以用图形化界面管理 RabbitMQ。

如果使用的是 Docker 的安装方式，可以找tag（版本）后缀为 management 的镜像，这样就可以安装 RabbitMQ 后再自动安装 Management 插件了：

- `docker pull rabbitmq:3.8-management-alpine`

可以设置为带上 Management 插件，并设置用户名 user，密码 123，：

```bash
docker run -d --name rabbitmq -e RABBITMQ_DEFAULT_USER=user -e RABBITMQ_DEFAULT_PASS=123 -p 15672:15672 -p 5672:5672 -p 25672:25672 -p 61613:61613 -p 1883:1883 rabbitmq:3.8-management-alpine
```

这样输入后，访问 http://localhost:15672 即可。

## RabbitMQ 基础概念

Virtual host：

-   可能会有多个项目使用同一个 RabbitMQ，而每个项目的数据肯定是不同的
-   所以使用 Virtual host 可以独立分隔不同项目的数据

Cluster：

-   RabbitMQ 也可以使用集群
-   该集群的某个节点的名称，为「rabbit@主机名称」，使用 Docker 启动的时候，可以添加 `--hostname 主机名称` 来命名

端口：

-   5672：Java 使用这个端口（amqp 协议）
-   25672：集群（clustering）使用这个端口
-   15672：http 用这个端口

## RabbitMQ Maven 依赖

```xml
<dependency>
    <groupId>com.rabbitmq</groupId>
    <artifactId>amqp-client</artifactId>
    <version>5.10.0</version>
</dependency>
```

# RabbitMQ 集群

## 部署 RabbitMQ 集群

**这里使用多个 Docker 容器在本地部署单机多实例 RabbitMQ 集群环境**

启动三个节点，一个主节点，一个从节点。

这里先启动一个节点 rabbit1，作为主节点（这里的主节点设置账号密码，其余节点不设置）：

```bash
docker run -d --hostname rabbit1host --name rabbit1 -p 15672:15672 -p 5672:5672 -e RABBITMQ_ERLANG_COOKIE='rabbitcookie' -e RABBITMQ_DEFAULT_USER=user -e RABBITMQ_DEFAULT_PASS=123 rabbitmq:3.8-management-alpine
```

然后启动节点 rabbit2，使用 `--link $需要连接的节点的name:$需要连接的节点的hostname` 来连接。（注意修改暴露的端口，避免端口冲突）：

```bash
docker run -d --hostname rabbit2host --name rabbit2 -p 15673:15672 -p 5673:5672 --link rabbit1:rabbit1host -e RABBITMQ_ERLANG_COOKIE='rabbitcookie' rabbitmq:3.8-management-alpine
```

最后，启动节点 rabbit3，这个节点需要同时连接之前的所有节点：

```bash
docker run -d --hostname rabbit3host --name rabbit3 -p 15674:15672 -p 5674:5672 --link rabbit1:rabbit1host --link rabbit2:rabbit2host -e RABBITMQ_ERLANG_COOKIE='rabbitcookie' rabbitmq:3.8-management-alpine
```

---

等所有节点（容器）启动完成后，重启每个节点（目的是让该节点离开其当前的集群），将从节点加入到主节点中。

对于主节点 rabbit1：

```bash
docker exec -it rabbit1 bash

# 使用 rabbitmqctl 命令，来停止当前节点
rabbitmqctl stop_app
# 重置节点的的数据，用于离开当前的集群
rabbitmqctl reset
# 现在启动该节点
rabbitmqctl start_app

exit
```

对于从节点 rabbit2:

```bash
docker exec -it rabbit2 bash
rabbitmqctl stop_app
rabbitmqctl reset
# 将当前的（rabbit2）节点，加入到 rabbit1 节点中（注意匹配的是 hostname）
# --ram：表示设置为内存节点（如果没有这个，就表示默认为磁盘节点）
rabbitmqctl join_cluster --ram rabbit@rabbit1host
rabbitmqctl start_app
exit
```

从节点 rabbit3，也要加入到主节点 rabbit1 中：

```bash
docker exec -it rabbit3 bash
rabbitmqctl stop_app
rabbitmqctl reset
rabbitmqctl join_cluster --ram rabbit@rabbit1host
rabbitmqctl start_app
exit
```

> 加入完成后，可以进入任何一个节点（容器），使用 `rabbitmqctl cluster_status` 来查看集群信息

---

因为 RabbitMQ 使用的语言是 Erlang，所以 RabbitMQ 集群的节点间相互的通信需要 Erlang Cookie 的认证（相当于节点间交换信息的秘钥）。

> Erlang Cookie 的位置： `/var/lib/rabbitmq/.erlang.cookie` 

只要去集群中任意一个节点的 Erlang Cookie 所在的位置，将里面的内容拷贝。然后，去该集群的其他节点中，将拷贝的内容，覆盖（剪切）到该节点的 Erlang Cookie 中，就可以了。

注意：因为上面演示的时候，已经在启动节点容器的命令中，加入了 `RABBITMQ_ERLANG_COOKIE='rabbitcookie'` 参数，所以演示的时候，已经将所有节点的  Erlang Cookie，设置为了统一的 "rabbitcookie"。

参考资料：

- [使用docker搭建RabbitMQ集群](https://www.jianshu.com/p/d231844b9c46)
- [docker搭建RabbitMQ单机集群](https://www.jianshu.com/p/aa537ff043bc)
- [【集群运维篇】使用docker搭建RabbitMQ集群](https://blog.csdn.net/xia296/article/details/108395796)

## 管理 RabbitMQ 集群

下文摘抄自：[docker搭建RabbitMQ单机集群](https://www.jianshu.com/p/aa537ff043bc)

### 常用命令

**rabbitmqctl join_cluster {cluster_node} [–ram]**
 将节点加入指定集群中。在这个命令执行前需要停止RabbitMQ应用并重置节点。

**rabbitmqctl cluster_status**
 显示集群的状态。

**rabbitmqctl change_cluster_node_type {disc|ram}**
 修改集群节点的类型。在这个命令执行前需要停止RabbitMQ应用。

**rabbitmqctl forget_cluster_node [–offline]**
 将节点从集群中删除，允许离线执行。

**rabbitmqctl update_cluster_nodes {clusternode}**

在集群中的节点应用启动前咨询clusternode节点的最新信息，并更新相应的集群信息。这个和join_cluster不同，它不加入集群。考虑这样一种情况，节点A和节点B都在集群中，当节点A离线了，节点C又和节点B组成了一个集群，然后节点B又离开了集群，当A醒来的时候，它会尝试联系节点B，但是这样会失败，因为节点B已经不在集群中了。

**rabbitmqctl cancel_sync_queue [-p vhost] {queue}**
 取消队列queue同步镜像的操作。

**rabbitmqctl set_cluster_name {name}**
 设置集群名称。集群名称在客户端连接时会通报给客户端。Federation和Shovel插件也会有用到集群名称的地方。集群名称默认是集群中第一个节点的名称，通过这个命令可以重新设置。

### 设置节点类型

如果你想更换节点类型可以通过命令修改，如下：

> rabbitmqctl stop_app
>
> rabbitmqctl change_cluster_node_type dist
>
> rabbitmqctl change_cluster_node_type ram
>
> rabbitmqctl start_app

### 移除节点

如果想要把节点从集群中移除，可使用如下命令实现：

> rabbitmqctl stop_app
>
> rabbitmqctl restart
>
> rabbitmqctl start_app

### 集群重启顺序

**集群重启的顺序是固定的，并且是相反的。** 如下所述：

- 启动顺序：磁盘节点 => 内存节点
- 关闭顺序：内存节点 => 磁盘节点

**最后关闭必须是磁盘节点**，不然可能回造成集群启动失败、数据丢失等异常情况

# 其他资料

[RabbitMQ消息可靠投递和重复消费等问题解决方案](https://blog.csdn.net/qq_40837310/article/details/109033000)
