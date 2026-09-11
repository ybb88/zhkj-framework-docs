# ZHKJ Framework 快速上手指南

> **目标**: 5 分钟内使用 `zhkj-framework` 搭建一个具备多租户隔离、审计填充、RPC 调用与在线文档的标准业务微服务应用。

---

## 🛠️ 1. 环境准备

- **JDK**: OpenJDK 21 (LTS)
- **Maven**: 3.8.1+ 或 3.9.x
- **IDE**: IntelliJ IDEA 2024+ 或 VSCode (启用 Annotation Processors)

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
        <!-- 1. 应用引导启动器 -->
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

        <!-- 4. MyBatis-Plus 持久层与多租户自动隔离 -->
        <dependency>
            <groupId>io.github.ybb88</groupId>
            <artifactId>zhkj-framework-starter-mybatis</artifactId>
        </dependency>
        <dependency>
            <groupId>io.github.ybb88</groupId>
            <artifactId>zhkj-framework-starter-tenant</artifactId>
        </dependency>

        <!-- 5. 注册与配置中心 (Nacos) -->
        <dependency>
            <groupId>io.github.ybb88</groupId>
            <artifactId>zhkj-framework-starter-nacos</artifactId>
        </dependency>

        <!-- 6. 在线接口文档 (Knife4j / OpenAPI 3) -->
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

使用 `ZhkjApplication.run(...)` 驱动应用启动，框架会自动探测操作系统环境、多环境 Profile 与启动耗时：

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

## 🗄️ 4. 编写实体类与 Mapper

继承专用的 `MybatisTenantEntity`，审计字段（`createTime`、`createBy`、`updateTime`、`updateBy`）与多租户隔离字段（`tenantId`）由框架全自动填充与行级拦截：

```java
package io.github.ybb88.sample.order.entity;

import com.baomidou.mybatisplus.annotation.TableName;
import io.github.ybb88.framework.mybatis.base.MybatisTenantEntity;
import lombok.Data;
import lombok.EqualsAndHashCode;
import java.math.BigDecimal;

@Data
@EqualsAndHashCode(callSuper = true)
@TableName("biz_order")
public class OrderEntity extends MybatisTenantEntity {

    private String orderNo;
    private BigDecimal amount;
    private Long customerId;
    private Integer status;
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

## 📡 5. 跨服务 RPC 调用与 Controller

使用现代 Spring 6+ HTTP Interfaces 声明远程微服务调用接口：

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

编写 Controller，返回值直接返回统一响应载体 `R<T>`：

```java
package io.github.ybb88.sample.order.controller;

import io.github.ybb88.framework.api.R;
import io.github.ybb88.sample.order.client.RemoteUserClient;
import io.github.ybb88.sample.order.entity.OrderEntity;
import io.github.ybb88.sample.order.mapper.OrderMapper;
import lombok.RequiredArgsConstructor;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/v1/orders")
@RequiredArgsConstructor
public class OrderController {

    private final OrderMapper orderMapper;
    private final RemoteUserClient remoteUserClient;

    @PostMapping
    public R<OrderEntity> createOrder(@RequestBody OrderEntity order) {
        orderMapper.insert(order);
        return R.data(order);
    }

    @GetMapping("/{id}")
    public R<OrderEntity> getOrder(@PathVariable Long id) {
        OrderEntity entity = orderMapper.selectById(id);
        return R.data(entity);
    }
}
```

---

## ⚙️ 6. 基础配置文件 (`application.yml`)

```yaml
server:
  port: 8080

spring:
  datasource:
    url: jdbc:mysql://127.0.0.1:3306/zhkj_order?useUnicode=true&characterEncoding=utf-8&serverTimezone=Asia/Shanghai
    username: root
    password: "123456"

zhkj:
  tenant:
    enabled: true
    column: "tenant_id"
    ignore-tables:
      - "sys_dict"
  http:
    client:
      headers:
        enabled: true # 跨服务自动透传租户与安全上下文
```
启动应用后，访问 `http://localhost:8080/doc.html` 即可查阅 Knife4j 在线 API 文档。
