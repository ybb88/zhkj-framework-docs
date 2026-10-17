# 消息队列通信组件：zhkj-framework-starter-kafka

> **模块定位**: 分布式消息队列 Starter，基于 Spring Kafka 深度封装，提供统一发布订阅契约（`ZhkjKafka`）、全链路 TraceId 自动染色、租户身份与安全上下文双向穿透，以及消费异常时的上下文强制双清理安全机制。  
> **适用基准**: OpenJDK 21 (LTS), Spring Boot 4.1.1, Apache Kafka 3.x+  
> **模块坐标**: `io.github.ybb88:zhkj-framework-starter-kafka`

---

## 📋 1. 架构定位与核心设计

在异步事件驱动与流式处理架构中，消息在跨服务传递时，常常丢失了原始请求的上下文（如链路 TraceId、多租户身份 `tenant_id`、发起操作人凭证）。这会导致：
1. 消费端处理逻辑脱离租户行级隔离保护，引发越权灾难；
2. 消费端异常时 MDC 未清理，常驻 Worker 线程复用导致日志 TraceId 串号；
3. 全链路日志追踪断裂，难以追踪跨异步消息的完整调用拓扑。

`zhkj-framework-starter-kafka` 构建了全链路穿透与防污染闭环：
- **`ZhkjKafka` 统一发布门面**：封装统一的消息发送契约，支持异步回调、同步阻塞与事务发送；
- **生产者上下文自动拦截注入 (`KafkaContextProducerInterceptor`)**：在发送消息的瞬间，自动从当前线程 `FrameworkContext` 中抓取租户、用户身份与链路 TraceId，写入 Kafka Record Headers 中；
- **消费者双回调强清理防污染 (`KafkaTraceRecordInterceptor`)**：
  - 在接收到消息时，自动从 Headers 中解析并还原 `FrameworkContext` 与 MDC；
  - **异常毁约清理保证**：同时覆写 `afterRecord` 与 `failure` 回调！无论业务 Listener 正常执行完毕还是抛出未捕获异常，均强制在 `finally` 阶段调用 `FrameworkContext.clearAll()` 与 `TraceUtil.clearTraceId()`，彻底堵死 Worker 线程池污染漏洞。

---

## 📦 2. 依赖坐标

```xml
<dependency>
    <groupId>io.github.ybb88</groupId>
    <artifactId>zhkj-framework-starter-kafka</artifactId>
</dependency>
```

---

## 🧱 3. 核心包结构与组件

```
io.github.ybb88.framework.kafka
├── client/
│   └── ZhkjKafka.java                 # 统一发布订阅核心客户端门面
├── config/
│   ├── KafkaAutoConfiguration.java    # 消息队列核心配置装配
│   └── KafkaProperties.java           # Kafka 全局参数配置
└── interceptor/
    ├── KafkaContextProducerInterceptor.java # 生产者上下文写入拦截器
    ├── KafkaTraceRecordInterceptor.java     # 单条记录消费者全生命周期清理拦截器
    └── KafkaTraceBatchInterceptor.java      # 批量记录消费者双清理拦截器
```

---

## ⚙️ 4. 核心配置清单 (`application.yml`)

```yaml
spring:
  kafka:
    bootstrap-servers: 127.0.0.1:9092
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.apache.kafka.common.serialization.StringSerializer
      retries: 3
      acks: 1
    consumer:
      group-id: demo-service-group
      auto-offset-reset: latest
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      enable-auto-commit: true
```

---

## 💡 5. 典型使用范式

### 5.1 发送异步消息 (`ZhkjKafka`)
```java
package io.github.ybb88.demo.service;

import io.github.ybb88.framework.kafka.client.ZhkjKafka;
import io.github.ybb88.framework.util.JsonUtil;
import io.github.ybb88.demo.dto.OrderCreatedEvent;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;

@Service
@RequiredArgsConstructor
public class OrderMessageProducer {

    private final ZhkjKafka zhkjKafka;

    public void publishOrderCreated(OrderCreatedEvent event) {
        String topic = "topic-order-created";
        String key = event.getOrderNo();
        String payload = JsonUtil.toJson(event);

        // 统一发送契约：底层拦截器全自动将当前线程的 TraceId、TenantId 注入 Record Headers
        zhkjKafka.send(topic, key, payload);
    }
}
```

### 5.2 消费消息与自动还原上下文
```java
package io.github.ybb88.demo.listener;

import io.github.ybb88.framework.tenant.context.TenantContext;
import io.github.ybb88.framework.util.JsonUtil;
import io.github.ybb88.demo.dto.OrderCreatedEvent;
import lombok.extern.slf4j.Slf4j;
import org.apache.kafka.clients.consumer.ConsumerRecord;
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.stereotype.Component;

@Slf4j
@Component
public class OrderMessageConsumer {

    /**
     * 消费端自动由 KafkaTraceRecordInterceptor 拦截并还原上下文：
     * 1. 日志框架 MDC 中已自动绑定发送端的 TraceId，实现日志穿透；
     * 2. TenantContext.getTenantId() 自动拥有发送端的租户身份；
     * 3. 无论消费成功或异常，拦截器均会强制执行双清理，绝不发生线程池复用污染。
     */
    @KafkaListener(topics = "topic-order-created")
    public void onMessage(ConsumerRecord<String, String> record) {
        String currentTenant = TenantContext.getTenantId();
        log.info("接收到订单消息，当前自动还原租户: [{}]，Key: [{}]", currentTenant, record.key());

        OrderCreatedEvent event = JsonUtil.parse(record.value(), OrderCreatedEvent.class);
        // 执行业务消费处理
    }
}
```
