# ZHKJ Framework 快速上手指南

> **目标**: 搭建一个具备多租户隔离、审计填充、国密加密与盲索引检索、响应脱敏、分布式锁、RPC 调用与在线文档的标准微服务应用。
> **适用版本**: OpenJDK 21 (LTS), Spring Boot 4.1.1, Spring Cloud 2025.1.2（以根 `pom.xml` 为准）

---

## 🛠️ 1. 环境准备

- **JDK**: OpenJDK 21 (LTS)
- **Maven**: 3.9+（`build.ps1` 使用 3.9.10）
- **IDE**: IntelliJ IDEA 或 VSCode（需开启 Annotation Processing，Lombok 必需）
- **框架构件**: 业务工程依赖的 `io.github.ybb88:*` 构件需先在本地仓库安装（在框架根目录执行 `mvn install "-DskipTests=true" "-Dgpg.skip=true"`）。`-Dgpg.skip=true` 仅对根 POM `release` profile 中绑定的 `maven-gpg-plugin` 生效；默认 `develop` profile（`activeByDefault`）不绑定 GPG 插件，该参数在默认构建中无实际作用，保留以与 CI 命令一致。
- **交付校验**: 在框架根目录执行 `mvn clean verify "-DskipTests=false" "-Dgpg.skip=true" -B`（与 `.github/workflows/ci.yml` 中命令一致）。`-DskipTests=false` 覆盖根 POM 默认 `<skipTests>true</skipTests>` 属性，完成测试与打包，不执行 `deploy`；`verify` 不安装构件，业务工程联调前仍需执行上述 `install`。
- **依赖管理边界**: 已移除框架与 NCAP POM 均未消费、且版本属性未定义的 Batik（`batik-all` / `batik-svgpp`）、XML Graphics Commons、XML Security、XXL-Job 与 P6Spy 管理项，不补猜版本；实际引入时应先确定版本并统一管理。

---

## 📦 2. 创建微服务与引入起步依赖

在业务微服务（例如 `sample-order-service`）的 `pom.xml` 中引入框架 Starter：

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

> 依赖关系以各模块 `pom.xml` 为准，见 `docs/architecture/overview.md` 第 3.11 节。`starter-mybatis` 对 `starter-crypto`、`starter-tenant`、`starter-datascope` 为 `optional`，需要时需显式引入。

---

## 🚀 3. 编写主启动类

使用 `ZhkjApplication.run(appName, Class, args)` 启动（`zhkj-framework-launcher` 中的静态方法，非注解）：

```java
package io.github.ybb88.sample.order;

import io.github.ybb88.framework.http.client.annotation.EnableHttpClients;
import io.github.ybb88.framework.launcher.launch.ZhkjApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@EnableHttpClients(basePackages = "io.github.ybb88.sample.order.client")
@SpringBootApplication
public class SampleOrderApplication {

    public static void main(String[] args) {
        ZhkjApplication.run("sample-order-service", SampleOrderApplication.class, args);
    }
}
```

---

## 🗄️ 4. 实体模型：多租户隔离、国密加密入库与响应打码

实体继承 `MybatisTenantEntity`，租户字段与审计字段由框架填充；敏感字段通过注解声明加密与盲索引：

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
     * 1. 存储态: 透明加密入库 (EncryptTypeHandler)
     * 2. 展示态: 响应序列化时掩码 (138****8000)
     * 3. 盲索引: 自动计算加盐哈希并注入 mobileHash 辅助列
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
使用 Spring 6+ 原生 `@HttpExchange`。跨服务请求头（如 `X-Tenant-Id`、`X-User-Id`）由 `HttpClientHeaderInterceptor` 透传，开关为 `zhkj.http.client.headers.enabled`（默认 `true`）：

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

import com.baomidou.mybatisplus.core.toolkit.Wrappers;
import io.github.ybb88.framework.api.R;
import io.github.ybb88.framework.redis.lock.annotation.DistributedLock;
import io.github.ybb88.framework.util.crypto.BlindIndexUtil;
import io.github.ybb88.sample.order.client.RemoteUserClient;
import io.github.ybb88.sample.order.entity.OrderEntity;
import io.github.ybb88.sample.order.mapper.OrderMapper;
import lombok.RequiredArgsConstructor;
import org.springframework.web.bind.annotation.*;

import java.util.concurrent.TimeUnit;

@RestController
@RequestMapping("/api/v1/orders")
@RequiredArgsConstructor
public class OrderController {

    private final OrderMapper orderMapper;
    private final RemoteUserClient remoteUserClient;

    /**
     * 创建订单：
     * @DistributedLock 防重提单（key 支持 SpEL）。waitTime / leaseTime 的单位由 timeUnit 决定，默认秒。
     */
    @PostMapping
    @DistributedLock(key = "'order:' + #order.orderNo", waitTime = 3000, leaseTime = 10000,
            timeUnit = TimeUnit.MILLISECONDS, message = "正在创建订单，请勿重复提交")
    public R<OrderEntity> createOrder(@RequestBody OrderEntity order) {
        orderMapper.insert(order);
        return R.data(order);
    }

    /**
     * 密文等值精准检索：
     * 传入明文手机号，计算盲索引后针对 mobileHash 辅助列等值检索
     */
    @GetMapping("/by-mobile")
    public R<OrderEntity> getOrderByMobile(@RequestParam String mobile) {
        String hash = BlindIndexUtil.generateIndex(mobile);
        OrderEntity order = orderMapper.selectOne(
            Wrappers.<OrderEntity>lambdaQuery().eq(OrderEntity::getMobileHash, hash)
        );
        return R.data(order); // 序列化时 mobile 自动脱敏为 138****8000
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

在 `@Async` 异步任务或自定义线程池中，使用 `FrameworkContext` 传递快照。`starter-web` 提供名为 `taskExecutor` 的 `Executor` Bean（`ZhkjExecutorConfiguration`）：

```java
@Service
@RequiredArgsConstructor
public class AsyncOrderService {

    private final Executor taskExecutor;

    public void processOrderAsync() {
        // 1. 在请求线程中抓取上下文快照（租户、用户身份、数据权限等）
        ContextSnapshot snapshot = FrameworkContext.capture();

        // 2. 在异步线程中受控执行
        taskExecutor.execute(() -> {
            try (ContextScope scope = FrameworkContext.openScope(snapshot)) {
                // 作用域内 TenantContext 与 SecurityContextHolder 已还原，
                // 持久层操作仍受当前租户隔离约束
            } // 离开作用域时自动清理，避免线程池复用污染
        });
    }
}
```

`ContextSnapshot` 与 `ContextScope` 位于 `io.github.ybb88.framework.context` 包。

---

## ⚙️ 7. 核心配置文件 (`application.yml`)

敏感值统一通过环境变量注入，示例中不写明文密钥：

```yaml
server:
  port: 8080

spring:
  application:
    name: sample-order-service
  datasource:
    url: jdbc:mysql://127.0.0.1:3306/zhkj_order?useUnicode=true&characterEncoding=utf-8&serverTimezone=Asia/Shanghai
    username: root
    # 数据库密码以 ENC(...) 密文存储，由 starter-crypto 启动期透明解密
    password: "ENC(XyKx5jN5684AxytPGmOTAV7+XSDh/S83BwCb8aCdyGU=)"

zhkj:
  # 1. 配置属性解密主密钥：此处以占位符 ${ZHKJ_CRYPTO_PROPERTY_PASSWORD} 注入；系统属性与环境变量 ZHKJ_CRYPTO_PASSWORD 优先，解析顺序见 migration.md 第 3.1 节
  crypto:
    property:
      enabled: true
      password: "${ZHKJ_CRYPTO_PROPERTY_PASSWORD}"

  # 2. 存储加密（EncryptTypeHandler 与盲索引依赖此配置；未配置 key 时加密阻断写入）
  storage:
    crypto:
      enabled: true
      algorithm: AES
      key: "${ZHKJ_STORAGE_CRYPTO_KEY}"
      version: v1
      blind-index-salt: "${ZHKJ_STORAGE_CRYPTO_BLIND_INDEX_SALT}"

  # 3. 多租户：全局开关与 MyBatis 行级隔离
  tenant:
    enabled: true
    mybatis:
      column: tenant_id
      ignore-tables:
        - sys_dict
        - sys_region

  # 4. 跨服务 RPC 请求头自动透传
  http:
    client:
      headers:
        enabled: true

  # 5. XSS 防护与放行路径
  xss:
    enabled: true
    skip-url:
      - /doc.html
      - /v3/api-docs/**
```

启动应用后，控制台输出 ASCII Banner（`zhkj-framework-launcher/src/main/resources/banner.txt`）。在线接口文档由 `zhkj-framework-starter-openapi` 提供（springdoc-openapi + Knife4j）；`/doc.html` 为 Knife4j 默认入口，框架源码中未显式配置该路径 `[待确认]`，以实际运行为准。
