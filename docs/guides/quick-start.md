# ZHKJ Framework 快速上手指南

> **目标**: 5 分钟内使用 `zhkj-framework` 搭建一个具备多租户隔离、审计填充、敏感国密加密与检索、响应脱敏、分布式锁、RPC 调用与在线文档的标准企业级微服务应用。
> **适用版本**: OpenJDK 21 (LTS), Spring Boot 4.1.1, Spring Cloud 2025.1.2

---

## 🛠️ 1. 环境准备

- **JDK**: OpenJDK 21 (LTS) (`21.0.2+`)
- **Maven**: 3.9.0+
- **IDE**: IntelliJ IDEA 2024+ 或 VSCode (务必开启 Annotation Processors)

---

## 📦 2. 创建微服务与引入起步依赖

在业务微服务（例如 `sample-order-service`）的 `pom.xml` 中引入框架核心 Starter：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>io.github.ybb88</groupId>
        <artifactId>zhkj-framework</artifactId>
        <version>2026.4.1.0-SNAPSHOT</version>
    </parent>

    <artifactId>sample-order-service</artifactId>

    <dependencies>
        <!-- 1. 统一应用引导启动器 -->
        <dependency>
            <groupId>io.github.ybb88</groupId>
            <artifactId>zhkj-framework-launcher</artifactId>
        </dependency>

        <!-- 2. Web MVC 增强与统一异常处理 -->
        <dependency>
            <groupId>io.github.ybb88</groupId>
            <artifactId>zhkj-framework-starter-web</artifactId>
        </dependency>

        <!-- 3. 声明式 HTTP 客户端 (微服务 RPC 通信) -->
        <dependency>
            <groupId>io.github.ybb88</groupId>
            <artifactId>zhkj-framework-starter-http</artifactId>
        </dependency>

        <!-- 4. MyBatis-Plus 持久层、多租户行级隔离与国密加密 -->
        <dependency>
            <groupId>io.github.ybb88</groupId>
            <artifactId>zhkj-framework-starter-mybatis</artifactId>
        </dependency>
        <dependency>
            <groupId>io.github.ybb88</groupId>
            <artifactId>zhkj-framework-starter-tenant</artifactId>
        </dependency>
        <dependency>
            <groupId>io.github.ybb88</groupId>
            <artifactId>zhkj-framework-starter-crypto</artifactId>
        </dependency>

        <!-- 5. 分布式缓存与声明式分布式锁 -->
        <dependency>
            <groupId>io.github.ybb88</groupId>
            <artifactId>zhkj-framework-starter-redis</artifactId>
        </dependency>

        <!-- 6. 注册与配置中心 (Nacos) -->
        <dependency>
            <groupId>io.github.ybb88</groupId>
            <artifactId>zhkj-framework-starter-nacos</artifactId>
        </dependency>

        <!-- 7. 在线接口文档 (Knife4j / OpenAPI 3) -->
        <dependency>
            <groupId>io.github.ybb88</groupId>
            <artifactId>zhkj-framework-starter-openapi</artifactId>
        </dependency>

        <!-- 数据库驱动 -->
        <dependency>
            <groupId>com.mysql</groupId>
            <artifactId>mysql-connector-j</artifactId>
            <scope>runtime</scope>
        </dependency>
    </dependencies>
</project>
```

---

## 🚀 3. 编写主启动类

使用 `ZhkjApplication.run(...)` 启动，框架自动完成多环境配置检测、系统环境自愈与启动耗时统计：

```java
package io.github.ybb88.sample.order;

import io.github.ybb88.framework.http.client.annotation.EnableHttpClients;
import io.github.ybb88.framework.launcher.launch.ZhkjApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@EnableHttpClients(basePackages = "io.github.ybb88.sample.**.client")
@SpringBootApplication
public class SampleOrderApplication {

    public static void main(String[] args) {
        ZhkjApplication.run("sample-order-service", SampleOrderApplication.class, args);
    }
}
```

---

## 🗄️ 4. 实体模型：多租户隔离、国密加密入库与响应打码

实体继承 `MybatisTenantEntity`，审计字段、租户物理隔离字段、国密加密与盲索引全自动注入：

```java
package io.github.ybb88.sample.order.entity;

import com.baomidou.mybatisplus.annotation.TableField;
import com.baomidou.mybatisplus.annotation.TableName;
import io.github.ybb88.framework.crypto.storage.annotation.BlindIndex;
import io.github.ybb88.framework.crypto.storage.annotation.EncryptField;
import io.github.ybb88.framework.desensitize.annotation.Desensitize;
import io.github.ybb88.framework.desensitize.DesensitizeType;
import io.github.ybb88.framework.mybatis.base.MybatisTenantEntity;
import io.github.ybb88.framework.mybatis.crypto.handler.EncryptTypeHandler;
import lombok.Data;
import lombok.EqualsAndHashCode;

import java.math.BigDecimal;

@Data
@EqualsAndHashCode(callSuper = true)
@TableName(value = "biz_order", autoResultMap = true)
public class OrderEntity extends MybatisTenantEntity {

    private String orderNo;
    private BigDecimal amount;
    private Long customerId;
    private Integer status;

    /**
     * 敏感字段：手机号
     * 1. 存储态: 国密 SM4 透明加密入库 (EncryptTypeHandler)
     * 2. 展示态: 返回前端自动掩码打码 (138****8000)
     * 3. 盲索引: 框架自动计算 HMAC-SM3 加盐哈希并注入 mobileHash 辅助列
     */
    @Desensitize(rule = DesensitizeType.MOBILE_PHONE)
    @EncryptField(blindIndexField = "mobileHash")
    @TableField(typeHandler = EncryptTypeHandler.class)
    private String mobile;

    /**
     * 手机号加盐哈希盲索引列（用于加密条件下的等值检索）
     */
    @BlindIndex(source = "mobile")
    private String mobileHash;
}
```

```java
package io.github.ybb88.sample.order.mapper;

import com.baomidou.mybatisplus.core.mapper.BaseMapper;
import io.github.ybb88.sample.order.entity.OrderEntity;
import org.apache.ibatis.annotations.Mapper;

@Mapper
public interface OrderMapper extends BaseMapper<OrderEntity> {
}
```

---

## 📡 5. 声明式 RPC 客户端与 Controller 实战

### 5.1 声明式 HTTP 客户端
无需过时的 Feign，直接使用原生 `@HttpExchange`，跨服务的租户 ID（`X-Tenant-Id`）与当前用户身份（`X-User-Id`）由框架全自动透传：

```java
package io.github.ybb88.sample.order.client;

import io.github.ybb88.framework.api.R;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.service.annotation.GetExchange;
import org.springframework.web.service.annotation.HttpExchange;

@HttpExchange(url = "http://system-service/api/system/user")
public interface RemoteUserClient {

    @GetExchange("/{userId}")
    R<Object> getUserById(@PathVariable("userId") Long userId);
}
```

### 5.2 Controller 实战（包含声明式分布式锁）
```java
package io.github.ybb88.sample.order.controller;

import io.github.ybb88.framework.api.R;
import io.github.ybb88.framework.util.crypto.BlindIndexUtil;
import io.github.ybb88.framework.redis.lock.annotation.DistributedLock;
import io.github.ybb88.sample.order.client.RemoteUserClient;
import io.github.ybb88.sample.order.entity.OrderEntity;
import io.github.ybb88.sample.order.mapper.OrderMapper;
import com.baomidou.mybatisplus.core.toolkit.Wrappers;
import lombok.RequiredArgsConstructor;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/v1/orders")
@RequiredArgsConstructor
public class OrderController {

    private final OrderMapper orderMapper;
    private final RemoteUserClient remoteUserClient;

    /**
     * 创建订单：
     * 使用 @DistributedLock 防重提单（支持 SpEL 表达式），加锁失败自动返回标准业务异常
     */
    @PostMapping
    @DistributedLock(key = "'order:' + #order.orderNo", waitTime = 3000, leaseTime = 10000, message = "正在创建订单，请勿重复提交")
    public R<OrderEntity> createOrder(@RequestBody OrderEntity order) {
        orderMapper.insert(order);
        return R.data(order);
    }

    /**
     * 密文等值精准检索：
     * 传入明文手机号，调用 BlindIndexUtil.generateIndex 针对 mobile_hash 辅助列检索
     */
    @GetMapping("/by-mobile")
    public R<OrderEntity> getOrderByMobile(@RequestParam String mobile) {
        String hash = BlindIndexUtil.generateIndex(mobile);
        OrderEntity order = orderMapper.selectOne(
            Wrappers.<OrderEntity>lambdaQuery().eq(OrderEntity::getMobileHash, hash)
        );
        return R.data(order); // 返回时 mobile 自动脱敏为 138****8000
    }

    /**
     * 跨服务 RPC 调用
     */
    @GetMapping("/user-info/{userId}")
    public R<Object> getUserInfo(@PathVariable Long userId) {
        return remoteUserClient.getUserById(userId);
    }
}
```

---

## 🧵 6. 异步线程池与批处理上下文传递

在 `@Async` 异步任务或多线程池中，必须使用 `FrameworkContext` 安全传递快照：

```java
@Service
@RequiredArgsConstructor
public class AsyncOrderService {

    private final Executor taskExecutor;

    public void processOrderAsync() {
        // 1. 在 Web 主线程抓取完整上下文快照 (租户、用户身份、权限票据)
        var snapshot = FrameworkContext.capture();

        // 2. 传递给异步线程池受控执行
        taskExecutor.execute(() -> {
            try (var scope = FrameworkContext.openScope(snapshot)) {
                // 在作用域内，TenantContext 与 SecurityContextHolder 自动完整还原
                // 执行持久层操作将自动受当前租户隔离保护
            } // 离开作用域时，自动彻底清理，杜绝线程池复用污染
        });
    }
}
```

---

## ⚙️ 7. 核心配置文件 (`application.yml`)

```yaml
server:
  port: 8080

spring:
  application:
    name: sample-order-service
  datasource:
    url: jdbc:mysql://127.0.0.1:3306/zhkj_order?useUnicode=true&characterEncoding=utf-8&serverTimezone=Asia/Shanghai
    username: root
    # 敏感配置加密存储，由 starter-crypto 启动期透明解密
    password: "ENC(XyKx5jN5684AxytPGmOTAV7+XSDh/S83BwCb8aCdyGU=)"

zhkj:
  # 1. 敏感配置解密主密钥 (生产环境推荐由环境变量注入)
  crypto:
    property:
      enabled: true
      password: "MasterKey@2026"

  # 2. 多租户全局开关与 MyBatis-Plus 行级隔离
  tenant:
    enabled: true
    mybatis:
      column: "tenant_id"
      ignore-tables:
        - "sys_dict"
        - "sys_region"

  # 3. 跨服务 RPC 请求头自动透传
  http:
    client:
      headers:
        enabled: true

  # 4. XSS 防护与白名单
  xss:
    enabled: true
    skip-url:
      - "/doc.html"
      - "/v3/api-docs/**"
```

启动应用后，控制台将输出精美 ASCII Banner 与环境概况，访问 `http://localhost:8080/doc.html` 即可进入在线接口文档中心。
