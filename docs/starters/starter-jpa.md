# JPA 持久层组件：zhkj-framework-starter-jpa

> **模块定位**: Spring Data JPA / Hibernate 持久层 Starter，提供审计实体基类、审计人自动填充、Hibernate 租户过滤器定义、租户字段自动赋值，以及存储加解密转换器与盲索引监听器。  
> **适用基准**: OpenJDK 21 (LTS), Spring Boot 4.1.1（Spring Data JPA 与 Hibernate 版本随 `spring-boot-starter-data-jpa` 管理）
> **模块坐标**: `io.github.ybb88:zhkj-framework-starter-jpa`

---

## 📋 1. 架构定位与职责

| 能力 | 类 | 说明 |
| :--- | :--- | :--- |
| 审计 | `JpaBaseEntity`、`AuditUserAuditorAware`、`JpaAutoConfiguration` | `@EnableJpaAuditing`，按 `AuditUserProvider` 填充创建人与修改人 |
| 租户字段 | `JpaTenantEntity`、`TenantEntityListener` | 统一规则填充租户，拒绝跨租户新增、更新及修改归属 |
| 租户过滤 | `JpaTenantIdResolver`、`JpaTenantAutoConfiguration` | Hibernate 自动过滤器覆盖每个 Session 的查询与主键加载 |
| 存储加密 | `CryptoAttributeConverter` | `AttributeConverter`，委托 `starter-crypto` 的 `StorageCryptoEngine` |
| 盲索引 | `BlindIndexEntityListener` | `@PrePersist` / `@PreUpdate` 计算盲索引哈希 |

依赖（来自 `pom.xml`）: `zhkj-framework-core`、`zhkj-framework-data-api`、`spring-boot-starter-data-jpa`；`zhkj-framework-starter-crypto`、`zhkj-framework-starter-tenant` 为 optional。JPA 模块不再依赖 MVC 或 Servlet。

---

## 🧱 2. 包结构

```
io.github.ybb88.framework.jpa
├── audit/AuditUserAuditorAware.java
├── config/JpaAutoConfiguration.java
├── crypto/
│   ├── CryptoAttributeConverter.java
│   └── listener/BlindIndexEntityListener.java
├── model/
│   ├── JpaBaseEntity.java
│   └── JpaTenantEntity.java
└── tenant/
    ├── JpaTenantAutoConfiguration.java
    ├── JpaTenantFilterConstant.java
    ├── JpaTenantFilterEnabledResolver.java
    ├── JpaTenantIdResolver.java
    └── listener/TenantEntityListener.java
```

自动装配（`META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`）注册:

- `io.github.ybb88.framework.jpa.config.JpaAutoConfiguration`
- `io.github.ybb88.framework.jpa.tenant.JpaTenantAutoConfiguration`

---

## ⚙️ 3. 配置项

### 3.1 租户（`zhkj.tenant`）

| 键 | 默认值 | 说明 |
| :--- | :--- | :--- |
| `zhkj.tenant.enabled` | `false` | 为 `true` 时执行租户过滤与写入校验；关闭时仍注册参数解析器，使普通 JPA 和显式租户字段实体均可工作 |

有效租户统一委托 `TenantRuleProvider.resolveTenantId()`，不读取请求 Header。默认 provider 支持已授权委托上下文与安全上下文自愈；无上下文时返回其默认租户 `0`，仍执行等值隔离，绝不放开全表。自定义 provider 返回 `null` 或空白时抛出 `ServiceException(ResultCode.UN_AUTHORIZED)`。

JPA 只使用 `zhkj.tenant.enabled` 开关，没有独立的租户列名或表忽略配置。需要隔离的实体继承 `JpaTenantEntity`，列固定为 `tenant_id`；公共实体使用 `JpaBaseEntity`。此适配器只使用 provider 的有效租户解析，不使用表忽略或二级维度接口。

### 3.2 存储加密（`zhkj.storage.crypto`，由 `starter-crypto` 解析）

| 键 | 默认值 | 说明 |
| :--- | :--- | :--- |
| `zhkj.storage.crypto.enabled` | `false` | 为 `false` 时转换器直接读写明文 |
| `zhkj.storage.crypto.write-enabled` | `true` | 为 `false` 时写入明文，读取仍解密历史密文 |
| `zhkj.storage.crypto.algorithm` | `AES` | 可选 `SM4` |
| `zhkj.storage.crypto.key` | — | 存储加密密钥，未配置时拒绝使用默认值 |
| `zhkj.storage.crypto.version` | `v1` | 密文版本前缀 |
| `zhkj.storage.crypto.blind-index-salt` | — | 盲索引盐值，未配置时拒绝使用默认值 |

### 3.3 Spring JPA 标准配置

```yaml
spring:
  jpa:
    open-in-view: false
    hibernate:
      ddl-auto: none
zhkj:
  tenant:
    enabled: true
```

---

## 🧩 4. 实体基类

### 4.1 `JpaBaseEntity`

`@MappedSuperclass`、`@EntityListeners(AuditingEntityListener.class)`，`public abstract class JpaBaseEntity implements IAuditEntity<Long>, ILogicDeleteEntity<Integer>, Serializable`。

| 字段 | 列 | 注解 |
| :--- | :--- | :--- |
| `id` | `id` | `@Id`、`@GeneratedValue(IDENTITY)` |
| `createBy` | `create_by`（不可更新） | `@CreatedBy` |
| `createTime` | `create_time`（不可更新） | `@CreatedDate` |
| `updateBy` | `update_by` | `@LastModifiedBy` |
| `updateTime` | `update_time` | `@LastModifiedDate` |
| `isDeleted` | `is_deleted`，默认 `0` | — |

> **代码现状**: `ILogicDeleteEntity<T>` 声明 `getDeleted()` / `setDeleted(T)`；`JpaBaseEntity` 通过 Lombok 生成 `getIsDeleted()` / `setIsDeleted()`，未实现 `getDeleted()` / `setDeleted()`（见 `JpaBaseEntity.java:28`、`:70-71`）。`JpaBaseEntity` 为抽象类，未实现接口方法不产生编译错误；具体实体须自行实现这两个方法（参考 `JpaTenantIsolationTest.java:298-302`）。

### 4.2 `JpaTenantEntity`

`@MappedSuperclass`、`@EntityListeners(TenantEntityListener.class)`，继承 `JpaBaseEntity` 并实现 `ITenantEntity`，新增字段 `tenantId`（列 `tenant_id`，`nullable = false`、`updatable = false`）。加载或新增成功后记录原始租户，更新时校验实体归属没有被改写。

### 4.3 `TenantEntityListener`

- 启用隔离时 `@PrePersist` 由统一 provider 解析有效租户；字段为空时填充，字段已赋值但不同于有效租户时抛出 `ServiceException(ResultCode.REQ_REJECT)`。
- `@PreUpdate` 同时校验当前有效租户、实体字段与原始持久化租户；不匹配直接拒绝，不再只输出日志。
- 关闭隔离时不执行租户填充和校验；若使用租户实体，仍须满足数据库 `tenant_id NOT NULL` 约束。
- 监听器与参数解析器（`TenantEntityListener`、`JpaTenantIdResolver`、`JpaTenantFilterEnabledResolver`）均带构造参数，需由 Hibernate 通过 Spring BeanContainer 实例化。本模块未定义静态 ApplicationContext 或 ThreadLocal 缓存。自建 EntityManagerFactory 时的 BeanContainer 配置需 `[待确认]`：本仓库未显式配置 `SpringBeanContainer`，Spring Boot 默认行为需运行时验证。

### 4.4 实体上的 Hibernate 过滤器

`JpaTenantEntity` 直接声明过滤器 `tenantFilter`（`JpaTenantFilterConstant.NAME`）：`@FilterDef(autoEnabled = true, applyToLoadByKey = true)` 含参数 `tenantId`（resolver `JpaTenantIdResolver`）与 `tenantIsolationEnabled`（resolver `JpaTenantFilterEnabledResolver`），并由 `@Filter` 绑定。Hibernate 会继承映射父类上的过滤器，无需子类重复声明。

过滤条件为 `(:tenantIsolationEnabled = false or tenant_id = :tenantId)`；启用标志只来自启动配置，有效租户由参数 resolver 按执行上下文解析，不显式设置、缓存 Session 级租户参数。关闭时租户参数仅绑定空字符串，禁用分支不执行租户匹配。

实体过滤器使用 Hibernate 直接映射注解，不以自定义组合元注解代替 `@FilterDef` / `@Filter`。

---

## 🔐 5. 审计人（`AuditUserAuditorAware`）

- 实现 `AuditorAware<Long>`，`getCurrentAuditor()` 读取容器中 `AuditUserProvider.getUserId()`（经 `ObjectProvider`）。
- 无 `AuditUserProvider` 或返回 `null` 时返回 `Optional.of(1L)`（代码现状，`AuditUserAuditorAware.java:40`）。
- 业务可通过声明自己的 `AuditorAware<Long>` Bean 覆盖（`JpaAutoConfiguration` 使用 `@ConditionalOnMissingBean(AuditorAware.class)`）。

---

## 🔑 6. 存储加密转换器（`CryptoAttributeConverter`）

```java
@Entity
@Table(name = "biz_customer")
public class CustomerEntity extends JpaTenantEntity {

    @Convert(converter = CryptoAttributeConverter.class)
    @Column(name = "mobile_cipher")
    private String mobileCipher;
}
```

- `@Converter(autoApply = false)`，需在字段上显式 `@Convert`。
- 写库时调用 `StorageCryptoEngine.encrypt`，读库时调用 `decrypt`。`null` 与空白值直接返回。
- 已带 `sm4:` 或 `v1:` 前缀的值不重复加密。
- 静态方法 `configure(...)` 与 `reset()` 供测试与手工注入使用。

---

## 🧮 7. 盲索引监听器（`BlindIndexEntityListener`）

- `@PrePersist` / `@PreUpdate` 时，经 `starter-crypto` 的 `BlindIndexBindingUtil.getBindings(entityClass)` 获取 `源字段 → 哈希字段` 映射，计算 `StorageBlindIndexUtil.generateIndex(明文)` 并写回哈希字段。
- 源字段为空白或已带 `sm4:` / `v1:` 前缀时跳过。
- 异常仅输出 `WARN` 日志。
- 本模块的 `JpaBaseEntity` 与 `JpaTenantEntity` 均未注册该监听器，需在实体上通过 `@EntityListeners(BlindIndexEntityListener.class)` 显式声明。

---

## 8. Session 生命周期与租户边界

过滤器由 Hibernate 为每个 Session 自动启用，不依赖 MVC 拦截器注册，也不创建请求级 EntityManager；租户过滤与事务使用同一个 Session 生命周期。

- `spring.jpa.open-in-view=false` 保持生效：Spring 事务创建的 Session 自带过滤器，事务结束照常关闭、解绑。
- OSIV、Servlet 请求、Batch 和普通非 Servlet 事务使用相同规则；参数 resolver 每次读取当前有效租户，不持有前一请求租户。请求结束仍由现有 FrameworkContext 清理机制释放上下文；Session 的原有所有者负责关闭，不新增 MVC 状态或清理链。
- 业务不得调用 `disableFilter` 或显式 `setParameter` 覆盖自动租户参数；不要跨租户复用含已加载实体的持久化上下文，Hibernate 一级缓存不会重新执行 SQL。需要切换租户时使用独立事务和 EntityManager。
- `applyToLoadByKey=true` 同时约束触发数据库读取的主键查找；实体生命周期监听器校验受管理实体新增、更新。原生 SQL 与绕过实体生命周期的批量 DML 不属于监听器校验边界，业务应显式加入租户条件，不能把任意原生 SQL 当作自动隔离查询。
- 只继承 `JpaBaseEntity` 的非租户实体不绑定过滤器；关闭 `zhkj.tenant.enabled` 不影响普通 JPA 使用，也不强制引入 Servlet/MVC。

回归测试 `JpaTenantIsolationTest` 使用独立 H2 内存数据库，覆盖无 OSIV 的事务查询与主键隔离、请求上下文清理后复用 Session 的参数更新、已授权委托、跨租户新增/更新/改写归属拒绝、空策略租户拒绝及禁用分支。测试不连接外部数据库。

---

## 💡 9. 实体定义示例

```java
@Getter
@Setter
@Entity
@Table(name = "biz_account")
public class AccountEntity extends JpaTenantEntity {

    private String accountNo;
    private BigDecimal balance;
}
```

> 不要在子类中重复声明 `id` 字段，主键由 `JpaBaseEntity` 提供。

---

## 🔗 10. 相关文档

- 数据契约与 POJO 实现: [data-api](./data-api.md)
- 租户模块: `zhkj-framework-starter-tenant`
- 存储加密与盲索引配置: `zhkj-framework-starter-crypto`
