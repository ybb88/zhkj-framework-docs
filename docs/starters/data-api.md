# 数据持久化抽象层：zhkj-framework-data-api

> **模块定位**: 与 ORM 无关的实体契约与 SPI 抽象层，只定义接口与 POJO 实现，不包含仓储、分页查询或 ORM 注解。  
> **适用基准**: OpenJDK 21 (LTS), Spring Boot 4.1.1  
> **模块坐标**: `io.github.ybb88:zhkj-framework-data-api`

---

## 📋 1. 架构定位与职责

`zhkj-framework-data-api` 位于 `zhkj-framework-core` 之上，被各持久层 Starter 引用：

- **框架无关**: 不依赖 MyBatis-Plus、JPA 或 Hibernate。
- **正交契约**: 主键、审计、租户、逻辑删除、乐观锁各自是独立接口，实体按需组合，不存在单一基类继承链。
- **SPI**: `AuditUserProvider` 提供当前操作人信息，由业务侧实现，持久层通过 `ObjectProvider` 获取。
- **无自动装配**: 本模块不包含 `AutoConfiguration.imports`，也不定义 `@ConfigurationProperties`。

```
zhkj-framework-core
        ▲
        │
zhkj-framework-data-api   (契约 + POJO 实现 + AuditUserProvider SPI)
        ▲
        ├── zhkj-framework-starter-jpa        (JpaBaseEntity / JpaTenantEntity)
        ├── zhkj-framework-starter-mybatis    (MybatisBaseEntity / MybatisTenantEntity)
        └── zhkj-framework-starter-datasource (依赖，无实体)
```

---

## 📦 2. 依赖坐标

```xml
<dependency>
    <groupId>io.github.ybb88</groupId>
    <artifactId>zhkj-framework-data-api</artifactId>
</dependency>
```

依赖关系（来自 `pom.xml`）:

| 依赖 | 范围 | 说明 |
| :--- | :--- | :--- |
| `zhkj-framework-core` | compile | 提供 `BaseModel`、`RoleUtil` |
| `lombok` | provided | 编译期 |
| `spring-data-commons` | optional | 仅供 `TypeInformation` 兼容垫片编译使用 |

---

## 🧱 3. 包结构

```
io.github.ybb88.framework.data
├── model/
│   ├── IEntity.java              # 主键根契约
│   ├── IAuditEntity.java         # 审计契约 (继承 IEntity)
│   ├── ITenantEntity.java        # 租户标记契约
│   ├── ILogicDeleteEntity.java   # 逻辑删除标记契约
│   ├── IVersionEntity.java       # 乐观锁版本契约
│   ├── PojoBaseEntity.java       # 纯 POJO 审计实体 (Long 主键)
│   └── PojoTenantEntity.java     # 纯 POJO 多租户实体
└── spi/
    └── AuditUserProvider.java    # 当前操作人 SPI

org.springframework.data.util.TypeInformation   # 兼容垫片 (@Deprecated(forRemoval = true))
```

---

## 🧩 4. 契约接口

| 接口 | 泛型 | 继承 | 方法 |
| :--- | :--- | :--- | :--- |
| `IEntity<ID extends Serializable>` | ID | `Serializable` | `ID getId()`、`void setId(ID id)` |
| `IAuditEntity<ID extends Serializable>` | ID | `IEntity<ID>` | `Long getCreateBy()`、`void setCreateBy(Long)`、`LocalDateTime getCreateTime()`、`void setCreateTime(LocalDateTime)`、`Long getUpdateBy()`、`void setUpdateBy(Long)`、`LocalDateTime getUpdateTime()`、`void setUpdateTime(LocalDateTime)` |
| `ITenantEntity` | — | `Serializable` | `String getTenantId()`、`void setTenantId(String)` |
| `ILogicDeleteEntity<T>` | T | `Serializable` | `T getDeleted()`、`void setDeleted(T)` |
| `IVersionEntity<T>` | T | `Serializable` | `T getVersion()`、`void setVersion(T)` |

> 注意：`IAuditEntity` 的审计人字段为 `createBy` / `updateBy`（`Long` 类型），与泛型 `ID` 无关。逻辑删除字段访问器为 `getDeleted()` / `setDeleted()`。

---

## 🧰 5. POJO 实现

### 5.1 `PojoBaseEntity`

`public abstract class PojoBaseEntity extends BaseModel implements IAuditEntity<Long>, ILogicDeleteEntity<Integer>, IVersionEntity<Integer>, Serializable`（类注解 `@Data`、`@EqualsAndHashCode(callSuper = true)`）

| 字段 | 类型 | 默认值 | 说明 |
| :--- | :--- | :--- | :--- |
| `id` | `Long` | — | 主键 |
| `createBy` / `updateBy` | `Long` | — | 操作人 ID |
| `createTime` / `updateTime` | `LocalDateTime` | — | 审计时间 |
| `deleted` | `Integer` | `0` | 逻辑删除标识（0 正常，1 已删除） |
| `version` | `Integer` | `1` | 乐观锁版本号 |

### 5.2 `PojoTenantEntity`

`public abstract class PojoTenantEntity extends PojoBaseEntity implements ITenantEntity`，新增 `String tenantId`。

> 两个类均为 `abstract`，需由业务实体继承。

---

## 🔐 6. SPI：`AuditUserProvider`

所有方法均为 `default` 实现，业务侧按需覆盖：

| 方法 | 返回值 | 默认实现 |
| :--- | :--- | :--- |
| `Long getUserId()` | 当前操作人 ID | `null` |
| `String getAccount()` | 账号 | `null` |
| `String getRoleName()` | 角色名称 | `null` |
| `String getDeptId()` | 部门 ID（字符串） | `null` |
| `String getRoleId()` | 角色 ID（字符串） | `null` |
| `Map<String, Object> toUserMap()` | 键值对（`userId`、`account`、`roleName`、`roleId`、`deptId`） | 仅包含非 `null` 项 |
| `boolean isAdmin()` | 是否具备管理员特权 | `RoleUtil.isPrivileged(getRoleName())` |

`starter-jpa` 的 `AuditUserAuditorAware` 通过 `ObjectProvider<AuditUserProvider>` 读取 `getUserId()`，用于 `@CreatedBy` / `@LastModifiedBy`。`starter-mybatis` 的 `ZhkjMetaObjectHandler` 同样通过 `ObjectProvider<AuditUserProvider>` 注入（由 `MybatisPlusConfiguration` 装配）。

---

## ⚠️ 7. 兼容垫片

`org.springframework.data.util.TypeInformation` 位于本模块源码中（`src/main/java/org/springframework/data/util/TypeInformation.java`），接口体为空，继承 `org.springframework.data.core.TypeInformation`，标注 `@Deprecated(forRemoval = true)`。该类与 Spring Data 的包名相同，使用时需确认类路径中实际生效的版本。

---

## 💡 8. 使用范式

### 8.1 纯 POJO 租户实体
```java
import io.github.ybb88.framework.data.model.PojoTenantEntity;
import lombok.Data;
import lombok.EqualsAndHashCode;

@Data
@EqualsAndHashCode(callSuper = true)
public class OrderRecord extends PojoTenantEntity {

    private String orderNo;
    private Integer status;
}
```

### 8.2 实现操作人 SPI
```java
import io.github.ybb88.framework.data.spi.AuditUserProvider;
import org.springframework.stereotype.Component;

@Component
public class AuthAuditUserProvider implements AuditUserProvider {

    @Override
    public Long getUserId() {
        // 由业务侧从认证上下文读取当前操作人 ID
        return currentUserIdOrNull();
    }

    @Override
    public String getAccount() {
        return currentAccountOrNull();
    }
}
```

---

## 🔗 9. 相关文档

- 持久层实现: [starter-jpa](./starter-jpa.md)（`JpaBaseEntity` / `JpaTenantEntity`）
- 核心基础: [core](./core.md)
- 数据权限已迁出本模块，参见 `zhkj-framework-starter-datascope`
