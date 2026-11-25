# 消息队列通信组件：zhkj-framework-starter-kafka

> **模块定位**: 基于 Spring Kafka 的消息 Starter，提供统一生产与消费的 `KafkaTemplate` / `ConcurrentKafkaListenerContainerFactory` 配置、高阶门面 `ZhkjKafka`，以及在生产与消费两端透传链路与租户上下文的拦截器。  
> **适用基准**: OpenJDK 21 (LTS), Spring Boot 4.1.1, Spring Kafka 4.1.0（根 pom `spring-kafka.version`）  
> **模块坐标**: `io.github.ybb88:zhkj-framework-starter-kafka`

---

## 📋 1. 架构定位与核心设计

1. **统一客户端配置**: 生产者与消费者工厂由本模块装配，参数来自 `zhkj.kafka.*`，Key 与 Value 序列化器固定为 String。
2. **高阶门面**: `ZhkjKafka` 将对象负载序列化为 JSON 字符串后发送，并写入 TraceId 头。
3. **生产端透传**: `KafkaContextProducerInterceptor` 将 TraceId、主体租户、维度与操作人信息写入 Record Header；不转发未核验委托或票据。
4. **消费端还原**: `KafkaTraceRecordInterceptor` 与 `KafkaTraceBatchInterceptor` 从 Header 还原 `FrameworkContext` 与 TraceId，并在完成后清理。

---

## 📦 2. 依赖坐标

```xml
<dependency>
    <groupId>io.github.ybb88</groupId>
    <artifactId>zhkj-framework-starter-kafka</artifactId>
</dependency>
```

依赖（来自 `pom.xml`）: `zhkj-framework-core`、`spring-kafka`、`spring-boot-starter`；`spring-boot-configuration-processor` 为 optional。

---

## 🧱 3. 包结构

```
io.github.ybb88.framework.kafka
├── config/ZhkjKafkaAutoConfiguration.java
├── core/ZhkjKafka.java
├── interceptor/
│   ├── KafkaContextProducerInterceptor.java   # ProducerInterceptor
│   ├── KafkaTraceRecordInterceptor.java       # RecordInterceptor (单条消费)
│   └── KafkaTraceBatchInterceptor.java        # BatchInterceptor (批量消费)
└── props/ZhkjKafkaProperties.java             # zhkj.kafka.*
```

---

## 🔁 4. 自动装配

`META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` 注册 `ZhkjKafkaAutoConfiguration`（`@EnableKafka`，条件 `zhkj.kafka.enabled` 缺省为 `true`）。

| Bean | 条件 | 说明 |
| :--- | :--- | :--- |
| `ProducerFactory<String, Object>` | 缺失时 | `DefaultKafkaProducerFactory`，注册 `KafkaContextProducerInterceptor` |
| `KafkaTemplate<String, Object>` | 缺失时 | — |
| `ConsumerFactory<String, Object>` | 缺失时 | `DefaultKafkaConsumerFactory` |
| `kafkaTraceRecordInterceptor` | 名称缺失时 | `KafkaTraceRecordInterceptor` |
| `kafkaTraceBatchInterceptor` | 名称缺失时 | `KafkaTraceBatchInterceptor` |
| `kafkaListenerContainerFactory` | 名称缺失时 | 并发数、轮询超时来自 `zhkj.kafka.*`，挂载两个 Trace 拦截器 |
| `zhkjKafka`（`ZhkjKafka`） | 缺失时 | 若容器中存在 Jackson 3 `ObjectMapper`（`tools.jackson.databind.ObjectMapper`）则注入，否则使用 `JsonUtil.getInstance()` |

### 4.1 地址与消费组解析

| 参数 | 解析顺序 |
| :--- | :--- |
| 连接地址 | `spring.kafka.bootstrap-servers` → `zhkj.kafka.bootstrap-servers` |
| 消费组 | `spring.kafka.consumer.group-id` → `${spring.application.name}-group` → `zhkj.kafka.group-id` |

> 生产者与消费者的其它参数（如 `acks`、`spring.kafka.producer.*`、`spring.kafka.consumer.*`）不参与本模块工厂的构建。本模块仅读取上表与 `zhkj.kafka.*` 项。

---

## ⚙️ 5. 配置项（`zhkj.kafka`）

| 键 | 类型 | 默认值 | 说明 |
| :--- | :--- | :--- | :--- |
| `zhkj.kafka.enabled` | boolean | `true` | 是否启用本模块 |
| `zhkj.kafka.bootstrap-servers` | String | `127.0.0.1:9092` | 连接地址（`spring.kafka.bootstrap-servers` 优先） |
| `zhkj.kafka.group-id` | String | `zhkj-default-group` | 仅在无 `spring.kafka.consumer.group-id` 且无 `spring.application.name` 时使用 |
| `zhkj.kafka.auto-offset-reset` | String | `earliest` | 消费者 `auto.offset.reset` |
| `zhkj.kafka.enable-auto-commit` | boolean | `false` | 消费者 `enable.auto.commit` |
| `zhkj.kafka.retries` | int | `3` | 生产者 `retries` |
| `zhkj.kafka.batch-size` | int | `16384` | 生产者 `batch.size` |
| `zhkj.kafka.linger-ms` | int | `5` | 生产者 `linger.ms` |
| `zhkj.kafka.buffer-memory` | int | `33554432` | 生产者 `buffer.memory` |
| `zhkj.kafka.concurrency` | int | `1` | 监听容器并发数 |
| `zhkj.kafka.poll-timeout` | long | `3000` | 监听容器轮询超时（毫秒） |

```yaml
spring:
  kafka:
    bootstrap-servers: 127.0.0.1:9092
  application:
    name: demo-service
zhkj:
  kafka:
    enabled: true
    concurrency: 3
    poll-timeout: 3000
```

---

## 📨 6. 上下文透传

### 6.1 生产端

`KafkaContextProducerInterceptor.onSend` 在消息发送前，将以下信息写入 Record Header（已存在的头不覆盖）:

| Header | 来源 |
| :--- | :--- |
| `X-Trace-Id` | `TraceUtil.getTraceId()`，为空时生成 |
| `X-Tenant-Id` | `FrameworkContext` 快照 `tenantId` |
| `X-Dimension-Id` | `dimensionId` |
| `X-Delegated-Tenant-Id` / `X-Grant-Ticket` | 不生成；无可信票据授权契约 |
| `X-User-Id` | `userId` |
| `X-User-Account` | `userAccount` |
| `X-User-Name` | `userName` |
| `X-Dept-Id` | `deptId` |

`ZhkjKafka.send` 另行写入 `X-Trace-Id`（来自 MDC）。

### 6.2 消费端

`KafkaTraceRecordInterceptor.intercept` 与 `KafkaTraceBatchInterceptor.intercept`（批量取首条记录的头，以批次为单位）:

1. 设置 MDC 中的 `traceId`（缺失时生成）；
2. 从 Header 构造 `ContextSnapshot`，租户取 `X-Tenant-Id`，其次 `tenant-id`、`zhkj-tenant-id`；
   对入站委托目标与票据不恢复到上下文，不能借 Kafka 消息建立跨租户权限；
3. 非空快照调用 `FrameworkContext.restore`。

完成后（单条：`success`、`failure`、`afterRecord`；批次：`success`、`failure`）调用 `clearContext()`，其实现为 `try { FrameworkContext.clearAll(); } finally { TraceUtil.clearTraceId(); }`，即使清空过程异常也会清理 TraceId。

---

## 💡 7. 使用范式

### 7.1 发送消息
```java
import io.github.ybb88.framework.kafka.core.ZhkjKafka;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;

@Service
@RequiredArgsConstructor
public class OrderEventPublisher {

    private final ZhkjKafka zhkjKafka;

    public void publishCreated(OrderCreatedEvent event) {
        // 异步发送，负载对象自动序列化为 JSON
        zhkjKafka.send("topic-order-created", event.getOrderNo(), event);
    }

    public void publishSync(String payload) throws Exception {
        // 同步发送，超时 5 秒
        zhkjKafka.sendSync("topic-audit", payload, 5L);
    }
}
```

`ZhkjKafka` 方法:

| 方法 | 返回值 |
| :--- | :--- |
| `send(String topic, Object data)` | `CompletableFuture<SendResult<String, Object>>` |
| `send(String topic, String key, Object data)` | `CompletableFuture<SendResult<String, Object>>` |
| `sendSync(String topic, Object data, long timeoutSeconds)` | `SendResult<String, Object>` |
| `sendSync(String topic, String key, Object data, long timeoutSeconds)` | `SendResult<String, Object>` |
| `getKafkaTemplate()` | `KafkaTemplate<String, Object>` |

`sendSync` 系列在超时或失败时抛出 `RuntimeException`（消息前缀 `Kafka 同步发送异常`）；`send` 系列的异常由返回的 `CompletableFuture` 承载，并在 `whenComplete` 中记录 ERROR 日志。

负载转换规则: `null` 发送空字符串；`CharSequence`、`Number`、`Boolean` 使用 `String.valueOf`；其它对象通过 `objectMapper.writeValueAsString` 序列化（未注入时为 `JsonUtil.getInstance()`），失败时回退 `String.valueOf`。

### 7.2 消费消息
```java
import org.apache.kafka.clients.consumer.ConsumerRecord;
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.stereotype.Component;

@Component
public class OrderCreatedListener {

    @KafkaListener(topics = "topic-order-created")
    public void onMessage(ConsumerRecord<String, String> record) {
        // 消费端已由 KafkaTraceRecordInterceptor 还原 TraceId 与 FrameworkContext
        OrderCreatedEvent event = JsonUtil.parse(record.value(), OrderCreatedEvent.class);
    }
}
```

消费者的 Value 反序列化器为 `StringDeserializer`，监听方法的参数类型应为 `String` 或 `ConsumerRecord<String, String>`。

---

## 🔗 8. 相关文档

- 核心工具与上下文: [core](./core.md)
- Web 层过滤器与 TraceId: [starter-web](./starter-web.md)
