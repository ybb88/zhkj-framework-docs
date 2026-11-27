# 持久层核心组件：zhkj-framework-starter-mybatis

> **模块定位**: MyBatis-Plus 集成 Starter，提供审计实体基类、审计字段与盲索引自动填充、存储加密 `EncryptTypeHandler`、多租户与数据权限拦截器的挂载编排、分页插件及 SQL 日志。  
> **适用基准**: OpenJDK 21, Spring Boot 4.x, MyBatis-Plus（`mybatis-plus-spring-boot3-starter`）  
> **模块坐标**: `io.github.ybb88:zhkj-framework-starter-mybatis`

---

## 📋 1. 架构定位与核心设计

1. **实体基类**：`MybatisAuditEntity<ID>` 提供审计字段；`MybatisBaseEntity` 固定 `Long` 雪花主键；`MybatisTenantEntity` 增加 `tenantId`；逻辑删除与租户组合见第 3 节。
2. **自动填充（`ZhkjMetaObjectHandler`）**：插入与更新时填充审计字段，并调用 `BatchBlindIndexHelper.prefill` 计算盲索引辅助列（源字段到哈希字段的映射由 `BlindIndexBindingUtil` 解析）。
3. **存储加密类型处理器（`EncryptTypeHandler`）**：写入调用 `StorageCryptoEngine.encrypt`，读取调用 `decrypt`。静态方法 `isSm4Available()` 委托 `StorageCryptoEngine.isSm4Available()`，经 `CryptoAlgorithmProvider.isAvailable()` 探测 SM4 是否可用。引擎开关与密钥见 `starter-crypto`。
4. **插件链编排（`MybatisPlusConfiguration.mybatisPlusInterceptor`）**：`TenantLineInnerInterceptor` → `MybatisDataScopeInterceptor` → 其他自定义 `InnerInterceptor` → `ZhkjPaginationInterceptor`。租户与数据权限装配后会调整到上述位置（见第 5 节）。
5. **Mapper 扫描**：`MapperScannerConfigurer` 的扫描包由 `zhkj.base-packages` 决定（默认 `com.zhkj`），扫描模式为 `<base>.**.mapper`。

---

## 📦 2. 依赖坐标

```xml
<dependency>
    <groupId>io.github.ybb88</groupId>
    <artifactId>zhkj-framework-starter-mybatis</artifactId>
</dependency>
```

该模块直接依赖 `zhkj-framework-core`、`zhkj-framework-data-api`、`mybatis-plus-spring-boot3-starter`、`mybatis-plus-jsqlparser`、`mybatis-spring`、`spring-boot-starter-jdbc`。以下依赖为 `optional`：`zhkj-framework-starter-crypto`、`zhkj-framework-starter-datascope`、`zhkj-framework-starter-tenant`、`jakarta.validation-api`；`zhkj-framework-starter-secure` 仅为 `test` 作用域，不向下游传递。

---

## 🧱 3. 核心包结构与组件

```
io.github.ybb88.framework.mybatis
├── base/
│   ├── MybatisAuditEntity.java          # 审计字段基类（泛型主键）
│   ├── MybatisBaseEntity.java           # Long 雪花主键
│   ├── MybatisLogicEntity.java          # + deletedId（@TableLogic）
│   ├── MybatisTenantEntity.java         # + tenantId（TenantId，ITenantEntity）
│   ├── MybatisTenantLogicEntity.java    # 租户 + 逻辑删除
│   ├── BaseRepository.java              # 扩展 IService 的仓储接口
│   └── BaseRepositoryImpl.java
├── ZhkjMetaObjectHandler.java           # 审计与盲索引自动填充
├── config/MybatisPlusConfiguration.java # 扫描器、SqlSessionFactory、插件链、MetaObjectHandler
├── aot/MybatisRuntimeHints.java         # AOT 运行时提示
├── crypto/
│   ├── handler/EncryptTypeHandler.java  # 存储加解密 TypeHandler
│   └── util/BatchBlindIndexHelper.java  # 实体盲索引预填充
├── datascope/
│   ├── config/MybatisDataScopeAutoConfiguration.java
│   ├── handler/MybatisDataScopeHandler.java, ZhkjScopeModelHandler.java
│   └── interceptor/MybatisDataScopeInterceptor.java
├── intercept/QueryInterceptor.java      # 查询拦截 SPI（继承 Ordered）
├── plugins/
│   ├── ZhkjPaginationInterceptor.java   # 分页插件（支持 QueryInterceptor）
│   ├── QueryInterceptorExecutor.java
│   └── SqlLogInterceptor.java           # zhkj.mybatis-plus.sql-log=true 时注册
├── props/MybatisPlusProperties.java     # zhkj.mybatis-plus.*
├── support/                             # Condition, EntityWrapper, Query, SqlKeyword 等
├── tenant/
│   ├── MybatisTenantAutoConfiguration.java
│   ├── MybatisTenantHandler.java        # TenantLineHandler
│   ├── MybatisTenantInterceptor.java    # TenantLineInnerInterceptor
│   ├── MybatisTenantProperties.java     # zhkj.tenant.mybatis.*
│   └── TenantIdTypeHandler.java         # TenantId <-> BIGINT 类型处理器
└── util/PageUtil.java
```

自动装配（`META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`）：

- `MybatisPlusConfiguration`
- `MybatisDataScopeAutoConfiguration`（需 `zhkj.datascope.enabled=true`）
- `MybatisTenantAutoConfiguration`（需 `zhkj.tenant.enabled=true`）

---

## ⚙️ 4. 核心配置清单 (`application.yml`)

```yaml
mybatis-plus:
  mapper-locations: classpath*:mapper/**/*.xml
  type-aliases-package: com.example.**.entity
  configuration:
    map-underscore-to-camel-case: true      # 框架在 SqlSessionFactory 中强制设为 true
  global-config:
    db-config:
      id-type: assign_id

zhkj:
  base-packages: com.example                # 默认 com.zhkj，用于 Mapper 扫描
  mybatis-plus:
    page-limit: 500                         # 默认 500，分页上限
    overflow: false                         # 默认 false，页码溢出处理
    sql-log: false                          # 默认 false，true 时注册 SqlLogInterceptor
  tenant:
    enabled: true                           # 默认 false（需显式开启）
    field-isolation:                        # 全局默认租户列与表规则（见第 5 节）
      column: tenant_id
      tables: []
      ignore-tables: [sys_tenant, sys_dict]
      default-tenant-id: "10000001"
    mybatis:                                # 可选：显式字段覆盖全局默认，仅影响 MyBatis
      column: tenant_id
  datascope:
    enabled: true                           # 默认 false
```

`deleted_id` 逻辑删除语义：未删除为 `0`，删除时写入记录主键 ID。框架在 `MybatisPlusConfiguration` 中固定 `db-config.logic-delete-value=id`，删除生成 `SET deleted_id = id`；实体 `@TableLogic` 字段 `deletedId`（`Long`）对应列 `deleted_id BIGINT UNSIGNED DEFAULT 0`，无需也不应再配置 `logic-delete-value`。

---

## 🔐 5. 拦截器挂载与生效规则

### 5.1 插件链顺序

| 序号 | 拦截器 | 装配条件 |
|------|--------|----------|
| 1 | `MybatisTenantInterceptor`（`TenantLineInnerInterceptor`） | `zhkj.tenant.enabled=true` 且类路径存在 `TenantContext` |
| 2 | `MybatisDataScopeInterceptor` | `zhkj.datascope.enabled=true` |
| 3 | 其他 `InnerInterceptor` Bean | 任意 |
| 末 | `ZhkjPaginationInterceptor` | 始终 |

`MybatisTenantAutoConfiguration` 将租户拦截器移到列表首位。`MybatisDataScopeAutoConfiguration` 将数据权限拦截器插入到租户拦截器之后；无租户拦截器时插入到分页拦截器之前。

### 5.2 租户列与忽略表的实际来源

当 `zhkj.tenant.enabled=true` 且使用 `TenantCoreAutoConfiguration` 自动创建的 `DefaultTenantRuleProvider` 时，`MybatisTenantAutoConfiguration` 在启动阶段适配字段配置，再由同一个默认规则引擎实现裁决：

1. `zhkj.tenant.field-isolation.column`、`tables`、`ignore-tables` 提供全局默认值。
2. `zhkj.tenant.mybatis.column`、`tables`、`ignore-tables` **按字段显式覆盖**全局值；未配置的字段仍继承全局规则。显式配置为默认列名 `tenant_id` 也属于覆盖；表列表整体替换而非追加，显式空列表可清空继承的列表。
3. 适配使用 Spring Binder 的实际绑定结果，不通过属性默认值猜测是否配置。MyBatis 使用独立的默认规则提供者实例，不修改共享全局配置或其他 ORM 的规则。
4. 应用自定义的 `TenantRuleProvider` 始终优先（包括直接注册的 `DefaultTenantRuleProvider`、其子类或代理）；上述 MyBatis 配置不会覆盖自定义提供者的列名或表级裁决。

租户 ID 由规则提供者解析；处理器在调用自定义规则提供者前，也会对委托目标或票据直接拒绝，不会因为 `grantTicket` 切换任意 SQL 的租户。其他情况下按主体 `TenantContext`、`FrameworkContext` 快照解析。默认提供者在完全无上下文时返回框架默认租户 ID；`default-tenant-id` 仅用于处理器未获得提供者 ID 时的兜底，不覆盖提供者已经返回的 ID。

若裁剪自动配置、未装配任何 `TenantRuleProvider`，处理器保留使用 `zhkj.tenant.mybatis.*` 的内建路径，此时不继承全局字段规则。

纯 MyBatis、批处理等非 Web 应用无需补充 Spring MVC 或 Servlet 依赖：`TenantCoreAutoConfiguration` 的注册表、上下文与规则提供者独立装配；Header/Subdomain 解析器及 Web 拦截器仅在 Servlet Web 应用且相关类存在时装配。

### 5.3 无租户上下文时的行为

`MybatisTenantHandler.ignoreTable` 与 `DefaultTenantRuleProvider.ignoreTable` 在完全无租户身份上下文时仍返回 `true`，即该 SQL 不附加租户条件，本次配置优先级适配不改变此全局策略。若仅缺少 `TenantContext`、但安全上下文或 `FrameworkContext` 快照仍有租户身份，默认提供者会自愈绑定并继续按有效表规则隔离。调用方在批处理、异步任务中须自行绑定租户（见 `starter-tenant`）。

未装配 `TenantRuleProvider` 时，`MybatisTenantHandler` 自身执行相同的自愈：`hasTenantContext()` 从 `FrameworkContext` 快照取得租户后调用 `autoSyncTenantContext` 补齐 `TenantContext`。

### 5.4 数据权限

`MybatisDataScopeInterceptor` 仅处理满足以下任一条件的 Mapper 方法：

- 方法或所在 Mapper 接口标注 `@DataAuth`（`enabled=false` 时跳过）；或
- 方法名包含 `zhkj.datascope.mapper-key` 中的关键字（默认 `page`、`Page`、`list`、`List`、`update`、`Update`、`delete`、`Delete`），且 Mapper 名不包含 `zhkj.datascope.mapper-exclude` 中的字符串（默认 `FlowMapper`）。

需要受控但不满足上述规则的查询，应显式标注 `@DataAuth`。

---

## 💡 6. 典型使用范式

### 6.1 实体（租户 + 审计 + 存储加密 + 盲索引）

```java
package io.github.ybb88.demo.entity;

import com.baomidou.mybatisplus.annotation.TableField;
import com.baomidou.mybatisplus.annotation.TableName;
import io.github.ybb88.framework.crypto.storage.annotation.BlindIndex;
import io.github.ybb88.framework.crypto.storage.annotation.EncryptField;
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

    @EncryptField(blindIndexField = "mobileHash")
    @TableField(typeHandler = EncryptTypeHandler.class)
    private String mobile;

    @BlindIndex(source = "mobile")
    private String mobileHash;
}
```

`EncryptTypeHandler` 仅在 `zhkj.storage.crypto.enabled=true` 时加密；盲索引需配置 `zhkj.storage.crypto.blind-index-salt`。

### 6.2 Mapper

```java
@Mapper
public interface OrderMapper extends BaseMapper<OrderEntity> {
}
```

### 6.3 数据权限

```java
@Mapper
public interface OrderMapper extends BaseMapper<OrderEntity> {

    @DataAuth(code = "order", column = "dept_id", type = DataScopeEnum.OWN_DEPT)
    List<OrderEntity> selectOrderList(@Param("query") OrderQuery query);
}
```

`DataAuth` 位于 `io.github.ybb88.framework.datascope.annotation`，`DataScopeEnum` 位于 `io.github.ybb88.framework.datascope.enums`，详见 `starter-datascope`。

### 6.4 盲索引等值检索

```java
String hash = StorageBlindIndexUtil.generateIndex(plainMobile);   // io.github.ybb88.framework.crypto.storage.util.StorageBlindIndexUtil
orderMapper.selectList(Wrappers.<OrderEntity>lambdaQuery().eq(OrderEntity::getMobileHash, hash));
```

盲索引仅支持等值（`EQ` / `IN`），不得使用 `LIKE`。

---

## 🧪 7. 审计与盲索引填充细节（`ZhkjMetaObjectHandler`）

| 字段（插入） | 填充规则 |
|--------------|----------|
| `createBy` / `createdBy`（Long） | `AuditUserProvider.getUserId()`，为空时 `1L` |
| `createBy` / `createdBy`（String） | `AuditUserProvider.getAccount()`，为空时 `system` |
| `updateBy` / `updatedBy` | 同上 |
| `createTime` / `createdTime`、`updateTime` / `updatedTime` | 当前时间（`Date` 或 `LocalDateTime`） |

更新操作仅填充 `updateBy`、`updatedBy`、`updateTime`、`updatedTime`。

盲索引在插入与更新时通过 `BatchBlindIndexHelper.prefill` 计算：对每个来源字段（明文）生成哈希并写入目标字段。来源值以 `sm4:` 或 `v1:` 开头时跳过。计算失败（例如盐未配置）仅记录 WARN 日志，目标字段保持原值。

---

## 📎 8. 相关文档

- `starter-crypto`：加密引擎、盲索引算法与配置
- `starter-datascope`：数据权限注解、SPI 与 SQL 改写规则
- `starter-tenant`：租户上下文与解析器
