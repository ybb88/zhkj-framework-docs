# 📚 ZHKJ Framework 官方文档中心

欢迎查阅 **ZHKJ Framework** 企业级微服务技术底座官方文档中心！本中心为您提供完整的架构全景、全生命周期安全方案、开发者规范及排错手册。

> ✍️ **创建人与核心架构师**: Jason.Yang（274107098@qq.com）  

---

## 🧭 核心专题文档全景导航

文档中心采用“**三纵三横**”清晰结构，分为三大核心板块，满足不同角色快速查阅需求：

```
docs/
├── architecture/                     # 📐 架构设计体系（给架构师与资深开发者）
│   ├── overview.md                   # 1. 架构总览：分层拓扑、18个模块职责矩阵与架构底线
│   ├── security.md                   # 2. 安全架构：全生命周期数据保护（配置/脱敏/存储国密盲索引）与 SSO
│   └── data-isolation.md             # 3. 数据隔离：多租户物理/行级隔离、拦截器顺序与 AST 数据权限
├── guides/                           # 🛠️ 开发者与运维指引（给业务研发，开箱即用）
│   ├── quick-start.md                # 1. 快速上手：5分钟搭建一个标准微服务应用
│   ├── coding-spec.md                # 2. 编码规范：工具类单数命名、现代 API 清单与红线规约
│   └── troubleshooting.md            # 3. 排错手册：开发与生产 16 大常见报错排查速查表 (FAQ)
└── evolution/                        # 🔄 架构决策与演进规划（系统治理全生命周期）
    ├── adr-history.md                # 1. 决策记录：重大架构决策记录 (ADR 001-009)
    ├── migration.md                  # 2. 版本迁移：Spring Boot 4 / JDK 21 升级平滑指引
    └── roadmap.md                    # 3. 演进路线：近期与中远期里程碑任务清单
```

| 专题分类 | 文档名称 | 核心内容提要 |
| :--- | :--- | :--- |
| **系统架构** | 📐 [架构总览与分层设计](architecture/overview.md) | 5 层拓扑模型、18 个模块矩阵、单向依赖拓扑与根 POM 依赖卫生 |
| **数据安全** | 🛡️ [数据全生命周期安全体系](architecture/security.md) | 配置态解密 `ENC(...)`、展示态动态脱敏与内部 RPC 旁路放行、存储态国密 SM4 加密与盲索引等值检索、统一 SSO |
| **数据隔离** | 🏢 [多租户隔离与数据权限体系](architecture/data-isolation.md) | 双模自愈探测、拦截器链拓扑锁定（Tenant -> DataScope -> Pagination）、AST 动态 SQL 重写 |
| **新手入门** | 🚀 [5分钟极速上手指南](guides/quick-start.md) | 创建微服务工程、引入起步依赖、实体继承、CRUD、声明式 RPC、分布式锁实战 |
| **代码规范** | 📝 [开发者编码规范与技术红线](guides/coding-spec.md) | 工具类单数命名规范、消除常量接口、Anti-FQCN 绝对铁律、DigestUtil 字节流、显式 ThreadLocal 传递 |
| **运维排错** | 🔍 [常见问题与排错手册 (FAQ)](guides/troubleshooting.md) | 涵盖多租户越权、松散绑定解密、RPC 脱敏旁路、Anti-FQCN、分布式锁异常映射等 16 大场景 |
| **架构决策** | 📜 [架构决策全记录 (ADR History)](evolution/adr-history.md) | 详细记录 Feign 废弃、Jackson 3 原生重构、国密改造、RPC 脱敏旁路与 Anti-FQCN (ADR 001-009) |
| **版本升级** | 🔄 [版本升级迁移指南 (MIGRATION)](evolution/migration.md) | 从 Spring Boot 2.x/3.x 与 Feign 平滑平移至当前版本的实操手册 |
| **演进规划** | 🗺️ [任务清单与演进路线图 (Roadmap)](evolution/roadmap.md) | 历史里程碑验收清单、近期适配测试与中远期 CLI 脚手架规划 |

---

## 📦 框架 18 个核心 Starter 就近文档索引

点击可直接跳转至各个独立 Starter 根目录下的详细组件使用手册：

### 1. 核心与数据抽象层
- **[zhkj-framework-core](../zhkj-framework-core/README.md)** - 通用工具箱（JsonUtil/IdUtil/DateUtil/DigestUtil/BlindIndexUtil）、统一响应包装（`R<T>`）、统一 DTO/VO 基类、统一上下文门面（`FrameworkContext`）。
- **[zhkj-framework-data-api](../zhkj-framework-data-api/README.md)** - 框架无关数据访问抽象层，定义 `IAuditEntity`、`ITenantEntity`、`PojoBaseEntity` 及多数据源 SPI。

### 2. 引导启动与 Web/通信层
- **[zhkj-framework-launcher](../zhkj-framework-launcher/README.md)** - 统一微服务/单体应用引导启动器（`ZhkjApplication`），集成环境自动探测、耗时统计与 ASCII Banner。
- **[zhkj-framework-starter-web](../zhkj-framework-starter-web/README.md)** - Web 应用增强基础设施，提供统一 Controller 响应自动封装、全局异常捕获、XSS 规整防护、TraceId 链路跟踪与微服务内部 RPC 脱敏旁路过滤器（`InternalRpcDesensitizeFilter`）。
- **[zhkj-framework-starter-http](../zhkj-framework-starter-http/README.md)** - 现代化声明式 HTTP 客户端，基于 Spring 6+ HTTP Interfaces（`@HttpExchange`），支持跨服务 Header 透明透传与优雅降级。

### 3. 持久层与数据管控 Starters
- **[zhkj-framework-starter-mybatis](../zhkj-framework-starter-mybatis/README.md)** - MyBatis-Plus 深度整合 Starter，包含通用分页、`ZhkjMetaObjectHandler` 审计自动填充、多租户行级拦截、基于 AST 的动态数据权限拦截及国密存储加密适配。
- **[zhkj-framework-starter-jpa](../zhkj-framework-starter-jpa/README.md)** - Spring Data JPA 深度整合 Starter，提供 Hibernate 租户 Filter 自动装配与 `JpaTenantEntity`。
- **[zhkj-framework-starter-datasource](../zhkj-framework-starter-datasource/README.md)** - 动态多数据源组件，支持 `@DS` 注解、SpEL 表达式路由与 Hikari 连接池管理。
- **[zhkj-framework-starter-tenant](../zhkj-framework-starter-tenant/README.md)** - 框架无关多租户治理核心，支持基于 Header/Token 的租户上下文传递（`TenantContext`）与租户规则引擎 SPI。
- **[zhkj-framework-starter-datascope](../zhkj-framework-starter-datascope/README.md)** - 细粒度数据权限管控核心，提供 `@DataAuth` 注解、权限模型 `ScopeModel` 及框架通用 SPI（`ScopeModelHandler`、`DeptAncestorProvider`）。

### 4. 中间件与安全 Starters
- **[zhkj-framework-starter-secure](../zhkj-framework-starter-secure/README.md)** - 轻量级统一鉴权中心，采用 JJWT 0.12.5 现代安全套件，实现无状态 Token 签发与验签、方法级权限拦截（`@PreAuth`）及 Actuator 安全保护。
- **[zhkj-framework-starter-sso-client](../zhkj-framework-starter-sso-client/README.md)** - 通用单点登录客户端 Starter，未登录请求重定向认证收银台，OAuth 2.1 授权码无感兑换 Token，URI 路径安全清洗防绕过，Cookie/Header 双向映射与上下文自动注入。
- **[zhkj-framework-starter-crypto](../zhkj-framework-starter-crypto/README.md)** - 全生命周期数据安全中心，提供 Spring Boot 4 官方原生 `EnvironmentPostProcessor` 配置文件敏感属性 `ENC(...)` 解密、实体国密 SM4/AES 存储加密引擎（`StorageCryptoEngine`）与加盐哈希盲索引声明（`@EncryptField` / `@BlindIndex`）。
- **[zhkj-framework-starter-redis](../zhkj-framework-starter-redis/README.md)** - 分布式缓存 Starter，采用官方推荐 `RedisSerializer.json()` 序列化与多级动态过期缓存管理器 `RedisAutoCacheManager`，内置 `@DistributedLock` 声明式切面与 Lua 持有者校验防误删。
- **[zhkj-framework-starter-kafka](../zhkj-framework-starter-kafka/README.md)** - 分布式消息队列 Starter，提供 `ZhkjKafka` 统一发布订阅契约，全链路隐式透传租户、用户身份与 TraceId。
- **[zhkj-framework-starter-nacos](../zhkj-framework-starter-nacos/README.md)** - Nacos 配置中心与服务发现 Starter，集成微服务环境自动注册与动态配置热刷新。
- **[zhkj-framework-starter-openapi](../zhkj-framework-starter-openapi/README.md)** - API 在线接口文档 Starter，基于 Knife4j 4.5.0 与 OpenAPI 3.0 (Swagger 3)，开箱即用。

---

## 🌐 网页在线访问与 Nginx 部署

文档中心已集成开箱即用的 **Docsify** 网页渲染引擎，零编译直接支持网页交互访问、侧边栏折叠、全文搜索与代码一键复制：

1. **本地预览**:
   在当前 `docs/` 目录下执行任意静态服务器命令即可（如 `npx docsify-cli serve` 或通过 Python `python -m http.server 3000`）。
2. **Nginx 部署**:
   直接将 `docs/` 目录挂载到 Nginx，配置模板参见：[nginx.conf](nginx.conf)。
