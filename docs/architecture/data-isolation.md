# ZHKJ Framework 多租户隔离与细粒度数据权限体系

> **文档定位**: 多租户行级隔离、拦截器拓扑编排、JSqlParser AST 动态 SQL 改写与部门祖先 SPI 深度指南  
> **设计与创作者**: Jason.Yang（274107098@qq.com）  

---

## 🏢 1. 多租户全生命周期治理 (`starter-tenant` + `starter-mybatis`)

### 1.1 核心设计理念
框架将多租户分为两层：
- **框架无关核心层 (`starter-tenant`)**: 负责租户上下文（`TenantContext`）的存储、租户解析（`HeaderTenantResolver`、`SubdomainTenantResolver`）与规则 SPI（`TenantRuleProvider`，默认实现 `DefaultTenantRuleProvider`）；
- **持久层适配插件 (`starter-mybatis` / `starter-jpa`)**: 根据核心层契约在 SQL 生成期（MyBatis-Plus `TenantLineInnerInterceptor`）或 ORM 会话中（JPA Hibernate Filter）追加租户隔离条件。

```
                    ┌─────────────────────────┐
                    │ HTTP / RPC / MQ Request │
                    └────────────┬────────────┘
                                 │
                                 ▼
                    ┌─────────────────────────┐
                    │  TenantContext 上下文   │
                    └────────────┬────────────┘
                                 │
                 ┌───────────────┴───────────────┐
                 ▼                               ▼
       ┌───────────────────┐           ┌───────────────────┐
       │   MyBatis-Plus    │           │  Spring Data JPA  │
       │ MybatisTenantHandler│         │  Hibernate Filter │
       └───────────────────┘           └───────────────────┘
```

### 1.2 行级隔离与双模探测自愈机制
在无 Servlet 请求上下文的场景（如批处理、Kafka 消费、`@Async` 线程池）中，若直接放行会导致越权。`MybatisTenantHandler` 的处理逻辑：
1. 注册了 `TenantRuleProvider` 时（常规装配），`MybatisTenantHandler.getTenantId()` 先拒绝携带未核验委托目标或票据的上下文，再调用 `ruleProvider.resolveTenantId()`；默认 provider 只从主体 `TenantContext`、安全上下文快照解析租户，缺失时返回默认租户 `0`。
2. 未注册 `TenantRuleProvider` 时（裁剪自动配置的内建路径），同样先拒绝委托元数据，再按主体 `TenantContext` → `FrameworkContext` 安全上下文快照顺序解析；绝不因 `grantTicket` 放宽任意 SQL。
3. 配置项 `zhkj.tenant.mybatis.default-tenant-id`（默认 `10000001`）仅在处理器仍取不到租户 ID 时兜底（第 54、74 行），不覆盖提供者已返回的 ID。
4. `ignoreTable()`：携带委托元数据时先拒绝；无租户上下文时，注册默认 `TenantRuleProvider` 或采用内建路径均会返回 `true`，不附加租户条件。业务入口必须先完成主体身份认证，不得把无上下文查询当成跨租户授权。

### 1.3 拦截器链拓扑编排
拦截器链的实际装配见 `docs/architecture/overview.md` 第 3.4 节：
$$\mathbf{TenantLineInnerInterceptor} \longrightarrow \mathbf{MybatisDataScopeInterceptor} \longrightarrow \mathbf{Custom} \longrightarrow \mathbf{PaginationInnerInterceptor}$$
- 租户拦截器先于数据权限拦截器、分页拦截器生效，保证分页计数也携带租户条件。

### 1.4 配置示例
租户相关配置项的前缀与绑定类：

| 配置前缀 | 绑定类 | 常用项 |
| :--- | :--- | :--- |
| `zhkj.tenant` | `TenantProperties` | `enabled`（默认 `true`）、`default-isolation-mode`、`resolvers`（默认 `header`、`subdomain`）、`tenants`、`field-isolation` |
| `zhkj.tenant.mybatis` | `MybatisTenantProperties` | `column`（默认 `tenant_id`）、`tables`、`ignore-tables`、`default-tenant-id`（默认 `10000001`） |

```yaml
zhkj:
  tenant:
    enabled: true
    mybatis:
      column: tenant_id
      ignore-tables:
        - sys_dict
        - sys_region
        - sys_tenant
```

### 1.5 JPA 自动过滤与实体生命周期

`JpaTenantEntity` 直接声明 Hibernate `@FilterDef(autoEnabled = true, applyToLoadByKey = true)` 与 `@Filter`，每个 Session 自动参与隔离。`JpaTenantIdResolver` 按当前 `TenantRuleProvider` 解析有效租户，`JpaTenantFilterEnabledResolver` 只读取启动配置 `zhkj.tenant.enabled`；不注册 MVC 拦截器，不强制开启 OSIV。

启用时 provider 返回空租户会拒绝操作，默认 provider 缺少上下文时仍隔离到其默认租户 `0`。`TenantEntityListener` 在新增时填充并校验租户，在更新时校验当前租户、实体字段及原始持久化归属；不匹配直接阻断。关闭隔离不影响公共实体 `JpaBaseEntity` 的持久化。

请求结束使用现有 FrameworkContext 清理机制；resolver 不保存前一请求租户。切换租户使用独立事务和 EntityManager，避免一级缓存复用；原生 SQL 和绕过实体生命周期的批量 DML 需显式携带租户条件。完整配置与范围见 [JPA 组件](../starters/starter-jpa.md)。

---

## 📊 2. 细粒度数据权限体系 (`starter-datascope` + `starter-mybatis`)

### 2.1 架构设计：JSqlParser AST 语法树改写
框架使用 JSqlParser 解析 SQL 为抽象语法树（AST），在 `WHERE` 条件中合并数据权限表达式，而非拼接字符串：
1. 数据权限注解 `@DataAuth` 标注的方法或类，在 MyBatis 拦截器（`MybatisDataScopeInterceptor`）中被识别；
2. 通过 `SqlDataScopeAstEngine` 解析原 SQL 并改写；
3. 标识符通过 `DataScopeSqlUtil` 校验（仅允许 `[a-zA-Z0-9_]`）。

### 2.2 组织架构解耦：`DeptAncestorProvider` SPI
框架基础层通过 SPI 获取部门层级，不写死业务部门表。接口定义（`datascope/provider/DeptAncestorProvider.java`）：

```java
@FunctionalInterface
public interface DeptAncestorProvider {
    /** 获取指定部门及其所有子孙部门的 ID 列表（包含目标部门本身） */
    List<Long> getDeptAncestors(Long deptId);
}
```

业务系统实现该接口即可适配任意组织树模型：
```java
@Component
public class CustomDeptAncestorProvider implements DeptAncestorProvider {

    @Autowired
    private SysDeptMapper deptMapper;

    @Override
    public List<Long> getDeptAncestors(Long deptId) {
        // 业务工程根据自身表结构（ancestors 列或闭包表）返回目标部门及子孙部门 ID
        return deptMapper.selectSelfAndDescendantIds(deptId);
    }
}
```

未注册 `DeptAncestorProvider` 实现时，框架基于 `DataScopeProperties` 配置的表与列执行动态 SQL 方案（见下表）。

### 2.3 配置项 (`zhkj.datascope`)
`DataScopeProperties` 配置项（默认值以源码为准）：

| 配置项 | 默认值 | 说明 |
| :--- | :--- | :--- |
| `enabled` | `false` | 数据权限总开关 |
| `dept-table` | `sys_org` | 部门表名（存量默认值，见下方说明） |
| `dept-id-column` | `id` | 部门 ID 列 |
| `dept-ancestors-column` | `ancestors` | 祖先路径列 |
| `dept-logic-delete-column` / `dept-logic-delete-value` | `deleted_id` / `0` | 部门表逻辑删除列与未删除值 |
| `scope-data-table` | `zhkj_scope_data` | 数据范围表（存量默认值） |
| `role-scope-table` | `zhkj_role_scope` | 角色数据范围表（存量默认值） |
| `mapper-key` | `page`, `Page`, `list`, `List`, `update`, `Update`, `delete`, `Delete` | 参与改写的 Mapper 方法名匹配关键字 |
| `mapper-exclude` | `FlowMapper` | 排除的 Mapper 名 |

> 存量待治理: 上表中 `dept-table`、`scope-data-table`、`role-scope-table` 的默认值为业务相关表名，框架层规约要求不硬编码业务表名，需改为无默认值或显式配置。

### 2.4 注解使用 (`@DataAuth`)
`@DataAuth` 可标注在方法或类上（`@Target({METHOD, TYPE})`），属性如下（`datascope/annotation/DataAuth.java`）：

| 属性 | 默认值 | 说明 |
| :--- | :--- | :--- |
| `enabled` | `true` | 是否启用 |
| `code` | `""` | 资源编号 |
| `column` | `DataScopeConstant.DEFAULT_COLUMN` | 数据权限对应字段 |
| `type` | `DataScopeEnum.ALL` | 数据权限规则 |
| `field` | `"*"` | 可见字段 |
| `value` | `""` | 数据权限规则值域 |

`DataScopeEnum` 取值：`ALL`（全部）、`OWN`（本人可见）、`OWN_DEPT`（所在机构可见）、`OWN_DEPT_CHILD`（所在机构及子级可见）、`CUSTOM`（自定义）。

```java
@Mapper
public interface CustomerMapper extends BaseMapper<CustomerEntity> {

    @DataAuth(column = "dept_id", type = DataScopeEnum.OWN_DEPT_CHILD)
    List<CustomerEntity> selectCustomerList(@Param("param") CustomerQueryDTO param);
}
```

---

## 🛡️ 3. 租户与数据权限协同生命周期速查表

| 场景 | 租户隔离行为 | 数据权限过滤行为 | 上下文清理保障 |
| :--- | :--- | :--- | :--- |
| **标准 HTTP 请求** | 从请求头 `X-Tenant-Id` 解析（`HeaderTenantResolver`） | 通过 `ScopeModelHandler` 等 SPI 解析当前用户数据范围 | `FrameworkContextClearFilter` 调用 `FrameworkContext.clearAll()` |
| **异步线程池 (`@Async`)** | 通过 `FrameworkContext.capture()` 捕获快照并在子线程还原 | 快照携带数据范围模型 | `try (ContextScope)` 离开作用域后清理 |
| **Kafka 消息消费** | 生产端写入 Record Headers，消费端还原 | 随消费者上下文还原 | `KafkaTraceRecordInterceptor` 在 `afterRecord` 中调用 `FrameworkContext.clearAll()` |
| **特权操作（受控切换）** | `TenantContext.runWithTenant(tenant, runnable)` 显式切换并自动还原 | `DataScopeContext.setDataScope(...)` 显式设置，使用后 `DataScopeContext.clear()` | 作用域结束后还原先前上下文 |
