# 数据持久化抽象层：zhkj-framework-data-api

> **模块定位**: 框架无关数据访问与持久层 SPI 抽象层，统一定义泛型实体规范、审计契约、正交维度标记及多数据源 SPI。  
> **适用基准**: OpenJDK 21 (LTS), Spring Boot 4.1.1  
> **模块坐标**: `io.github.ybb88:zhkj-framework-data-api`

---

## 📋 1. 架构定位与职责

`zhkj-framework-data-api` 作为持久层契约抽象层，位于 `core` 之上、具体持久化 Starter（`starter-mybatis` / `starter-jpa`）之下：

- **框架无关**: 不包含任何第三方 ORM 框架（如 MyBatis-Plus 或 Hibernate）的依赖或注解，保持 100% 纯粹性；
- **SPI 优先**: 通过 SPI 接口规范多数据源接入与动态仓储获取；
- **实体正交维度解耦**: 彻底废弃单一强耦合继承树，通过多接口正交解耦（主键强类型泛型、审计契约、租户标记、逻辑删除标记、乐观锁标记），供业务实体按需组合。

```
              ┌───────────────────────────┐
              │  zhkj-framework-core      │
              └─────────────┬─────────────┘
                            ▼
              ┌───────────────────────────┐
              │  zhkj-framework-data-api  │
              │  - IAuditEntity<ID>       │
              │  - ITenantEntity          │
              │  - PojoBaseEntity / Pojo  │
              │  - DataSourceProvider SPI │
              └─────────────┬─────────────┘
                            │
              ┌─────────────┴─────────────┐
              ▼                           ▼
┌───────────────────────────┐ ┌───────────────────────────┐
│starter-mybatis (MP实现)   │ │starter-jpa (JPA实现)      │
└───────────────────────────┘ └───────────────────────────┘
```

---

## 📦 2. 依赖坐标

```xml
<dependency>
    <groupId>io.github.ybb88</groupId>
    <artifactId>zhkj-framework-data-api</artifactId>
</dependency>
```

> [!NOTE]
> 通常业务工程只需引入 `zhkj-framework-starter-mybatis` 或 `zhkj-framework-starter-jpa`，`data-api` 会作为传递依赖自动引入。

---

## 🧱 3. 核心包结构与模型契约

```
io.github.ybb88.framework.data
├── model/                    # 核心模型与正交维度契约
│   ├── IEntity.java          # 泛型主键根契约 (ID getId(), void setId(ID id))
│   ├── IAuditEntity.java     # 审计契约 (创建人、创建时间、更新人、更新时间)
│   ├── ITenantEntity.java    # 多租户标记契约 (String getTenantId(), void setTenantId(String))
│   ├── ILogicDeleteEntity.java# 逻辑删除正交标记契约 (T getIsDeleted(), void setIsDeleted(T))
│   ├── IVersionEntity.java   # 乐观锁版本正交标记契约 (T getVersion(), void setVersion(T))
│   ├── PojoBaseEntity.java   # 纯 POJO 基础审计实体 (主键默认 Long)
│   └── PojoTenantEntity.java # 纯 POJO 多租户审计实体 (主键默认 Long，包含 tenantId)
├── repository/               # 仓储抽象契约
│   ├── Repository.java       # 通用 CRUD 仓储接口
│   ├── Page.java             # 通用分页数据容器
│   └── PageRequest.java      # 分页查询参数请求
├── query/                    # 条件构造抽象
│   └── QueryWrapper.java     # 通用查询包装器抽象
└── spi/                      # 架构级扩展 SPI 接口
    ├── DataAccessProvider.java# 数据访问提供者 SPI
    └── DataSourceProvider.java# 数据源提供者 SPI
```

---

## 🛡️ 4. 正交维度解耦设计体系

传统框架往往设计一条单一而沉重的 `BaseEntity` 继承链，导致业务实体无法自由组合持久化能力。`data-api` 将维度完全解耦：

| 契约接口 | 职责定位 | 继承关系 | 核心方法 |
| :--- | :--- | :--- | :--- |
| **`IEntity<ID extends Serializable>`** | 强类型主键根接口 | 继承 `Serializable` | `ID getId()` / `void setId(ID id)` |
| **`IAuditEntity<ID extends Serializable>`**| 实体审计字段契约 | 继承 `IEntity<ID>` | `createUser`, `createTime`, `updateUser`, `updateTime` |
| **`ITenantEntity`** | 多租户隔离标记 | 独立继承 `Serializable` | `String getTenantId()` / `void setTenantId(...)` |
| **`ILogicDeleteEntity<T>`** | 逻辑删除标记 | 独立继承 `Serializable` | `T getIsDeleted()` / `void setIsDeleted(...)` |
| **`IVersionEntity<T>`** | 乐观锁并发标记 | 独立继承 `Serializable` | `T getVersion()` / `void setVersion(...)` |

### 维度组合范式
- **纯粹 POJO 单体/轻量场景**：直接使用 `PojoBaseEntity` 或 `PojoTenantEntity`；
- **MyBatis-Plus 场景**：业务实体继承 `MybatisBaseEntity` 或 `MybatisTenantEntity`（自动实现上述接口并装配 MP 注解）；
- **JPA 场景**：业务实体继承 `JpaBaseEntity` 或 `JpaTenantEntity`。

---

## 💡 5. 使用范式与 SPI 扩展

### 5.1 自定义仓储 SPI 接入 (`DataAccessProvider`)
在无需依赖具体 ORM 的通用模块中，可通过 SPI 契约调用数据仓储：

```java
package io.github.ybb88.demo.spi;

import io.github.ybb88.framework.data.spi.DataAccessProvider;
import io.github.ybb88.framework.data.repository.Repository;
import org.springframework.stereotype.Component;

@Component
public class CustomDataAccessProvider implements DataAccessProvider {

    @Override
    public <T, ID> Repository<T, ID> getRepository(Class<T> entityClass) {
        // 返回与特定存储引擎适配的仓储实例
        return null;
    }
}
```

### 5.2 规范与防踩坑提示
- **避免直接在业务层手写 `new PojoTenantEntity()`**：在业务持久层中，推荐直接继承对应 ORM 的专用实体基类（如 `MybatisTenantEntity`），以保证字段自动填充与拦截器生效；
- **数据权限不可依赖 `data-api`**：细粒度数据权限管控已全面收敛至 `starter-datascope` 模块，严禁将数据权限逻辑散落在本抽象层中。
