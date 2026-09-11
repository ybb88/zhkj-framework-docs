# 📚 ZHKJ Framework 官方文档中心

欢迎查阅 **ZHKJ Framework** 企业级微服务技术底座官方文档中心！本中心为您提供完整的架构全景、全生命周期安全方案、开发者规范及排错手册。

> ✍️ **创建人与核心架构师**: Jason.Yang（274107098@qq.com）  

---

## 🧭 核心专题文档直达

```
docs/
├── architecture/                     # 📐 架构设计体系
│   ├── overview.md                   # 架构总览：分层拓扑、18个模块职责矩阵与架构底线
│   └── security.md                   # 安全架构：全生命周期数据保护（配置/脱敏/存储国密盲索引）与 SSO
└── guides/                           # 🛠️ 开发者与运维指引
    ├── quick-start.md                # 快速上手：5分钟搭建一个标准微服务应用
    ├── coding-spec.md                # 编码规范：工具类单数命名、现代 API 清单与红线规约
    └── troubleshooting.md            # 排错手册：开发与生产十大常见报错排查速查表 (FAQ)
```

| 专题分类 | 文档名称 | 核心内容提要 |
| :--- | :--- | :--- |
| **系统架构** | 📐 [架构总览与分层设计](architecture/overview.md) | 5 层物理/逻辑分层模型、18 个模块职责矩阵、单向依赖拓扑与持久层解耦 |
| **数据安全** | 🛡️ [数据全生命周期安全体系](architecture/security.md) | 配置态解密 `ENC(...)`、展示态动态脱敏 `@Desensitize`、存储态国密 SM4 加密与 HMAC-SM3 盲索引模糊检索、JJWT 0.12+ 验签与 SSO |
| **新手入门** | 🚀 [5分钟极速上手指南](guides/quick-start.md) | 创建微服务工程、引入起步依赖、主启动类、MyBatis-Plus 多租户实体、声明式 RPC 客户端实战 |
| **代码规范** | 📝 [开发者编码规范与技术红线](guides/coding-spec.md) | 工具类单数强制命名、消除常量接口反模式、现代 API 调用规约、`FrameworkContext` 统一上下文生命周期 |
| **运维排错** | 🔍 [常见问题与排错手册 (FAQ)](guides/troubleshooting.md) | 多租户越权排查、Spring Boot 4 松散绑定解密 ClassCast 避坑、Batch 6 启动报错、UTF-8 BOM 乱码等速查表 |

---

## 📦 框架 18 个核心 Starter 就近文档索引

点击可直接跳转至各个独立 Starter 根目录下的详细组件使用手册：

### 1. 核心与数据抽象层
- **[zhkj-framework-core](../zhkj-framework-core/README.md)** - 通用工具箱（JsonUtil/IdUtil/DateUtil/DigestUtil）、统一响应包装（`R<T>`）、统一 DTO/VO 基类、统一上下文门面（`FrameworkContext`）。
- **[zhkj-framework-data-api](../zhkj-framework-data-api/README.md)** - 框架无关数据访问抽象层，定义 `IAuditEntity`、`ITenantEntity`、`PojoBaseEntity` 及多数据源 SPI。

### 2. 引导启动与 Web/通信层
- **[zhkj-framework-launcher](../zhkj-framework-launcher/README.md)** - 统一微服务/单体应用引导启动器（`ZhkjApplication`），集成环境自动探测、耗时统计与 ASCII Banner。
- **[zhkj-framework-starter-web](../zhkj-framework-starter-web/README.md)** - Web 应用增强基础设施，提供统一 Controller 响应自动封装、全局异常捕获、XSS 防护、线程池及 TraceId 链路跟踪。
- **[zhkj-framework-starter-http](../zhkj-framework-starter-http/README.md)** - 现代化声明式 HTTP 客户端，基于 Spring 6+ HTTP Interfaces（`@HttpExchange`），支持跨服务 Header 透明透传与优雅降级。

### 3. 持久层与数据管控 Starters
- **[zhkj-framework-starter-mybatis](../zhkj-framework-starter-mybatis/README.md)** - MyBatis-Plus 深度整合 Starter，包含通用分页、`ZhkjMetaObjectHandler` 审计自动填充、多租户行级拦截、基于 AST 的动态数据权限拦截及国密存储加密适配。
- **[zhkj-framework-starter-jpa](../zhkj-framework-starter-jpa/README.md)** - Spring Data JPA 深度整合 Starter，提供 Hibernate 租户 Filter 自动装配与 `JpaTenantEntity`。
- **[zhkj-framework-starter-datasource](../zhkj-framework-starter-datasource/README.md)** - 动态多数据源组件，支持 `@DS` 注解、SpEL 表达式路由与 Hikari 连接池管理。
- **[zhkj-framework-starter-tenant](../zhkj-framework-starter-tenant/README.md)** - 框架无关多租户治理核心，支持基于 Header/Token 的租户上下文传递（`TenantContext`）与租户规则引擎 SPI。
- **[zhkj-framework-starter-datascope](../zhkj-framework-starter-datascope/README.md)** - 细粒度数据权限管控核心，提供 `@DataAuth` 注解、权限模型 `ScopeModel` 及框架通用 SPI（`ScopeModelHandler`、`DeptAncestorProvider`）。

### 4. 中间件与安全 Starters
- **[zhkj-framework-starter-secure](../zhkj-framework-starter-secure/README.md)** - 轻量级统一鉴权中心，采用 JJWT 0.12.5 现代安全套件，实现无状态 Token 签发与验签、方法级权限拦截（`@PreAuth`）及 Actuator 安全保护。
- **[zhkj-framework-starter-sso-client](../zhkj-framework-starter-sso-client/README.md)** - 通用单点登录客户端 Starter，未登录请求重定向认证收银台，OAuth 2.1 授权码无感兑换 Token，Cookie/Header 双向映射与上下文自动注入。
- **[zhkj-framework-starter-crypto](../zhkj-framework-starter-crypto/README.md)** - 全生命周期数据安全中心，提供 Spring Boot 4 官方原生 `EnvironmentPostProcessor` 配置文件敏感属性 `ENC(...)` 解密、Jackson 响应敏感字段动态脱敏（`@Desensitize`）以及国密 SM4/AES 存储加密引擎与盲索引。
- **[zhkj-framework-starter-redis](../zhkj-framework-starter-redis/README.md)** - 分布式缓存 Starter，采用官方推荐 `RedisSerializer.json()` 序列化与多级动态过期缓存管理器 `RedisAutoCacheManager`。
- **[zhkj-framework-starter-kafka](../zhkj-framework-starter-kafka/README.md)** - 分布式消息队列 Starter，提供 `ZhkjKafka` 统一发布订阅契约。
- **[zhkj-framework-starter-nacos](../zhkj-framework-starter-nacos/README.md)** - Nacos 配置中心与服务发现 Starter，集成微服务环境自动注册与动态配置热刷新。
- **[zhkj-framework-starter-openapi](../zhkj-framework-starter-openapi/README.md)** - API 在线接口文档 Starter，基于 Knife4j 4.5.0 与 OpenAPI 3.0 (Swagger 3)，开箱即用。

---

## 🌐 网页在线访问与 Nginx 部署

文档中心已集成开箱即用的 **Docsify** 网页渲染引擎，零编译直接支持网页交互访问、侧边栏折叠、全文搜索与代码一键复制：

1. **本地预览**:
   在当前 `docs/` 目录下执行任意静态服务器命令即可（如 `npx docsify-cli serve` 或通过 Python `python -m http.server 3000`）。
2. **Nginx 部署**:
   直接将 `docs/` 目录挂载到 Nginx，配置模板参见：[nginx.conf](nginx.conf)。

---

## 🔄 路线图与版本演进
- 📋 [任务清单与演进路线 (TODO)](TODO.md)
- 🔄 [版本迁移指南 (MIGRATION)](MIGRATION.md)
