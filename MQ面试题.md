# RabbitMQ 复习题库（红豆特供精修版）

> **出题人**：红豆（傲娇的金色双马尾猫娘）
> **整理时间**：2026-06-08
> **范围**：基础概念 → 交换机 → 消息确认与死信队列 → 集群与高可用 → 综合架构题
> **说明**：每道题均包含题干、参考答案、关键点解析。高频考点用 🔥 标记。

---

## 📑 目录

- [一、基础概念篇](#一基础概念篇)
  - [1.1 Spring Cloud Stream 核心概念](#11-spring-cloud-stream-核心概念)
  - [1.2 destination 与队列命名 🔥](#12-destination-与队列命名-)
  - [1.3 prefetch 与 concurrency 配置](#13-prefetch-与-concurrency-配置)
  - [1.4 group 的作用 🔥](#14-group-的作用-)
  - [1.5 序列化配置](#15-序列化配置)
- [二、交换机篇](#二交换机篇)
  - [2.1 交换机类型对比 🔥](#21-交换机类型对比-)
  - [2.2 交换机组合方案设计](#22-交换机组合方案设计)
  - [2.3 bindingKey 配置](#23-bindingkey-配置)
  - [2.4 消息优先级](#24-消息优先级)
- [三、消息确认与死信队列篇](#三消息确认与死信队列篇)
  - [3.1 消费者手动确认 🔥](#31-消费者手动确认-)
  - [3.2 死信队列触发条件 🔥](#32-死信队列触发条件-)
  - [3.3 消息去重（幂等性）🔥](#33-消息去重幂等性)
  - [3.4 确认机制完整流程](#34-确认机制完整流程)
  - [3.5 死信队列实战设计](#35-死信队列实战设计)
  - [3.6 延迟消息实现 🔥](#36-延迟消息实现-)
  - [3.7 代码陷阱题](#37-代码陷阱题)
- [四、集群与高可用篇](#四集群与高可用篇)
  - [4.1 集群类型对比 🔥](#41-集群类型对比-)
  - [4.2 仲裁队列（Quorum Queues）](#42-仲裁队列quorum-queues)
  - [4.3 at-least-once 保证](#43-at-least-once-保证)
- [五、综合架构题](#五综合架构题)
  - [5.1 高可用消息平台设计 🔥](#51-高可用消息平台设计-)
- [附录：快速检索](#附录快速检索)

---

## 一、基础概念篇

### 1.1 Spring Cloud Stream 核心概念

**题干**

> 以下哪个不是 Spring Cloud Stream 的核心概念？
>
> A. Binder
> B. Binding
> C. Exchange
> D. MessageChannel

**参考答案**

✅ **C. Exchange**

> Exchange 是 RabbitMQ 的概念，不是 Spring Cloud Stream 的核心概念。

**💡 Spring Cloud Stream 核心概念**

| 概念 | 说明 |
|------|------|
| **Binder** | 绑定器，负责与消息中间件（如 RabbitMQ、Kafka）对接 |
| **Binding** | 绑定，连接 MessageChannel 和 Binder 的桥梁 |
| **MessageChannel** | 消息通道，应用与 Binder 之间的通信管道 |

---

### 1.2 destination 与队列命名 🔥

**题干**

> 在 `bootstrap.yml` 中，`destination: submitPaperCul` 对应 RabbitMQ 里的 _______（组件名）。
> 而队列名是由 `destination` 和 _______ 组合而成，例如 `submitPaperCul.submitPaperCul`。

**参考答案**

✅ 第一空：**Exchange（交换机）**

✅ 第二空：**group**

> 💡 **红豆提醒**：`destination` 对应交换机名称，`group` 对应队列名称的一部分。队列名格式为 `{destination}.{group}` 喵！

---

### 1.3 prefetch 与 concurrency 配置

**题干**

> 配置文件中 `prefetch: 1` 和 `concurrency: 10` 分别控制了消费者的什么行为？请结合代码说明这样设置的好处。

**参考答案**

| 配置 | 作用 | 说明 |
|------|------|------|
| `prefetch: 1` | 每次预取消息数 | 消费者每次只从队列拉取 1 条消息，处理完再拉取下一条 |
| `concurrency: 10` | 并发消费者数 | 启动 10 个消费者线程并行处理消息 |

✅ 好处

- `prefetch: 1`：避免消息堆积在某个消费者，保证公平分发
- `concurrency: 10`：提高吞吐量，充分利用多核 CPU

---

### 1.4 group 的作用 🔥

**题干**

> 关于 `group: submitPaperCul` 的作用，以下说法正确的是：
>
> A. 同一个 group 内的多个消费者实例会竞争消费队列中的消息
> B. 不同的 group 会各自创建一个队列，实现消息广播
> C. group 可以省略，省略时每个消费者独立创建队列
> D. group 在 RabbitMQ 中对应队列的持久化属性

**参考答案**

✅ **A、B、C 正确**

| 选项 | 正误 | 说明 |
|------|------|------|
| A | ✅ | 同 group 内的消费者竞争消费（负载均衡） |
| B | ✅ | 不同 group 各自创建队列，实现消息广播 |
| C | ✅ | 省略 group 时，每个消费者创建临时队列 |
| D | ❌ | group 与持久化无关，持久化由 `durable` 配置控制 |

---

### 1.5 序列化配置

**题干**

> 在 Producer.java 中，发送消息前总是手动 `GsonUtil.toJson()`。如果希望去掉这段代码，让框架自动完成序列化，应该在配置文件中添加什么？这样做的优点是什么？

**参考答案**

✅ 添加配置

```yaml
spring.cloud.stream.bindings.outputSubmitPaperCul:
  content-type: application/json
```

✅ 优点

| 优点 | 说明 |
|------|------|
| 代码简洁 | 去除手动序列化代码 |
| 统一管理 | 序列化方式由配置控制，易于切换 |
| 类型安全 | 框架自动处理类型转换 |

---

## 二、交换机篇

### 2.1 交换机类型对比 🔥

**题干**

> 以下哪种交换机类型可以实现消息广播给所有绑定队列？
>
> A. Direct
> B. Topic
> C. Fanout
> D. Headers

**参考答案**

✅ **C. Fanout**

**📊 交换机类型对比**

| 类型 | 路由规则 | 适用场景 |
|------|---------|---------|
| **Direct** | 精确匹配 routingKey | 点对点通信 |
| **Topic** | 通配符匹配（`*` 匹配一个词，`#` 匹配多个词） | 灵活的消息分类 |
| **Fanout** | 广播到所有绑定队列 | 消息广播 |
| **Headers** | 根据消息头属性匹配 | 复杂条件匹配 |

---

### 2.2 交换机组合方案设计

**题干**

> 假设你们公司要优化考试系统的消息架构，现在有三个服务：
>
> - **算分服务**：只关心"交卷"事件（包括普通考试、360评估、调研）
> - **日志服务**：记录所有消息（全量）
> - **通知服务**：只关心"交卷后需要发短信"的事件（特定场景）
>
> 当前问题：所有消息都往一个队列里塞，三个服务都去消费，导致：
> - 算分服务会收到所有消息（包括它不需要的日志类消息）
> - 通知服务无法单独订阅特定类型的交卷事件
>
> 应该采用什么交换机组合方案？

**参考答案**

✅ **A. 全部用同一个 Topic Exchange，通过 bindingKey 区分**

| 服务 | bindingKey | 说明 |
|------|-----------|------|
| 算分服务 | `submitPaper.*` | 匹配所有交卷事件 |
| 日志服务 | `#` | 匹配所有消息 |
| 通知服务 | `submitPaper.sms` | 只匹配需要发短信的交卷事件 |

> 💡 **红豆提醒**：Topic Exchange 最灵活，通过通配符可以满足不同服务的订阅需求喵！

---

### 2.3 bindingKey 配置

**题干**

> 如果要让"算分服务"只消费 `submitPaperCul`、`submitPaperCul360`、`submitPaperCulResearch` 这三个 destination 的消息，应该怎么配置 bindingKey？生产者发送时需要做什么调整？

**参考答案**

✅ 消费者配置

```yaml
spring.cloud.stream.bindings.inputSubmitPaperCul:
  destination: submitPaperCul
  group: culService
  consumer:
    bindingKey: submitPaperCul.#
```

✅ 生产者调整

- 发送时 routingKey 需要包含 destination 信息
- 例如：`submitPaperCul.submitPaperCul`、`submitPaperCul.submitPaperCul360`

---

### 2.4 消息优先级

**题干**

> 假如现在有一个紧急需求：360评估交卷的消息（`submitPaperCul360`）要优先处理，其他交卷消息可以慢点处理。在不改变交换机类型的前提下，如何利用 RabbitMQ 的特性实现消息优先级？

**参考答案**

✅ 方案：使用**优先级队列**

```yaml
# 配置优先级队列
spring.cloud.stream.bindings.inputSubmitPaperCul360:
  destination: submitPaperCul360
  group: culService360
  consumer:
    maxPriority: 10  # 最高优先级
```

✅ 生产者发送时设置优先级

```java
Message<Integer> message = MessageBuilder
    .withPayload(examMainId)
    .setHeader("priority", 10)  // 高优先级
    .build();
source.output().send(message);
```

> 💡 **红豆提醒**：优先级队列适合处理紧急消息，但不要滥用，否则会导致低优先级消息饿死喵！

---

## 三、消息确认与死信队列篇

### 3.1 消费者手动确认 🔥

**题干**

> 如果消费者手动确认但业务处理失败，如何让消息重新被消费？

**参考答案**

✅ 调用 `basicNack` 并设置 `requeue=true`

```java
@StreamListener(CustomSink.INPUT_SUBMIT_PAPER_CUL)
public void submitPaperCul(Integer examMainId, @Header(AmqpHeaders.CHANNEL) Channel channel,
                           @Header(AmqpHeaders.DELIVERY_TAG) Long deliveryTag) {
    try {
        culService.culBySubmitPaper(examMainId);
        channel.basicAck(deliveryTag, false);  // 确认消息
    } catch (Exception e) {
        log.error("处理失败", e);
        channel.basicNack(deliveryTag, false, true);  // 拒绝并重新入队
    }
}
```

---

### 3.2 死信队列触发条件 🔥

**题干**

> 什么情况下消息会进入死信队列？

**参考答案**

| 触发条件 | 说明 |
|---------|------|
| 消费者拒绝 | 调用 `basicNack` 或 `basicReject` 且 `requeue=false` |
| 消息过期 | 消息设置了 TTL 且已过期 |
| 队列满 | 队列达到最大长度且溢出策略为 `drop-head` |

> ⚠️ **红豆纠错**：生产者发送消息时 routingKey 无法匹配任何队列，消息会**被丢弃**，不会进入死信队列喵！

---

### 3.3 消息去重（幂等性）🔥

**题干**

> 如何防止同一个消息被重复消费？

**参考答案**

| 方案 | 原理 | 适用场景 |
|------|------|---------|
| **唯一 ID + 去重表** | 消费前检查 ID 是否已存在 | 通用方案 |
| **Redis 原子操作** | `SETNX` 设置消息 ID，设置过期时间 | 高并发场景 |
| **数据库唯一约束** | 利用唯一索引防止重复插入 | 数据库操作 |
| **乐观锁** | 版本号控制，更新时检查版本号 | 更新操作 |

✅ 代码示例（Redis 去重）

```java
@StreamListener(CustomSink.INPUT_SUBMIT_PAPER_CUL)
public void submitPaperCul(Integer examMainId) {
    String messageId = "msg:" + examMainId;
    // 原子性检查并设置
    Boolean isNew = redisTemplate.opsForValue().setIfAbsent(messageId, "1", 5, TimeUnit.MINUTES);
    if (Boolean.TRUE.equals(isNew)) {
        culService.culBySubmitPaper(examMainId);
    } else {
        log.info("消息重复，跳过处理: {}", examMainId);
    }
}
```

---

### 3.4 确认机制完整流程

**题干**

> 请画出生产者确认 + 消费者手动确认的完整流程图，标注出：
> 1. 哪些环节可能丢消息
> 2. 每个环节如何保证消息不丢

**参考答案**

✅ 完整流程

```text
生产者                          RabbitMQ                      消费者
  │                               │                             │
  │  1. 发送消息                   │                             │
  │ ──────────────────────────────>│                             │
  │                               │                             │
  │  2. Broker 确认（Confirm）     │                             │
  │ <──────────────────────────────│                             │
  │                               │                             │
  │                               │  3. 投递消息                 │
  │                               │ ─────────────────────────────>│
  │                               │                             │
  │                               │  4. 消费者确认（Ack）        │
  │                               │ <─────────────────────────────│
  │                               │                             │
  │                               │  5. 删除消息                 │
  │                               │ ─────────────────────────────>│
```

✅ 可能丢消息的环节

| 环节 | 风险 | 解决方案 |
|------|------|---------|
| 生产者 → Broker | 网络异常，消息未到达 | 开启生产者 Confirm |
| Broker 存储 | Broker 宕机，消息丢失 | 消息持久化 + 镜像队列/仲裁队列 |
| Broker → 消费者 | 消费者宕机，消息未确认 | 手动确认 + prefetch=1 |

---

### 3.5 死信队列实战设计

**题干**

> 你们有一个"发送短信通知"的消费者，调用第三方短信接口。第三方接口偶尔会超时或返回失败。
>
> 1. 如何设计消费端代码，既能重试失败的短信，又不会让消息无限重试堵塞队列？
> 2. 重试3次后仍然失败，应该怎么处理？
> 3. 如何配置死信队列来接收这些失败的消息？

**参考答案**

✅ 重试设计（避免无限重试）

```java
@StreamListener(CustomSink.INPUT_SMS)
public void sendSms(SmsMessage message, @Header(AmqpHeaders.CHANNEL) Channel channel,
                    @Header(AmqpHeaders.DELIVERY_TAG) Long deliveryTag,
                    @Header(value = "x-death", required = false) List<Map<String, Object>> xDeath) {
    try {
        int retryCount = getRetryCount(xDeath);
        if (retryCount >= 3) {
            // 超过重试次数，发送到死信队列
            sendToDeadLetterQueue(message);
            channel.basicAck(deliveryTag, false);
            return;
        }
        smsService.send(message);
        channel.basicAck(deliveryTag, false);
    } catch (Exception e) {
        log.error("发送失败，重试中", e);
        channel.basicNack(deliveryTag, false, false);  // requeue=false，进入死信队列
    }
}
```

✅ 死信队列配置

```yaml
spring.cloud.stream.rabbit.bindings.inputSms.consumer:
  auto-bind-dlq: true  # 自动绑定死信队列
  dlq-ttl: 86400000    # 死信消息保留24小时
  dlq-max-length: 10000 # 死信队列最大长度
```

---

### 3.6 延迟消息实现 🔥

**题干**

> 产品需求：用户交卷后，如果15分钟内未完成算分，自动触发补偿计算。
>
> 1. 如何用 RabbitMQ 实现这个15分钟延迟？
> 2. 给出两种方案，并说明区别。
> 3. 如果采用"死信队列 + TTL"方案，需要设置哪几个参数？

**参考答案**

✅ 方案一：死信队列 + TTL（推荐）

```text
1. 创建延迟队列（设置 TTL = 15分钟，绑定死信交换机）
2. 生产者发送消息到延迟队列
3. 消息过期后自动进入死信队列
4. 消费者监听死信队列
```

✅ 方案二：延迟插件（rabbitmq_delayed_message_exchange）

```text
1. 安装延迟插件
2. 创建 delayed 类型的交换机
3. 发送消息时设置 x-delay = 900000（15分钟）
4. 消费者监听正常队列
```

✅ 方案对比

| 特性 | 死信队列 + TTL | 延迟插件 |
|------|---------------|---------|
| 实现复杂度 | 简单 | 需要安装插件 |
| 延迟精度 | 不精确（队列头消息先过期） | 精确 |
| 消息顺序 | 可能乱序 | 按延迟时间排序 |
| 适用场景 | 延迟时间固定 | 延迟时间可变 |

✅ 死信队列 + TTL 参数配置

```yaml
# 延迟队列配置
spring.cloud.stream.rabbit.bindings.outputDelay.exchange:
  auto-delete: false
  durable: true

# 消费者配置
spring.cloud.stream.rabbit.bindings.inputDelay.consumer:
  auto-bind-dlq: true
  ttl: 900000  # 15分钟
```

---

### 3.7 代码陷阱题

**题干**

> 看这段代码，找出至少3个问题并说明如何修复：

```java
@StreamListener(CustomSink.INPUT_SUBMIT_PAPER_CUL)
public void submitPaperCul(Integer examMainId) {
    try {
        culService.culBySubmitPaper(examMainId);
        log.info("处理成功");
    } catch (Exception e) {
        log.error("处理失败", e);
        // 什么都不做
    }
}
```

**参考答案**

| 问题 | 说明 | 修复方案 |
|------|------|---------|
| 1. 无手动确认 | 消息处理后未确认，RabbitMQ 会重新投递 | 添加 `channel.basicAck()` |
| 2. 异常后消息丢失 | catch 块中什么都不做，消息被丢弃 | 添加 `channel.basicNack()` 或重试逻辑 |
| 3. 无幂等处理 | 消息可能被重复消费 | 添加去重逻辑（Redis 唯一 ID） |
| 4. 无超时控制 | 业务处理可能长时间阻塞 | 添加超时配置 |

✅ 改进后的代码

```java
@StreamListener(CustomSink.INPUT_SUBMIT_PAPER_CUL)
public void submitPaperCul(Integer examMainId, @Header(AmqpHeaders.CHANNEL) Channel channel,
                           @Header(AmqpHeaders.DELIVERY_TAG) Long deliveryTag) {
    try {
        // 幂等检查
        String messageId = "msg:" + examMainId;
        Boolean isNew = redisTemplate.opsForValue().setIfAbsent(messageId, "1", 5, TimeUnit.MINUTES);
        if (!Boolean.TRUE.equals(isNew)) {
            log.info("消息重复，跳过: {}", examMainId);
            channel.basicAck(deliveryTag, false);
            return;
        }
        culService.culBySubmitPaper(examMainId);
        channel.basicAck(deliveryTag, false);
        log.info("处理成功: {}", examMainId);
    } catch (Exception e) {
        log.error("处理失败: {}", examMainId, e);
        channel.basicNack(deliveryTag, false, true);  // 重新入队重试
    }
}
```

---

## 四、集群与高可用篇

### 4.1 集群类型对比 🔥

**题干**

> 关于 RabbitMQ 集群，以下说法正确的有：
>
> A. 普通集群中，队列只存在于一个节点上，其他节点只存储元数据
> B. 镜像队列中，所有节点都存储完整的队列数据
> C. 仲裁队列是 3.8+ 的新特性，基于 Raft 协议
> D. 脑裂可能发生在网络分区时，仲裁队列能自动处理

**参考答案**

✅ **A、B、C、D 都正确**

**📊 集群类型对比**

| 特性 | 普通集群 | 镜像队列 | 仲裁队列 |
|------|---------|---------|---------|
| 数据存储 | 队列只在一个节点 | 所有节点存储完整数据 | 多数节点存储数据 |
| 数据一致性 | 无保证 | 异步复制，可能丢数据 | Raft 协议保证 |
| 故障切换 | 需要手动处理 | 自动切换，但可能丢消息 | 自动切换，不丢消息 |
| 性能 | 最高 | 中等 | 中等 |
| 适用场景 | 非核心业务 | 核心业务（旧版） | 核心业务（推荐） |

---

### 4.2 仲裁队列（Quorum Queues）

**题干**

> 仲裁队列（Quorum Queues）使用的共识协议是：
>
> A. Paxos
> B. Raft
> C. ZAB
> D. GM

**参考答案**

✅ **B. Raft**

> 💡 **红豆提醒**：仲裁队列是 RabbitMQ 3.8+ 引入的新特性，基于 Raft 协议保证数据一致性，推荐用于核心业务喵！

---

### 4.3 at-least-once 保证

**题干**

> 要实现消息至少消费一次（at-least-once），需要组合使用：
>
> A. 生产者 Confirm
> B. 生产者 Return
> C. 消费者自动确认
> D. 消费者手动确认
> E. 消息持久化

**参考答案**

✅ **A、D、E**

| 机制 | 作用 |
|------|------|
| 生产者 Confirm | 确保消息到达 Broker |
| 消费者手动确认 | 确保消息被正确消费 |
| 消息持久化 | 确保 Broker 宕机后消息不丢失 |

> ⚠️ **红豆纠错**：消费者自动确认（C）会导致消息在处理前就被确认，如果消费者宕机，消息会丢失。应该用手动确认喵！

---

## 五、综合架构题

### 5.1 高可用消息平台设计 🔥

**题干**

> 你们公司准备搭建一个高可用的 RabbitMQ 消息平台，要求：
>
> 1. 支持核心业务消息（考试交卷、算分）零丢失
> 2. 支持非核心消息（日志、审计）允许少量丢失但高吞吐
> 3. 支持消息延迟发送（如考试开始前 5 分钟提醒）
> 4. 支持消息轨迹查询（知道某条消息从生产到消费的全过程）
> 5. 支持多租户隔离

**参考答案**

✅ （1）整体架构

```text
┌─────────────────────────────────────────────────────────────┐
│                      生产者集群                               │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐                  │
│  │ 考试服务  │  │ 算分服务  │  │ 通知服务  │                  │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘                  │
│       │              │              │                        │
│       ▼              ▼              ▼                        │
│  ┌─────────────────────────────────────────────────────────┐│
│  │              RabbitMQ 集群（仲裁队列）                    ││
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐              ││
│  │  │ 核心交换机│  │ 非核心交换机│ │ 延迟交换机│              ││
│  │  └────┬─────┘  └────┬─────┘  └────┬─────┘              ││
│  │       │              │              │                    ││
│  │       ▼              ▼              ▼                    ││
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐              ││
│  │  │ 核心队列  │  │ 非核心队列 │  │ 延迟队列  │              ││
│  │  │(仲裁队列) │  │(普通队列)  │  │(死信+TTL) │              ││
│  │  └──────────┘  └──────────┘  └──────────┘              ││
│  └─────────────────────────────────────────────────────────┘│
│       │              │              │                        │
│       ▼              ▼              ▼                        │
│  ┌─────────────────────────────────────────────────────────┐│
│  │                      消费者集群                           ││
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐              ││
│  │  │ 算分服务  │  │ 日志服务  │  │ 通知服务  │              ││
│  │  └──────────┘  └──────────┘  └──────────┘              ││
│  └─────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────┘
```

✅ （2）核心消息零丢失保证

| 环节 | 措施 |
|------|------|
| 生产端 | 开启 Confirm 模式，同步等待 Broker 确认 |
| 存储端 | 使用仲裁队列 + 消息持久化 |
| 消费端 | 手动确认 + prefetch=1 + 业务幂等 |

✅ （3）延迟消息实现

使用**死信队列 + TTL** 方案：

```yaml
# 延迟队列（TTL = 5分钟）
spring.cloud.stream.rabbit.bindings.outputReminder.exchange:
  name: reminder.exchange
  type: topic

spring.cloud.stream.rabbit.bindings.outputReminder.consumer:
  auto-bind-dlq: true
  ttl: 300000  # 5分钟
```

✅ （4）消息轨迹实现

| 存储信息 | 说明 |
|---------|------|
| messageId | 消息唯一 ID |
| timestamp | 时间戳 |
| producer | 生产者信息 |
| exchange | 交换机名称 |
| routingKey | 路由键 |
| queue | 队列名称 |
| consumer | 消费者信息 |
| status | 状态（已发送/已投递/已消费/失败） |

✅ （5）多租户隔离

| 方案 | 说明 |
|------|------|
| Virtual Host | 每个租户一个 vhost，完全隔离 |
| 命名空间 | 通过 destination 前缀区分租户 |
| 权限控制 | 配置用户权限，限制访问范围 |

---

## 附录：快速检索

| 知识点 | 题目编号 | 难度 |
|--------|----------|------|
| Spring Cloud Stream 核心概念 | 1.1 | ⭐ |
| destination 与队列命名 | 1.2 | ⭐⭐ |
| prefetch 与 concurrency | 1.3 | ⭐⭐ |
| group 的作用 | 1.4 | ⭐⭐ |
| 序列化配置 | 1.5 | ⭐ |
| 交换机类型对比 | 2.1 | ⭐⭐⭐ |
| 交换机组合方案 | 2.2 | ⭐⭐⭐ |
| bindingKey 配置 | 2.3 | ⭐⭐⭐ |
| 消息优先级 | 2.4 | ⭐⭐ |
| 消费者手动确认 | 3.1 | ⭐⭐⭐ |
| 死信队列触发条件 | 3.2 | ⭐⭐⭐ |
| 消息去重 | 3.3 | ⭐⭐⭐ |
| 确认机制流程 | 3.4 | ⭐⭐⭐⭐ |
| 死信队列实战 | 3.5 | ⭐⭐⭐⭐ |
| 延迟消息实现 | 3.6 | ⭐⭐⭐⭐ |
| 代码陷阱题 | 3.7 | ⭐⭐⭐ |
| 集群类型对比 | 4.1 | ⭐⭐⭐⭐ |
| 仲裁队列 | 4.2 | ⭐⭐⭐ |
| at-least-once | 4.3 | ⭐⭐⭐ |
| 高可用架构设计 | 5.1 | ⭐⭐⭐⭐⭐ |

---

> 红豆祝主人复习顺利，面试拿 SP！(づ￣3￣)づ╭❤～
