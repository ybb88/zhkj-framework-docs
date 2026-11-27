# 多数据源路由组件：zhkj-framework-starter-datasource

> **模块定位**: 对 Baomidou `dynamic-datasource-spring-boot3-starter` 的增强，提供编程式数据源增删辅助、`@DS` 事务内切换告警，以及 ShardingSphere-JDBC 数据源向动态路由的挂载。  
> **适用基准**: OpenJDK 21 (LTS), Spring Boot 4.1.1, dynamic-datasource 4.3.1（根 pom `dynamic.version`），ShardingSphere-JDBC 5.4.1（根 pom `shardingsphere.version`，可选）  
> **模块坐标**: `io.github.ybb88:zhkj-framework-starter-datasource`

---

## 📋 1. 架构定位与职责

本模块不实现数据源路由本身。路由、连接池与 `@DS` 注解均由 Baomidou 提供，本模块提供以下增强:

| 能力 | 类 | 说明 |
| :--- | :--- | :--- |
| 编程式数据源管理 | `DynamicDataSourceHelper` | 查询、添加、移除路由中的数据源 |
| 事务内切换告警 | `DynamicDataSourceTransactionGuardAspect` | 活跃 Spring 事务中 `@DS` 切换到不同数据源时输出 `WARN` 日志 |
| ShardingSphere 挂载 | `ShardingSphereDynamicRoutingSupport` | 将容器中的 `ShardingSphereDataSource` 以键 `sharding` 挂入路由 |

依赖（来自 `pom.xml`）: `zhkj-framework-core`、`zhkj-framework-data-api`、`spring-boot-starter-jdbc`、`spring-boot-starter-aspectj`、`spring-boot-autoconfigure`、`com.baomidou:dynamic-datasource-spring-boot3-starter`、`com.zaxxer:HikariCP`；`shardingsphere-jdbc-core` 为 optional。

---

## 🧱 3. 包结构

```
io.github.ybb88.framework.datasource
├── config/ZhkjDynamicDataSourceAutoConfiguration.java
├── guard/DynamicDataSourceTransactionGuardAspect.java
├── shardingsphere/ShardingSphereDynamicRoutingSupport.java
└── toolkit/DynamicDataSourceHelper.java

org.apache.shardingsphere.infra.util.yaml.representer.ShardingSphereYamlRepresenter
    # @Deprecated(since = "2026.4.1.0", forRemoval = true)，供 ShardingSphere YAML 序列化使用
```

自动装配（`META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`）注册:

- `io.github.ybb88.framework.datasource.config.ZhkjDynamicDataSourceAutoConfiguration`
- `io.github.ybb88.framework.datasource.shardingsphere.ShardingSphereDynamicRoutingSupport`

---

## ⚙️ 4. 配置

路由配置使用 Baomidou 的 `spring.datasource.dynamic.*` 命名空间。以下为本模块读取或影响的键:

| 键 | 默认值 | 说明 |
| :--- | :--- | :--- |
| `spring.datasource.dynamic.guard.enabled` | `true` | 是否装配事务内切换告警切面（`matchIfMissing = true`） |

Baomidou 标准键（由该库解析，本模块不解析）:

| 键 | 说明 |
| :--- | :--- |
| `spring.datasource.dynamic.primary` | 默认数据源名称 |
| `spring.datasource.dynamic.strict` | 严格模式，未找到数据源时是否抛出异常 |
| `spring.datasource.dynamic.datasource.<name>.*` | 各数据源的 `url`、`username`、`password`、`driver-class-name`（见 `DynamicDataSourceTest`）；`hikari` 参数为第三方键 `[待确认]`，本仓库未见示例 |

```yaml
spring:
  datasource:
    dynamic:
      primary: master
      strict: false
      datasource:
        master:
          url: jdbc:mysql://127.0.0.1:3306/db_master?useUnicode=true&characterEncoding=utf-8
          username: root
          password: ENC(encrypted_master_pwd)
          driver-class-name: com.mysql.cj.jdbc.Driver
          hikari:
            maximum-pool-size: 20
            minimum-idle: 5
        slave:
          url: jdbc:mysql://127.0.0.1:3306/db_slave?useUnicode=true&characterEncoding=utf-8
          username: root
          password: ENC(encrypted_slave_pwd)
          driver-class-name: com.mysql.cj.jdbc.Driver
          hikari:
            maximum-pool-size: 30
            minimum-idle: 10
```

`ENC(...)` 的解密由 `zhkj-framework-starter-crypto` 的 `EncryptablePropertyEnvironmentPostProcessor` 负责（属性前缀 `zhkj.crypto.property.*`）。

---

## 🔀 5. 数据源切换（`@DS`）

`@DS` 注解为 Baomidou 提供的 `com.baomidou.dynamic.datasource.annotation.DS`，本模块不定义同名注解。

```java
import com.baomidou.dynamic.datasource.annotation.DS;

@Service
public class OrderQueryService {

    @DS("slave")
    public List<OrderEntity> listRecentOrders() {
        return orderMapper.selectList(null);
    }

    public void createOrder(OrderEntity order) {
        orderMapper.insert(order);
    }
}
```

SpEL 动态数据源名称由 Baomidou 解析，例如 `@DS("#tenantCode")`，具体语法以 Baomidou 文档为准（第三方行为，`[待确认]`：本仓库未包含该用例）。

### 5.1 事务内切换告警

`DynamicDataSourceTransactionGuardAspect`（`@Order(Ordered.HIGHEST_PRECEDENCE + 1)`）在方法或类标注 `@DS` 时检查:

- `TransactionSynchronizationManager.isActualTransactionActive()` 为真，且
- 目标数据源与当前数据源（`DynamicDataSourceContextHolder.peek()`）不同。

满足时输出 `WARN` 日志，提示 Spring 本地事务已绑定连接，切换不会生效。该切面只告警，不阻断调用。

---

## 🧰 6. 编程式管理（`DynamicDataSourceHelper`）

`ZhkjDynamicDataSourceAutoConfiguration` 注册 `zhkjDynamicDataSourceHelper`（缺失时）。

| 方法 | 说明 |
| :--- | :--- |
| `DynamicRoutingDataSource getRoutingDataSource()` | 获取路由数据源，不存在时返回 `null` |
| `boolean containsDataSource(String dsKey)` | 是否已注册；路由不存在或 `dsKey` 为 `null` 时返回 `false` |
| `void addDataSource(String dsKey, DataSource dataSource)` | 注册数据源实例；路由不存在时仅输出 WARN 日志，不抛异常 |
| `void removeDataSource(String dsKey)` | 从路由中移除；路由不存在时无操作 |
| `Set<String> getDataSourceKeys()` | 所有已注册键 |
| `DataSource getDataSource(String dsKey)` | 按键获取，不存在时返回 `null` |

```java
@Service
@RequiredArgsConstructor
public class TenantDataSourceService {

    private final DynamicDataSourceHelper helper;

    public void attach(String tenantKey, DataSource dataSource) {
        if (!helper.containsDataSource(tenantKey)) {
            helper.addDataSource(tenantKey, dataSource);
        }
    }

    public void detach(String tenantKey) {
        helper.removeDataSource(tenantKey);
    }
}
```

`DataSource` 实例由业务侧构造（例如使用 Baomidou 提供的数据源创建能力或 HikariCP）。本模块不提供基于配置对象的数据源创建方法。

---

## 🔗 7. ShardingSphere 挂载（`ShardingSphereDynamicRoutingSupport`）

- 条件: 类路径存在 `DynamicRoutingDataSource` 与 `ShardingSphereDataSource`（即 `shardingsphere-jdbc-core` 已引入）。
- 时机: 所有单例初始化完成后（`SmartInitializingSingleton`）。
- 行为: 若路由中不存在键 `sharding`，且容器中存在 `ShardingSphereDataSource` 类型的 `DataSource` Bean，则以键 `sharding` 注册到路由。

使用示例:

```java
@DS("sharding")
public List<OrderEntity> listByUser(Long userId) {
    return orderMapper.selectByUserId(userId);
}
```

---

## 🔗 8. 相关文档

- 持久层实体与审计: [data-api](./data-api.md)、[starter-jpa](./starter-jpa.md)
- 配置加密: `zhkj-framework-starter-crypto`
