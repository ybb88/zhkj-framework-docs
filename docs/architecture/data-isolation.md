# ZHKJ Framework 多租户隔离与细粒度数据权限体系

> **文档定位**: 企业级多租户物理/逻辑隔离、行级过滤拦截器顺序治理、JSqlParser AST 抽象语法树动态 SQL 重写与组织拓扑 SPI 深度指南  
> **设计与创作者**: Jason.Yang（274107098@qq.com）  

---

## 🏢 1. 多租户全生命周期治理 (`starter-tenant` + `starter-mybatis`)

### 1.1 核心设计理念
框架将多租户架构分为两层：
- **框架无关核心层 (`starter-tenant`)**: 负责多租户上下文的接收、存储、校验与 SPI 规则匹配；
- **持久层适配插件 (`starter-mybatis` / `starter-jpa`)**: 作为轻量级插件，根据核心层契约在 SQL 生成期或 ORM 会话中自动追加租户隔离条件。

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
       │ TenantLineHandler │           │  Hibernate Filter │
       └───────────────────┘           └───────────────────┘
```

### 1.2 行级隔离与双模探测自愈机制
在现实微服务中，可能存在无 Servlet 请求上下文的场景（如 Spring Batch 批处理、Kafka 异步消息消费或 `@Async` 线程池）。此时直接放行会导致严重的数据越权！
框架的 `MybatisTenantHandler` 实现了**双模自愈探测机制**：
1. 首先读取 `TenantContext.getTenantId()`；
2. 若为空，自动动态降级探测安全上下文 `SecurityContextHolder.getTenantId()`；
3. 若安全上下文中持有租户身份，立即执行**自动自愈回填** `TenantContext.setTenantId(...)` 并强制执行行级数据过滤（`WHERE tenant_id = 'xxx'`），彻底杜绝多租户越权漏洞。

### 1.3 拦截器链严格拓扑编排（杜绝注入竞态）
在 Spring 容器中，租户拦截器与数据权限拦截器可能由不同配置类注入。框架通过动态索引编排机制，强制保证物理链拓扑：
$$\mathbf{TenantLineInnerInterceptor} \longrightarrow \mathbf{MybatisDataScopeInterceptor} \longrightarrow \mathbf{PaginationInnerInterceptor}$$
- **租户拦截器绝对处于第 0 位**：租户隔离是最高级别的物理/逻辑边界，必须先于租户内的数据权限过滤与分页截断生效。

### 1.4 基础配置示例
```yaml
zhkj:
  tenant:
    enabled: true
    column: "tenant_id"
    ignore-tables:
      - "sys_dict"
      - "sys_region"
      - "sys_tenant"
```

---

## 📊 2. 细粒度数据权限体系 (`starter-datascope`)

### 2.1 架构设计：JSqlParser AST 语法树重写
传统框架常使用裸字符串拼接（如 `String.format(" AND dept_id IN (%s)", ...)`），极易引发 SQL 语法错误或 SQL 注入漏洞。
`zhkj-framework` 全面采用 **JSqlParser 抽象语法树（AST）** 解析引擎：
1. 拦截带有 `@DataAuth` 的 Mapper 方法或业务查询；
2. 解析原始 SQL 为 AST 语法树结构；
3. 将数据权限条件（本部门、本部门及子部门、仅本人、自定义等）构建为安全的 `BinaryExpression` 节点；
4. 递归合并进主查询或子查询的 `WHERE` 条件中，保障生成的 SQL 100% 语法严密、零注入风险。

### 2.2 组织架构解耦：`DeptAncestorProvider` SPI
框架基础层严禁硬编码任何特定业务部门表（如 `sys_org`、`sys_dept`）或特定字段。
业务系统只需实现框架提供的 SPI 接口，即可适配任意企业特定的组织树拓扑模型：

```java
@Component
public class CustomDeptAncestorProvider implements DeptAncestorProvider {

    @Autowired
    private SysDeptMapper deptMapper;

    @Override
    public List<Long> getSubDeptIds(Long deptId) {
        // 业务工程自由根据自身表结构（祖籍列表 ancestors 或闭包表）获取所有子部门 ID
        return deptMapper.selectChildDeptIds(deptId);
    }
}
```

### 2.3 注解使用实战 (`@DataAuth`)
在 Mapper 查询接口上直接声明 `@DataAuth`，框架自动注入过滤逻辑：

```java
@Mapper
public interface CustomerMapper extends BaseMapper<CustomerEntity> {

    /**
     * 根据当前登录用户的角色数据权限范围（全部/本部门及下级/仅本人）自动追加过滤条件
     */
    @DataAuth(deptField = "dept_id", userField = "create_by")
    List<CustomerEntity> selectCustomerPage(IPage<CustomerEntity> page, @Param("param") CustomerQueryDTO param);
}
```

---

## 🛡️ 3. 租户与数据权限协同生命周期速查表

| 场景 | 租户隔离行为 | 数据权限过滤行为 | 上下文清理保障 |
| :--- | :--- | :--- | :--- |
| **标准 HTTP 请求** | 从请求头 `X-Tenant-Id` 提取，自动追加 `tenant_id = ?` | 从用户角色配置解析，追加部门/人员限制 | `FrameworkContextClearFilter` 在 `finally` 强制清空 |
| **异步线程池 (`@Async`)** | 通过 `FrameworkContext.capture()` 捕获快照并还原 | 快照完整携带权限模型与票据信息 | `try (scope)` 离开作用域自动清空，防复用污染 |
| **Kafka 消息消费** | 生产者写入 Record Headers，消费者自动还原 | 随消费者上下文还原 | `KafkaTraceRecordInterceptor` 在 `afterRecord` 强制清空 |
| **特权导出 / 全局查询** | 使用 `TenantContext.runWithTenant(...)` 显式受控切换 | 使用 `DataScopeContext` 显式受控调整 | 执行完毕自动还原先前上下文，防线程残留 |
