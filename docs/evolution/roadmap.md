# ZHKJ Framework 任务清单与演进路线图 (Roadmap)

> **当前版本**: 2026.4.1.0-SNAPSHOT  
> **框架模块**: 17 个（`zhkj-framework-*`，见根 `pom.xml` 的 `<modules>`）；示例工程 `samples/`（不在根 reactor 中）  
> **历史验收记录**: 早期记录中的 “全量构建 BUILD SUCCESS / 双工程模块全绿” 结论，本次文档核对未能复现，构建结果以 CI（`.github/workflows/ci.yml`）为准  
> **核心架构师**: Jason.Yang（274107098@qq.com）  

---

## ✅ 已完成框架重大里程碑 (Milestones)

### 1. 架构解耦与现代化底座
- [x] 多租户模块全面解耦（独立为 `zhkj-framework-starter-tenant`）
- [x] 数据权限模块重构（拆分为框架无关抽象 `starter-datascope` 与 MyBatis AST 拦截器实现）
- [x] Web 模块规范化重构（统一引导入口 `launcher` + 独立组件 `starter-web`，包名为 `io.github.ybb88.framework.web.*`）
- [x] 数据访问统一抽象设计（建立 `zhkj-framework-data-api`，定义 `IAuditEntity` / `ITenantEntity` / `PojoBaseEntity`）
- [x] 声明式 HTTP 客户端演进（废弃 OpenFeign，采用 Spring 6+ HTTP Interfaces `@HttpExchange`，实现于 `zhkj-framework-starter-http`）
- [x] 现代化技术栈治理（Spring Boot 4.1.1 基线；Spring Framework 版本以 Boot BOM 为准，根 POM 注释写作 6.x，`[待确认]`）
- [x] 数据安全与全生命周期加解密（`starter-crypto` 提供 `EnvironmentPostProcessor` 配置解密、Jackson 动态脱敏、MyBatis `EncryptTypeHandler` 透明加密与 HMAC 盲索引检索）
- [x] Nacos 模块解耦与 Jasypt 下线（`starter-nacos` 仅负责服务治理与配置接入，不含配置加解密逻辑）
- [x] 工具类单数命名与历史代码清理（`JsonUtil`、`IdUtil`、`DigestUtil` 等现行类，`StringPool`/`CharPool` 为 final 类）
- [x] Jackson 3 现代化升级（`tools.jackson.*`，`JacksonConfiguration` 中使用 `DateTimeFeature`）
- [x] 根 POM 依赖纳管（框架内部模块与第三方版本集中于根 POM；子模块 POM 未发现字面量版本）
  > 已完成清理：经检索框架与 NCAP 全部 POM，引用未定义版本属性的 Batik（`batik-all` / `batik-svgpp`）、XML Graphics Commons、XML Security、XXL-Job、P6Spy 均无消费声明，对应六项依赖管理已删除，不补猜版本。

### 2. 安全合规与生产级稳定性
- [x] 数据权限 UPDATE / DELETE 行级 AST 改写与恒假条件防穿透（`starter-datascope` + `starter-mybatis`）
- [x] Spring Data JPA 国密存储加密（`starter-jpa` + `starter-crypto`，`CryptoAttributeConverter`）
- [x] 统一安全上下文与 Redis Token 黑名单（`starter-secure` 的 `RedisTokenBlacklistService` + `starter-sso-client`）
- [x] 通用单点登录客户端（`starter-sso-client`：`cleanPath` 路径规整、授权码换 Token、仅请求头令牌认证（不读取 Cookie）、`SsoUserContext` 继承 `ZhkjUser`）
- [x] 分布式锁切面与注解 `@DistributedLock`（`starter-redis`：Lua 持有者校验、SpEL Key、失败抛出 `DistributedLockException`）
- [x] MyBatis-Plus 拦截器顺序编排（`Tenant -> DataScope -> Custom -> Pagination`，见 `overview.md` 第 3.4 节）
- [x] 系统性硬伤治理（默认加解密种子已移除；SM2 与 Redis 监听使用 UTF-8；`SqlKeyword` 正则静态预编译；Kafka `failure` 回调清理上下文；`StringPool`/`CharPool` 改为 final 类）
- [x] 微服务内部 RPC 脱敏边界收敛：`InternalRpcDesensitizeFilter`（`HIGHEST_PRECEDENCE + 20`）仅接受服务端 `RpcRequestAttribute.DESENSITIZE_AUTHORIZED == Boolean.TRUE`；旧 `X-Internal-RPC: true` 仅为标记，普通 RPC 与公共展示端点默认脱敏。宿主迁移须先认证和授权目标内部端点，再受控设置请求属性并在 `finally` 删除（见 `starter-web.md` 第 11 节）。
- [x] 委托权限独立为服务端 `RpcRequestAttribute.TENANT_DELEGATION_AUTHORIZED`，机器认证不等于跨租户授权；网关在认证、匿名及 disabled 路径均剥离三个内部头。此里程碑描述代码行为，不替代部署边界验收。
- [x] Anti-FQCN 规约与摘要字节流（`DigestUtil.sha256(byte[])`）

### 3. 全链路可观测性与上下文闭环
- [x] 全链路 TraceId 透传（`core` 的 `TraceUtil` + `starter-web` 的 `TraceIdFilter` / `MdcTaskDecorator` + `starter-http` 的 `HttpClientHeaderInterceptor` + `starter-kafka` 的 `ZhkjKafka` / `KafkaTraceRecordInterceptor`）
- [x] Kafka 上下文穿透（生产端拦截器写入 Header，消费端拦截器还原并在 `afterRecord` 清理）
- [x] 多数据库方言兼容性测试套件（`zhkj-framework-starter-mybatis/src/test` 中的 `DatabaseDialectCompatibilityTest` 覆盖 MySQL 8、PostgreSQL、DM8 达梦分页与 AST 方言改写）
- [x] 示例工程（`samples/sample-monolith-boot` 单体示例与 `samples/sample-cloud-service` 微服务示例）

---

## 🚧 近期演进计划 (v2026.4.2)

- [ ] **多数据库方言压测套件**: 覆盖 Oracle、达梦（DM8）、人大金仓（KingbaseES）高并发分页与 AST 重写基准测试
- [ ] **全链路指标监控**: 增加 Micrometer Prometheus 与 Grafana 自动化 Dashboard 仪表盘模板
- [ ] **批处理增强（Spring Batch 6）**: 完善分片读写与动态 JobOperator 操作端点（框架 pom 当前未引入 Spring Batch，承载位置待确认）

---

## 🔭 中远期规划 (v2026.5.x)

- **动态插件体系**: 设计基于 Java SPI 与 ClassLoader 隔离的动态热插拔扩展机制
- **企业级低代码代码生成器**: 基于 AST 与 FreeMarker 打造支持国密加密、多租户与数据权限的代码生成器
- **开发者 CLI 脚手架**: 提供交互式终端脚手架命令 `zhkj-cli`，用于拉取业务微服务脚手架
