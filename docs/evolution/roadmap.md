# ZHKJ Framework 任务清单与演进路线图 (Roadmap)

> **当前版本**: 2026.4.1.0-SNAPSHOT  
> **验证基准**: 全量 18 个框架核心 Starter 与示例工程编译测试全部通过 (100% BUILD SUCCESS, 0 Errors, 0 Failures)  
> **核心架构师**: Jason.Yang（274107098@qq.com）  

---

## ✅ 已完成框架重大里程碑 (Milestones)

### 1. 架构解耦与现代化底座
- [x] 多租户模块全面解耦（独立为 `zhkj-framework-starter-tenant`）
- [x] 数据权限模块重构（拆分为框架无关抽象 `starter-datascope` 与 MyBatis AST 拦截器实现）
- [x] Web 模块规范化重构（统一引导器 `launcher` + 独立组件 `starter-web`，包名规范为 `io.github.ybb88.framework.web.*`）
- [x] 数据访问统一抽象设计（建立 `zhkj-framework-data-api`，定义 `IAuditEntity` / `ITenantEntity` / `PojoBaseEntity`）
- [x] 声明式 HTTP 客户端演进（彻底废弃已停滞维护的 OpenFeign，全面采用 Spring 6+ HTTP Interfaces `@HttpExchange`，独立收敛为 `zhkj-framework-starter-http`）
- [x] 现代化技术栈全面治理（Spring 7 / Spring Boot 4.1.1 / Spring Batch 6.0.5 API 治理，消除全部过时与废弃编译器警告）
- [x] 数据安全与全生命周期加解密（`starter-crypto` 统一提供 Spring Boot 4 `EnvironmentPostProcessor` 配置解密、Jackson 动态脱敏、MyBatis `EncryptTypeHandler` 字段透明存储加密与 HMAC-SM3 盲索引模糊检索）
- [x] Nacos 模块彻底解耦与 Jasypt 下线（移除所有旧 Jasypt 与 RSA 配置加密逻辑，Nacos 专注服务治理与配置同步）
- [x] 全工程代码瘦身与规范工具类单数命名（删除历史 `base64/` 复制源码及空工具类，新增 `JsonUtil`、`IdUtil`、`DigestUtil`，消除常量继承反模式）
- [x] 全模块 Jackson 3 原生现代化升级（`tools.jackson.core:jackson-databind` 3.x 深度重构，废弃类与方法全面替代）
- [x] 根 POM 依赖卫生规约落地（所有第三方外部依赖版本 100% 集中由根 POM `<properties>` 与 `<dependencyManagement>` 纳管，子模块 POM 零硬编码版本）

### 2. 安全合规与生产级稳定性闭环
- [x] 数据权限 UPDATE / DELETE 行级 AST 改写闭环与 1=0 恒假防穿透（`starter-datascope` + `starter-mybatis`）
- [x] Spring Data JPA 国密存储加密对称性实现（`starter-jpa` + `starter-crypto`）
- [x] 统一安全上下文自适应互通与 Redis 分布式 Token 黑名单（`starter-secure` + `starter-sso-client`）
- [x] 通用单点登录客户端 SDK 研制（`starter-sso-client`：URI 白名单 `cleanPath` 安全清洗防绕过、OAuth2 授权码自动换 Token、Cookie/Header 双向映射、`SsoUserContext` 归一化继承 `ZhkjUser`）
- [x] 生产级声明式分布式锁切面与注解 `@DistributedLock`（`starter-redis`：Lua 脚本持有者 Owner 校验防误删、SpEL 动态 Key、抢锁失败强类型映射为 `BaseFrameworkException` 受控 400 响应）
- [x] MyBatis-Plus 拦截器严格拓扑编排（强制保持 `Tenant -> DataScope -> Pagination` 执行顺序，彻底杜绝动态注册竞态）

### 3. 全链路可观测性与上下文闭环
- [x] 全链路分布式 TraceId 隐式自动染色与透传闭环（`core` 的 `TraceUtil` + `starter-web` 的 `TraceIdFilter` / `MdcTaskDecorator` + `starter-http` 的 `HttpClientHeaderInterceptor` + `starter-kafka` 的 `ZhkjKafka` / `KafkaTraceRecordInterceptor`）
- [x] Kafka 消息队列全链路租户与操作人上下文穿透（生产端拦截器写入 Headers，消费端拦截器还原并在 finally 彻底清理防线程复用污染）
- [x] 多数据库方言兼容性验证套件（`starter-mybatis` 覆盖 MySQL 8、PostgreSQL、DM8 达梦分页与 AST 方言转义改写）
- [x] 开箱即用轻量级演示工程（`samples/sample-monolith-boot` 单体架构与 `samples/sample-cloud-service` 微服务架构）

---

## 🚧 近期演进计划 (v2026.4.2)

- [ ] **多数据库方言压测套件**: 覆盖 Oracle、达梦（DM8）、人大金仓（KingbaseES）高并发分页与 AST 重写基准测试
- [ ] **全链路指标监控**: 增加 Micrometer Prometheus 与 Grafana 自动化 Dashboard 仪表盘模板
- [ ] **Spring Batch 6 批处理增强**: 完善分片读写与动态 JobOperator 操作端点

---

## 🔭 中远期规划 (v2026.5.x)

- **动态插件体系**: 设计基于 Java SPI 与 ClassLoader 隔离的动态热插拔扩展机制
- **企业级低代码代码生成器**: 基于 AST 与 FreeMarker 打造支持国密加密、多租户与数据权限一键生成的代码生成器
- **开发者 CLI 脚手架**: 提供交互式终端脚手架命令 `zhkj-cli`，支持秒级拉取业务微服务脚手架
