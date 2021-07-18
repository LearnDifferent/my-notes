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

# GitHub

相关仓库：

- [my-rabbitmq-java-exercises](https://github.com/LearnDifferent/my-rabbitmq-java-exercises)
- [spring-amqp-sample](https://github.com/LearnDifferent/spring-amqp-sample)
