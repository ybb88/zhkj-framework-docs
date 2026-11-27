# 多租户治理核心：zhkj-framework-starter-tenant

> **模块定位**: 框架无关的多租户上下文与解析模块。提供 `TenantContext` 线程上下文、请求头与子域名解析器、租户注册表、RPC 租户头透传，以及 MyBatis 行级隔离所依赖的 `TenantRuleProvider` SPI。  
> **适用基准**: OpenJDK 21, Spring Boot 4.x  
> **模块坐标**: `io.github.ybb88:zhkj-framework-starter-tenant`

---

## 📋 1. 架构定位与核心设计

1. **上下文与 ORM 解耦**：`TenantContext` 基于 `ThreadLocal<TenantInfo>`，不依赖具体持久层。MyBatis 行级隔离由 `starter-mybatis` 的 `MybatisTenantHandler` 通过 `TenantRuleProvider` 读取上下文。
2. **请求解析链**：`TenantInterceptor` 按 `TenantResolver` 的 `@Order` 依次执行；首个返回非 `null` 的解析结果写入 `TenantContext`，请求结束后清理。
3. **未识别租户不拒绝请求**：拦截器对未识别的请求放行，租户上下文为空。是否需要租户由业务或 ORM 层判断。
4. **静态注册与请求状态分离**：`TenantRegistry` 仅保存静态元数据（不含 `dimensionId`、`delegatedTenantId`、`grantTicket`）。解析结果为请求级实例，不回写注册表。
5. **快照传播**：`TenantContextSnapshotProvider` 注册到 `FrameworkContext`，跨线程使用 `FrameworkContext.capture()` 与 `runWith` / `openScope` 传递；工作线程不会自动继承 `ThreadLocal`。

---

## 📦 2. 依赖坐标

```xml
<dependency>
    <groupId>io.github.ybb88</groupId>
    <artifactId>zhkj-framework-starter-tenant</artifactId>
</dependency>
```

---

## 🧱 3. 核心包结构与类

```
io.github.ybb88.framework.tenant
├── annotation/TenantIgnore.java                 # 方法/类级忽略租户解析（仅作用于 HTTP 拦截器）
├── config/TenantCoreAutoConfiguration.java      # 自动装配（需 zhkj.tenant.enabled=true）
├── context/
│   ├── TenantContext.java                       # ThreadLocal 上下文与作用域工具
│   ├── TenantContextHolder.java                 # 门面（委托 TenantContext）
│   └── TenantContextSnapshotProvider.java       # FrameworkContext 快照（order 100）
├── core/
│   ├── TenantConstant.java                      # DEFAULT_TENANT_ID = "0"，ADMIN_TENANT_ID = "10000001"
│   ├── TenantRuleProvider.java                  # 行级隔离规则 SPI
│   ├── DefaultTenantRuleProvider.java           # 默认实现（读取 zhkj.tenant.field-isolation.*）
│   ├── TenantId.java, ZhkjTenantId.java         # 租户 ID 生成 SPI（默认返回 "0"）
├── interceptor/TenantInterceptor.java           # Servlet 拦截器
├── model/
│   ├── TenantInfo.java                          # 租户模型
│   ├── TenantIsolationMode.java                 # SHARED_DATABASE / SHARED_INSTANCE / ISOLATED_DATABASE（仅枚举）
│   └── TenantDataSourceConfig.java              # 数据源元数据（仅保存）
├── properties/
│   ├── TenantProperties.java                    # zhkj.tenant.*
│   └── TenantFieldIsolationProperties.java      # column / tables / ignore-tables / default-tenant-id
├── registry/TenantRegistry.java
├── resolver/
│   ├── TenantResolver.java                      # 解析 SPI
│   ├── HeaderTenantResolver.java                # @Order(1)
│   └── SubdomainTenantResolver.java             # @Order(2)
└── rpc/TenantRpcHeaderProvider.java             # RPC 头透传
```

自动装配（`META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`）：`TenantCoreAutoConfiguration`。

启用后，核心配置独立装配 `TenantRegistry`、`TenantContextSnapshotProvider`、`TenantRuleProvider` 与 `TenantRpcHeaderProvider`，纯 MyBatis、批处理等非 Web 应用无需补充 Spring MVC 或 Servlet 依赖。Header/Subdomain 解析器、`TenantInterceptor` 及 MVC 注册逻辑隔离在静态嵌套 `TenantWebMvcConfiguration` 中，仅在 Servlet Web 应用且 `WebMvcConfigurer`、`HttpServletRequest` 类存在时装配，避免核心配置的方法签名提前加载可选 Web 类型。

---

## ⚙️ 4. 核心配置清单 (`application.yml`)

```yaml
zhkj:
  tenant:
    enabled: true                        # 必须显式设为 true，缺省不装配
    field-isolation:
      column: tenant_id                  # 默认 tenant_id
      tables: []                         # 非空时仅这些表参与隔离
      ignore-tables: [sys_tenant, sys_dict]
      default-tenant-id: "10000001"      # 绑定值，DefaultTenantRuleProvider 未读取
    tenants:                             # 静态租户元数据（可选）
      - tenant-id: "1001"
        tenant-name: "示例机构"
        tenant-code: "demo"              # 子域名解析键
        isolation-mode: SHARED_DATABASE
        data-source-key: "ds_demo"       # 仅作元数据保存
```

| 键 | 默认值 | 说明 |
|----|--------|------|
| `zhkj.tenant.enabled` | 未设置时不装配 | 条件注解 `matchIfMissing=false` |
| `zhkj.tenant.default-isolation-mode` | `SHARED_DATABASE` | 已绑定，本模块内未读取 |
| `zhkj.tenant.resolvers` | `[header, subdomain]` | 已绑定，本模块内未读取，解析器顺序由 `@Order` 决定 |
| `zhkj.tenant.tenants` | 空 | 启动时注册到 `TenantRegistry` |
| `zhkj.tenant.field-isolation.column` | `tenant_id` | 默认规则提供者的全局隔离列 |
| `zhkj.tenant.field-isolation.tables` | 空 | 全局白名单表（空表示全部表，忽略表除外） |
| `zhkj.tenant.field-isolation.ignore-tables` | 空 | 全局不做租户隔离的表 |

MyBatis 配置优先级（详见 [starter-mybatis](starter-mybatis.md)）：

- 使用框架自动创建的 `DefaultTenantRuleProvider` 时，`zhkj.tenant.field-isolation.column`、`tables`、`ignore-tables` 提供全局默认值；`zhkj.tenant.mybatis` 下的同名字段按实际显式绑定逐字段覆盖，未配置字段继续继承全局值。
- 适配通过 Spring Binder 完成，不以属性默认值推断配置显式性；显式 `column: tenant_id` 仍属于覆盖，列表整体替换，显式空列表可清空继承列表。MyBatis 使用独立的默认规则提供者实例，不修改共享全局配置或其他 ORM 规则。
- 应用自定义的 `TenantRuleProvider` 始终优先，包括直接注册的 `DefaultTenantRuleProvider`、其子类或代理；MyBatis 配置不会覆盖其列名与表级裁决。租户身份解析和是否忽略隔离仍由规则提供者负责，配置适配不改变缺失上下文策略。
- 若裁剪自动配置、完全没有 `TenantRuleProvider` Bean，`MybatisTenantHandler` 才使用 `zhkj.tenant.mybatis.*` 的内建路径，此时不继承全局字段规则。

---

## 🔎 5. 租户解析

### 5.1 请求头（`HeaderTenantResolver`，`@Order(1)`）

按顺序读取第一个非空值作为租户 ID：`X-Tenant-Id`、`X-Tenant-ID`、`tenant-id`、`zhkj-tenant-id`。

- 注册表中存在该租户：使用注册元数据的副本；
- 不存在：构造仅属于本次请求的 `TenantInfo`（`SHARED_DATABASE`），不写入注册表；
- `X-Dimension-Id` 非空时写入 `dimensionId`；
- 不接受 `X-Delegated-Tenant-Id` 或 `X-Grant-Ticket`；即使 `TENANT_DELEGATION_AUTHORIZED` 请求属性为 `Boolean.TRUE`，也不授予跨租户请求权限。

> 未知请求头租户会成为当前请求的租户上下文，**租户解析本身不做授权校验**。公网入口须剥离客户端自带的租户与内部委托头，并由业务完成授权。网关在认证、匿名白名单及 `enabled=false` 分支均剥离 `X-Internal-RPC`、`X-Delegated-Tenant-Id`、`X-Grant-Ticket`。

**安全行为变更与迁移**：目前没有可核验的委托票据签发及端点级资源/行为校验契约，委托头和可信 JWT 中的委托声明均不能作为跨租户授权。ORM 对携带委托元数据的上下文直接拒绝；业务必须先实现受保护的具体操作及服务端审批、主体、目标租户、资源范围、时效和撤销校验，不能通过通用上下文切换开放写入。

### 5.2 子域名（`SubdomainTenantResolver`，`@Order(2)`）

取主机名首段（主机至少 3 段，如 `demo.example.com`）作为键，在注册表中按 `tenant-code` 查找。

### 5.3 解析器扩展

实现 `TenantResolver`（`HttpServletRequest` → `TenantInfo`，`getOrder()` 决定顺序，默认 100），注册为 Bean 即被 `TenantInterceptor` 收集。`TenantCoreAutoConfiguration.TenantWebMvcConfiguration` 中的 `headerTenantResolver`、`subdomainTenantResolver`、`tenantInterceptor` 均为 `@ConditionalOnMissingBean`。

### 5.4 `TenantInterceptor`

- 仅 Servlet Web 应用且 MVC/Servlet 相关类存在时装配，注册路径 `/**`；
- 方法或类标注 `@TenantIgnore` 时跳过解析；
- 按解析器顺序执行，首个非空结果绑定到 `TenantContext`；解析器抛出的异常仅记录 WARN；
- `afterCompletion` 清理 `TenantContext`。

`@TenantIgnore` 只影响 HTTP 解析，不影响 MyBatis 的 SQL 改写。

---

## 💡 6. 上下文 API

### 6.1 `TenantContext`（静态方法）

| 分类 | 方法 |
|------|------|
| 读取 | `getTenant()`、`getTenantId()`、`getDimensionId()`、`getDelegatedTenantId()`、`getGrantTicket()`、`getDataSourceKey()`、`hasTenant()` |
| 写入 | `setTenant(TenantInfo)`、`setTenantId(String)`、`setDimensionId(String)`、`setDelegatedTenantId(String)`、`setGrantTicket(String)`、`restoreSnapshot(...)`、`clear()` |
| 作用域 | `executeInTenant` / `callInTenant` / `callInTenantChecked`（仅切换租户 ID，新作用域不继承维度与委托）；`runWithTenant` / `callWithTenant` / `callWithTenantChecked`（以 `TenantInfo` 切换，退出时恢复原值）；`runWithDimension`、`callWithDimension`；`runWithDelegation`、`callWithDelegation` |

`restoreSnapshot` 对未提供的字段按 `null` 写入，不保留先前值。

### 6.2 `TenantContextHolder`

与 `TenantContext` 功能对应的门面，提供 `setTenant`、`setTenantId`、`getTenantId`、`hasTenant`、`clear`、`runWithTenant`、`runWithTenantId`、`callWithTenant`、`callWithTenantChecked`。

### 6.3 `TenantRegistry`

`registerTenant(TenantInfo)`（别名 `register`）、`getTenantInfo(tenantId)`、`getTenantBySubdomain(code)`、`getTenantByDataSourceKey(key)`、`removeTenant(tenantId)`、`getAllTenants()`、`exists(tenantId)`。所有读取返回元数据副本。

### 6.4 `TenantInfo`

`tenantId`、`dimensionId`、`delegatedTenantId`、`grantTicket`、`tenantName`、`tenantCode`、`isolationMode`、`dataSourceKey`、`schemaName`、`dataSourceConfig`（`TenantDataSourceConfig`）、`status`、`isDefault`。

`dataSourceKey`、`schemaName`、`dataSourceConfig` 仅作为元数据保存，本模块不执行数据源切换。

---

## 🔗 7. RPC 与跨线程

- `TenantRpcHeaderProvider` 只透传 `X-Tenant-Id`、`X-Tenant-ID`、`zhkj-tenant-id`（主体租户 ID）及 `X-Dimension-Id`；不透传委托目标与票据。租户 ID 依次取自 `TenantContext`、`FrameworkContext`、安全上下文快照。
- 接收端忽略委托头；传输头与异步上下文快照均不承载跨租户授权。普通 RPC 与公共展示端点默认脱敏，明文权限另由 `DESENSITIZE_AUTHORIZED` 请求属性控制。
- 异步任务应在提交线程捕获快照：

```java
ContextSnapshot snapshot = FrameworkContext.capture();
executor.execute(() -> FrameworkContext.runWith(snapshot, this::doWork));
```

---

## 🧩 8. MyBatis 行级隔离规则（由 `starter-mybatis` 使用）

`DefaultTenantRuleProvider`（`zhkj.tenant.enabled=true` 时注册）提供：

- `getTenantColumn()`：`zhkj.tenant.field-isolation.column`；
- `ignoreTable(name)`：无租户上下文时返回 `true`；否则命中 `ignore-tables` 或不在非空 `tables` 白名单中时返回 `true`；
- `resolveTenantId()`：存在未核验委托目标或票据时拒绝；否则依次读取主体 `TenantContext`、安全上下文快照，最后返回 `TenantConstant.DEFAULT_TENANT_ID`（`"0"`）。

表名比较忽略大小写，并去除反引号与双引号。

---

## 💡 9. 典型使用范式

### 9.1 获取当前租户
```java
String tenantId = TenantContext.getTenantId();
if (tenantId == null) {
    throw new IllegalStateException("当前未检测到租户上下文");
}
```

### 9.2 受控切换租户
```java
// 调用方须先完成目标租户的授权校验
TenantContext.executeInTenant(targetTenantId, this::generateReport);
```

### 9.3 忽略 HTTP 解析
```java
@TenantIgnore
@GetMapping("/health")
public String health() { return "OK"; }
```

### 9.4 自定义行级规则
```java
@Bean
public TenantRuleProvider tenantRuleProvider(TenantProperties properties) {
    return new DefaultTenantRuleProvider(properties);   // 或实现 TenantRuleProvider 接口
}
```

注册自定义 Bean 时，`TenantCoreAutoConfiguration` 的默认 Bean 不再创建。

### 9.5 静态注册租户
```java
TenantInfo tenant = new TenantInfo();
tenant.setTenantId("1001");
tenant.setTenantCode("demo");
tenant.setIsolationMode(TenantIsolationMode.SHARED_DATABASE);
tenantRegistry.registerTenant(tenant);
```

---

## 🛡️ 10. 行为约束

- 无租户上下文时，MyBatis 租户条件不附加（`ignoreTable` 返回 `true`）。批处理与异步任务须显式绑定租户。
- 未知请求头租户不做授权校验；授权由调用链上游或网关保证。
- `TenantIsolationMode` 与 `dataSourceKey` 不会自动切换数据源。

---

## 📎 11. 相关文档

- `starter-mybatis`：`MybatisTenantAutoConfiguration`、插件链顺序
- `starter-sso-client`、`starter-secure`：身份与租户的关系
