# ZHKJ Framework 架构总览与设计规约

> **适用基准**: OpenJDK 21 (LTS), Spring Boot 4.1.1, Spring Cloud 2025.1.2, Spring Batch 6.0.5  
> **文档定位**: 框架总体拓扑、分层职责、模块矩阵与全局架构底线约束  
> **架构设计与创作者**: Jason.Yang（274107098@qq.com）  

---

## 📐 1. 整体分层架构

框架基于**单向依赖**和**关注点分离**原则，分为 5 个清晰的物理与逻辑层次：

```
┌─────────────────────────────────────────────────────────────────────────┐
│                       应用业务层 (Application Layer)                     │
│                     业务服务 / Controller / Scheduler                   │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                     表现层与通信层 (Web & RPC Layer)                     │
│   ┌───────────────────────────────┬─────────────────────────────────┐   │
│   │ zhkj-framework-starter-web    │ zhkj-framework-starter-http     │   │
│   │ - 统一 R<T> 响应自动封装       │ - Spring 6+ @HttpExchange RPC   │   │
│   │ - 全局异常强类型映射处理       │ - 全链路 Header 透传拦截器       │   │
│   │ - XSS 路径规整清洗防护        │ - 全局优雅降级 FallbackFactory  │   │
│   │ - TraceId 分布式链路追踪      │ - 负载均衡微服务直连支持        │   │
│   └───────────────────────────────┴─────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                    功能起步组件层 (Starters Integration)                 │
│   ┌───────────────────┬───────────────────┬─────────────────────────┐   │
│   │   数据与持久化    │     数据治理      │       安全与中间件      │   │
│   │ starter-mybatis   │ starter-tenant    │ starter-secure          │   │
│   │ starter-jpa       │ starter-datascope │ starter-sso-client      │   │
│   │ starter-datasource│                   │ starter-crypto          │   │
│   │                   │                   │ starter-redis           │   │
│   │                   │                   │ starter-kafka           │   │
│   │                   │                   │ starter-nacos           │   │
│   │                   │                   │ starter-openapi         │   │
│   └───────────────────┴───────────────────┴─────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                       数据抽象层 (Data API Layer)                        │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │ zhkj-framework-data-api                                         │   │
│   │ - IAuditEntity / ITenantEntity 统一数据访问与多租户审计契约     │   │
│   │ - PojoBaseEntity / PojoTenantEntity 纯 POJO 基础实体模型        │   │
│   │ - 框架无关数据源提供者 (DataSourceProvider SPI)                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                       核心基础层 (Core Engine Layer)                     │
│   ┌───────────────────────────────┬─────────────────────────────────┐   │
│   │ zhkj-framework-core           │ zhkj-framework-launcher         │   │
│   │ - 统一 R<T> 响应载体          │ - ZhkjApplication 启动器        │   │
│   │ - FrameworkContext 上下文门面 │ - 多环境配置与 Profile 探测     │   │
│   │ - Jackson 字段动态脱敏规则    │ - 启动耗时统计与 ASCII Banner   │   │
│   │ - 100+ 核心单数工具类库       │                                 │   │
│   └───────────────────────────────┴─────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 🧩 2. 模块职责矩阵 (18 Modules)

整个框架采用扁平 Starter 组合化设计，彻底根除空壳伪聚合模块：

| 层次 | 模块名 (`artifactId`) | 职责定位 | 核心类与契约 |
| :--- | :--- | :--- | :--- |
| **核心层** | `zhkj-framework-core` | 通用工具箱、协议模型、统一上下文门面、脱敏规则 | `R<T>`, `FrameworkContext`, `HeaderConstant`, `DateUtil`, `JsonUtil` |
| **核心层** | `zhkj-framework-launcher` | 微服务/单体统一引导启动器、环境探测、耗时分析 | `ZhkjApplication`, `LauncherService`, `PropUtil` |
| **数据抽象** | `zhkj-framework-data-api` | 框架无关持久化契约、审计模型、数据源 SPI | `IAuditEntity`, `ITenantEntity`, `PojoBaseEntity`, `DataSourceProvider` |
| **通信层** | `zhkj-framework-starter-http` | 声明式 HTTP 客户端、全链路 Header 透传、优雅降级 | `@EnableHttpClients`, `@HttpExchange`, `HttpClientHeaderInterceptor` |
| **Web层** | `zhkj-framework-starter-web` | 统一响应切面、全局异常处理、XSS 规整防护、TraceId | `DefaultErrorController`, `XssFilter`, `TraceFilter`, `WebUtil` |
| **持久层** | `zhkj-framework-starter-mybatis` | MyBatis-Plus 增强、自动审计填充、国密存储加密与盲索引 | `ZhkjMetaObjectHandler`, `MybatisTenantEntity`, `EncryptTypeHandler` |
| **持久层** | `zhkj-framework-starter-jpa` | Spring Data JPA 增强、Hibernate 租户 Filter 适配 | `JpaTenantAutoConfiguration`, `JpaTenantEntity`, `JpaBaseEntity` |
| **持久层** | `zhkj-framework-starter-datasource` | 动态多数据源路由、Hikari 连接池生命周期管理 | `DynamicDataSource`, `@DS`, `DynamicRoutingDataSource` |
| **数据治理** | `zhkj-framework-starter-tenant` | 多租户上下文治理、双模自愈探测、租户规则引擎 SPI | `TenantContext`, `TenantRuleProvider`, `TenantLineHandler` |
| **数据治理** | `zhkj-framework-starter-datascope` | 细粒度数据权限注解与 AST 树动态重写 SPI | `@DataAuth`, `ScopeModel`, `ScopeModelHandler`, `DeptAncestorProvider` |
| **安全技术** | `zhkj-framework-starter-secure` | 无状态令牌鉴权、方法级权限、Actuator 安全保护 | `SecureUtil`, `@PreAuth`, `ActuatorSecurityConfiguration`, `AuthAspect` |
| **安全技术** | `zhkj-framework-starter-sso-client` | 统一单点登录客户端、OAuth 2.1 换 Token、URI 规整防绕过 | `SsoClientFilter`, `SsoContextHolder`, `SsoUserContext` |
| **安全技术** | `zhkj-framework-starter-crypto` | 引导期配置解密 ENC(...)、持久层国密加密与盲索引引擎 | `EncryptablePropertyEnvironmentPostProcessor`, `BlindIndexUtil`, `SM4Util` |
| **中间件** | `zhkj-framework-starter-redis` | 分布式缓存管理器、声明式分布式锁、持有者校验 | `RedisAutoCacheManager`, `DistributedLockAspect`, `RedisSerializer.json()` |
| **中间件** | `zhkj-framework-starter-kafka` | 消息队列收发统一发布订阅契约、上下文双向透传与清理 | `ZhkjKafka`, `KafkaContextProducerInterceptor`, `KafkaTraceRecordInterceptor` |
| **中间件** | `zhkj-framework-starter-nacos` | Nacos 注册发现与配置动态热更新 | `ZhkjNacosAutoConfiguration`, `NacosProperties` |
| **基础设施** | `zhkj-framework-starter-openapi` | Knife4j 4.5.x + OpenAPI 3.0 在线接口文档集成 | `OpenApiAutoConfiguration`, `OpenApiProperties` |
| **工程根** | `zhkj-framework` | 统一依赖版本与构建插件治理 (BOM) | `pom.xml` (所有第三方依赖版本 100% 集中纳管) |

---

## 🛡️ 3. 架构设计黄金法则

### 3.1 严格单向依赖与职责纯粹
1. **依赖拓扑**: `core` $\rightarrow$ `data-api` $\rightarrow$ `starters` $\rightarrow$ `starter-web` $\rightarrow$ `业务微服务`。严禁反向依赖与循环依赖。
2. **根 POM 依赖洁癖**: 根 POM `<dependencies>` 严禁声明任何业务 Starter 或运行时监控包，仅保留编译期增强注解器（`lombok`、`spring-boot-configuration-processor`）。
3. **依赖版本全部纳管**: 所有外部第三方依赖版本必须收敛至根 `pom.xml` 的 `<properties>` 与 `<dependencyManagement>`，子模块严禁声明写死的 `<version>`。

### 3.2 实体多态与持久层解耦
1. **契约抽象**: `zhkj-framework-data-api` 仅提供 `IAuditEntity`、`ITenantEntity` 契约接口以及无第三方注解的纯 POJO 基类（`PojoBaseEntity`、`PojoTenantEntity`）。
2. **ORM 专属基类**:
   - MyBatis-Plus 实体继承 `MybatisAuditEntity` / `MybatisTenantEntity`；
   - JPA 实体继承 `JpaBaseEntity` / `JpaTenantEntity`；
   - 彻底杜绝强行将持久层专用注解绑架到底层通用基类的架构反模式。

### 3.3 全局上下文生命周期门面 (`FrameworkContext`)
1. **统一门面**: 租户、全局安全身份与 SSO 凭证由统一门面 `FrameworkContext`（位于 `core` 模块）统一调度。
2. **显式传递与清理**: 废弃在线程池环境下不安全的隐式 `InheritableThreadLocal` 传递，统一采用 `FrameworkContext.capture()` 捕获快照，并使用 `try (ContextScope scope = FrameworkContext.openScope(snapshot))` 或 `FrameworkContext.runWith(snapshot, ...)` 进行受控执行与自动资源释放。
3. **彻底防泄漏**: Web 请求出口处由最高优先级过滤器（`FrameworkContextClearFilter`）在 `finally` 阶段兜底清空所有 6 大上下文（快照、脱敏、安全认证、多租户、单点登录、数据权限）。

### 3.4 MyBatis-Plus 拦截器链严格拓扑编排
为避免多个 Starter 动态注入拦截器时引发的顺序竞态，框架规定并锁定了拦截器物理链执行次序：
$$\mathbf{TenantLineInnerInterceptor} \longrightarrow \mathbf{MybatisDataScopeInterceptor} \longrightarrow \mathbf{PaginationInnerInterceptor}$$
确保物理级的多租户行级隔离绝对优先于租户内的数据权限过滤与分页插件。

### 3.5 严格工具类单数命名规范
1. **统一标准**: 框架所有工具类一律采用单数命名：`JsonUtil`、`IdUtil`、`DateUtil`、`StringUtil`、`DigestUtil`、`BeanUtil`、`PropUtil`。
2. **零容忍**: 严禁新建任何 `*Utils` 复数类，所有历史复数类已物理清理。

### 3.6 零业务概念反向渗透 (Zero Business Penetration)
1. **纯粹 SPI 抽象**: 底层框架 `zhkj-framework` 严禁硬编码具体业务系统代号（如 `ncap`）、具体催收表名或业务字段（如 `case_no`）。
2. **契约外置**: 任何业务定制点（如部门祖先拓扑计算、加密算法实现、租户规则），必须且只能通过 SPI 抽象接口（如 `DeptAncestorProvider`、`CryptoAlgorithmProvider`）由上层业务微服务实现并注入。

### 3.7 高频路径 GC 防护与正则预编译
1. **零动态正则编译**: 严禁在 MyBatis AST 拦截器、动态 SQL 构建或查询主链路中使用动态拼接并重新编译的正则表达式（如动态 `param.replaceAll("(?i)" + regex, "")`）。
2. **静态常量复用**: 必须统一提取为 `private static final Pattern` 预编译常量，消除在高并发催收跑批作业下的 Minor GC 堆内存抖动。

### 3.8 工具类架构级防御 (Sealed Utility Defense)
1. **不可继承与私有构造器**: 全框架所有工具类必须声明为 `public final class`，并显式提供私有构造器 `private XxxUtil() {}`，防范外部反射或实例化破坏。
2. **常量池类封装**: `StringPool` 与 `CharPool` 统一采用带私有构造器的 `public final class`，彻底根除常量接口（Constant Interface）反模式。
