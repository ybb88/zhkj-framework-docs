# 📚 ZHKJ Framework 官方文档中心

欢迎查阅 **ZHKJ Framework** 企业级微服务技术底座官方文档中心。本中心提供架构全景、17 个框架模块的组件手册、数据安全方案、开发者规范及排错手册。

> ✍️ **创建人与核心架构师**: Jason.Yang（274107098@qq.com）  
> 🏷️ **最新版本**: 2026.4.1.0-SNAPSHOT  
> ☕ **适用基线**: OpenJDK 21 (LTS), Spring Boot 4.1.1, Spring Cloud 2025.1.2, Jackson 3.x（以根 `pom.xml` 为准）

---

## 🧭 核心专题文档全景导航

文档中心分为四大板块：

```
docs/
├── architecture/                     # 📐 架构设计体系（面向架构师与资深开发者）
│   ├── overview.md                   # 1. 架构总览：分层模型、17 个框架模块职责矩阵、依赖拓扑、架构底线
│   ├── zhkj-framework-refactor-blueprint.md # 2. 重构全景图：重构问题清单与实施现状核对
│   ├── security.md                   # 3. 安全架构：全生命周期数据保护（配置/脱敏/存储加密/盲索引）与 SSO
│   └── data-isolation.md             # 4. 数据隔离：多租户行级隔离、拦截器顺序与 AST 数据权限
├── starters/                         # 📦 核心起步依赖手册（17 个框架模块官方指南）
│   ├── core.md                       # 通用基础核心库 (R<T>, 工具箱, 上下文门面)
│   ├── data-api.md                   # 持久层数据抽象 (实体契约, 维度标记, SPI)
│   ├── launcher.md                   # 应用引导启动器 (ZhkjApplication, 环境探测, 耗时统计)
│   ├── starter-web.md                # Web MVC 增强 (统一封装, 异常映射, XSS, 内部 RPC 脱敏旁路)
│   ├── starter-http.md               # 声明式 HTTP 客户端 (@HttpExchange RPC, Header 透传)
│   ├── starter-mybatis.md            # MyBatis-Plus 持久层 (审计填充, 存储加密, 拦截器)
│   ├── starter-jpa.md                # Spring Data JPA 整合 (自动租户 Filter, 实体写入校验)
│   ├── starter-datasource.md         # 动态多数据源路由 (@DS 注解, 连接池管理)
│   ├── starter-tenant.md             # 多租户治理核心 (TenantContext, 租户规则 SPI)
│   ├── starter-datascope.md          # 细粒度数据权限 (@DataAuth, AST 改写, 部门 SPI)
│   ├── starter-crypto.md             # 全生命周期国密安全 (ENC 配置解密, 存储加密, 盲索引)
│   ├── starter-secure.md             # 无状态安全鉴权中心 (JJWT 0.12, @PreAuth 权限)
│   ├── starter-sso-client.md         # 统一单点登录客户端 (授权码换 Token, URI 清洗)
│   ├── starter-redis.md              # 分布式缓存与锁 (RedisSerializer.json, @DistributedLock)
│   ├── starter-kafka.md              # 分布式消息队列 (ZhkjKafka 契约, 上下文透传与清理)
│   ├── starter-nacos.md              # 服务注册与配置中心 (服务发现接入)
│   └── starter-openapi.md            # API 在线接口文档 (Knife4j 4.5.0 + OpenAPI 3)
├── guides/                           # 🛠️ 开发者与运维指引（给业务研发，开箱即用）
│   ├── quick-start.md                # 1. 快速上手：搭建一个标准微服务应用
│   ├── coding-spec.md                # 2. 编码规范：工具类命名、常量类形式、Anti-FQCN、现代 API 与存量清单
│   └── troubleshooting.md            # 3. 排错手册：开发与生产 17 类常见问题排查速查表 (FAQ)
└── evolution/                        # 🔄 架构决策与演进规划（系统治理全生命周期）
    ├── adr-history.md                # 1. 决策记录：重大架构决策记录 (ADR 001-009) 与实施现状
    ├── migration.md                  # 2. 版本迁移：Spring Boot 4 / JDK 21 升级指引
    └── roadmap.md                    # 3. 演进路线：历史里程碑与近期、中远期规划
```

---

## 📐 核心架构专题索引

| 专题分类 | 文档名称 | 核心内容提要 |
| :--- | :--- | :--- |
| **系统架构** | 📐 [架构总览与分层设计](architecture/overview.md) | 分层模型、17 个框架模块职责矩阵、依赖拓扑（以 pom 为准）、示例工程说明 |
| **重构作战** | 🗺️ [2026 全项目重构作战全景图](architecture/zhkj-framework-refactor-blueprint.md) | 重构问题清单（加解密种子、SM2 字符集、Kafka 清理、正则预编译、ThreadLocal）与实施现状 |
| **数据安全** | 🛡️ [数据全生命周期安全体系](architecture/security.md) | 配置态解密 `ENC(...)`、展示态脱敏与内部 RPC 旁路、存储加密与盲索引、统一 SSO |
| **数据隔离** | 🏢 [多租户隔离与数据权限体系](architecture/data-isolation.md) | 租户行级隔离、拦截器顺序、AST 动态 SQL 改写、`DeptAncestorProvider` SPI |
| **新手入门** | 🚀 [5分钟极速上手指南](guides/quick-start.md) | 创建微服务工程、引入起步依赖、实体继承、CRUD、声明式 RPC、分布式锁实战 |
| **代码规范** | 📝 [开发者编码规范与技术红线](guides/coding-spec.md) | 工具类命名、常量类形式、Anti-FQCN、现代 API、存量待治理清单 |
| **运维排错** | 🔍 [常见问题与排错手册 (FAQ)](guides/troubleshooting.md) | 多租户、松散绑定解密、RPC 脱敏旁路、分布式锁异常映射等 17 类场景 |
| **架构决策** | 📜 [架构决策全记录 (ADR History)](evolution/adr-history.md) | ADR 001-009 决策记录，每条附实施现状注记 |
| **版本升级** | 🔄 [版本升级迁移指南 (MIGRATION)](evolution/migration.md) | 从 Spring Boot 2.x/3.x 与 Feign 迁移至当前版本的操作指南 |
| **演进规划** | 🗺️ [任务清单与演进路线图 (Roadmap)](evolution/roadmap.md) | 历史里程碑、近期与中远期规划 |

---

## 📦 核心起步依赖手册索引 (Starters Manual)

### 1. 核心底座与数据抽象层
- **[通用核心基础库 (core)](starters/core.md)** - 工具箱（JsonUtil/IdUtil/DateUtil/DigestUtil/StringUtil/BlindIndexUtil）、统一响应（`R<T>`）、上下文门面（`FrameworkContext`）。
- **[持久层数据抽象 (data-api)](starters/data-api.md)** - 框架无关数据访问抽象，定义 `IEntity<ID>`、`IAuditEntity<ID>`、维度标记（`ITenantEntity`、`ILogicDeleteEntity`、`IVersionEntity`）及审计用户 SPI。
- **[应用引导启动器 (launcher)](starters/launcher.md)** - 统一启动入口 `ZhkjApplication`，集成环境探测与启动耗时统计。

### 2. 表现层与远程通信
- **[Web MVC 核心增强 (starter-web)](starters/starter-web.md)** - 响应封装、全局异常处理、XSS 规整防护、TraceId 过滤器与内部 RPC 脱敏旁路过滤器（`InternalRpcDesensitizeFilter`）。
- **[声明式 HTTP 客户端 (starter-http)](starters/starter-http.md)** - 基于 Spring 6+ HTTP Interfaces（`@HttpExchange`），支持跨服务 Header 透传与降级。

### 3. 持久层与数据管控
- **[MyBatis-Plus 持久层 (starter-mybatis)](starters/starter-mybatis.md)** - 审计填充（`ZhkjMetaObjectHandler`）、租户行级拦截、数据权限拦截、存储加密适配。
- **[Spring Data JPA 整合 (starter-jpa)](starters/starter-jpa.md)** - `JpaTenantEntity` 直接绑定 Hibernate 自动过滤器，`JpaTenantIdResolver` 统一解析有效租户；无需 MVC/OSIV，覆盖查询、数据库主键加载与实体新增/更新校验。
- **[动态多数据源路由 (starter-datasource)](starters/starter-datasource.md)** - 集成 dynamic-datasource，支持 `@DS` 路由与 ShardingSphere 路由辅助。
- **[多租户治理核心 (starter-tenant)](starters/starter-tenant.md)** - 框架无关租户上下文（`TenantContext`）、租户解析与规则 SPI。
- **[细粒度数据权限 (starter-datascope)](starters/starter-datascope.md)** - `@DataAuth` 注解、`DataScopeModel`、JSqlParser AST 改写与 `DeptAncestorProvider` SPI。

### 4. 安全认证与国密加密
- **[全生命周期国密安全 (starter-crypto)](starters/starter-crypto.md)** - 配置属性 `ENC(...)` 解密、存储加密引擎（`StorageCryptoEngine`）、盲索引注解（`@EncryptField` / `@BlindIndex`）与接口报文加解密。
- **[无状态安全鉴权中心 (starter-secure)](starters/starter-secure.md)** - JJWT 0.12 无状态 Token、`@PreAuth` 方法级权限与 Actuator 安全配置。
- **[统一单点登录客户端 (starter-sso-client)](starters/starter-sso-client.md)** - 未登录重定向、授权码换 Token、URI 路径规整。

### 5. 中间件与基础设施
- **[分布式缓存与锁 (starter-redis)](starters/starter-redis.md)** - `RedisSerializer.json()` 序列化、`@DistributedLock` 声明式切面与 Lua 持有者校验。
- **[分布式消息队列 (starter-kafka)](starters/starter-kafka.md)** - `ZhkjKafka` 发布订阅契约，上下文与 TraceId 透传，消费端清理。
- **[服务注册与配置中心 (starter-nacos)](starters/starter-nacos.md)** - Nacos 服务发现与配置中心接入（`ZhkjNacosAutoConfiguration`）。
- **[API 在线接口文档 (starter-openapi)](starters/starter-openapi.md)** - Knife4j 4.5.0 与 OpenAPI 3。

---

## 🌐 网页在线访问与 Nginx 部署

文档中心使用 **Docsify** 渲染（`docs/index.html`，依赖 CDN）：

1. **本地预览**: 在 `docs/` 目录下执行任意静态服务器命令（如 `npx docsify-cli serve` 或 `python -m http.server 3000`）。
2. **Nginx 部署**: 将 `docs/` 目录挂载到 Nginx，配置模板参见 [nginx.conf](nginx.conf)。
