# 多数据源路由组件：zhkj-framework-starter-datasource

> **模块定位**: 企业级动态多数据源路由 Starter，提供读写分离、按库隔离、基于 `@DS` 注解与 SpEL 表达式的动态路由机制，以及高性能 HikariCP 连接池纳管。  
> **适用基准**: OpenJDK 21 (LTS), Spring Boot 4.1.1, HikariCP  
> **模块坐标**: `io.github.ybb88:zhkj-framework-starter-datasource`

---

## 📋 1. 架构定位与核心设计

在企业复杂微服务架构中，单服务常需要连接多个物理数据库（如主从读写分离、历史冷库与在线热库分离、不同租户物理分库）：

1. **`DynamicRoutingDataSource` 动态数据源代理**：基于 Spring 原生 `AbstractRoutingDataSource` 深度定制，内部维护数据源池容器；
2. **声明式 `@DS` 注解与 SpEL 动态路由**：支持在类或方法上标注 `@DS("slave")` 或 `@DS("#tenantId")`，支持在运行时根据入参动态选择数据源；
3. **数据源优雅回退机制**：未显式指定或找不到目标数据源时，自动回退到 `master` 主数据源，保障服务高可用。

---

## 📦 2. 依赖坐标

```xml
<dependency>
    <groupId>io.github.ybb88</groupId>
    <artifactId>zhkj-framework-starter-datasource</artifactId>
</dependency>
```

---

## ⚙️ 3. 核心配置清单 (`application.yml`)

```yaml
spring:
  datasource:
    dynamic:
      primary: master               # 默认主数据源别名
      strict: false                 # 严格模式：true 时找不到数据源直接抛异常
      datasource:
        # 主数据源 (写库)
        master:
          url: jdbc:mysql://127.0.0.1:3306/db_master?useUnicode=true&characterEncoding=utf-8
          username: root
          password: "ENC(encrypted_master_pwd)"
          driver-class-name: com.mysql.cj.jdbc.Driver
          hikari:
            maximum-pool-size: 20
            minimum-idle: 5
        # 从数据源 (只读库)
        slave:
          url: jdbc:mysql://127.0.0.1:3306/db_slave?useUnicode=true&characterEncoding=utf-8
          username: root
          password: "ENC(encrypted_slave_pwd)"
          driver-class-name: com.mysql.cj.jdbc.Driver
          hikari:
            maximum-pool-size: 30
            minimum-idle: 10
```

---

## 💡 4. 典型使用范式

### 4.1 方法级别切换从库
```java
package io.github.ybb88.demo.service;

import io.github.ybb88.framework.datasource.annotation.DS;
import io.github.ybb88.demo.entity.OrderEntity;
import io.github.ybb88.demo.mapper.OrderMapper;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;

import java.util.List;

@Service
@RequiredArgsConstructor
public class OrderQueryService {

    private final OrderMapper orderMapper;

    /**
     * 读取从库数据，减轻主库查询压力
     */
    @DS("slave")
    public List<OrderEntity> listRecentOrders() {
        return orderMapper.selectList(null);
    }

    /**
     * 默认走 master 主库
     */
    public void createOrder(OrderEntity order) {
        orderMapper.insert(order);
    }
}
```

### 4.2 基于 SpEL 表达式动态路由
```java
@Service
public class TenantArchiveService {

    /**
     * 根据方法入参动态路由到对应数据源池 (例如 "ds_tenant_01")
     */
    @DS("'ds_' + #tenantCode")
    public void archiveData(String tenantCode) {
        // 数据操作将路由至目标租户独立数据源
    }
}
```
