# ZHKJ Framework - 任务清单与演进路线

> **当前版本**: 2026.4.1.0-SNAPSHOT  
> **最后更新**: 2026-09-11  
> **状态**: 全量 17 个框架核心 Starter 与示例工程全量编译与单测全部通过 (100% BUILD SUCCESS, 0 Errors, 0 Failures)

---

## ✅ 已完成框架里程碑 (Milestones)

### 1. 架构梳理与解耦改造
- [x] 多租户模块全面解耦（从 MyBatis 独立为 `zhkj-framework-starter-tenant`）
- [x] 数据权限模块重构（拆分为框架无关抽象 `starter-datascope` 与 MyBatis AST 拦截器实现）
- [x] Web 模块规范化重构（启动引导器更名为 `zhkj-framework-launcher` + 独立组件 `zhkj-framework-starter-web`，包名规范为 `io.github.ybb88.framework.web.*`）
- [x] 模块去重与瘦身优化（彻底清理 core 模块废弃类与 web/servlet 重复工具，消除 starter-web 与 starter-secure 循环依赖）
- [x] 数据访问统一抽象设计（建立 `zhkj-framework-data-api`，定义 `IAuditEntity` / `ITenantEntity` / `PojoBaseEntity`）
- [x] 声明式 HTTP 客户端演进（彻底废弃已停滞维护的 OpenFeign，全面采用 Spring 6+ HTTP Interfaces `@HttpExchange`，独立收敛为 `zhkj-framework-starter-http`）
- [x] 现代化技术栈全面治理（Spring 7 / Spring Boot 4.1.1 / Spring Batch 6.0.5 API 治理，消除全部过时与废弃编译器警告）
- [x] 数据安全与全生命周期加解密（`starter-crypto` 统一提供 Spring Boot 4 `EnvironmentPostProcessor` 配置解密、Jackson 动态脱敏、MyBatis `EncryptTypeHandler` 字段透明存储加密与 HMAC-SM3 盲索引模糊检索）
- [x] Nacos 模块彻底解耦与 Jasypt 下线（移除所有旧 Jasypt 与 RSA 配置加密逻辑，Nacos 专注服务治理与配置同步）
- [x] 全工程代码瘦身与规范工具类单数命名（删除历史 `base64/` 复制源码及空工具类，新增 `JsonUtil`、`IdUtil`、`DigestUtil`，消除常量继承反模式）
- [x] 清理全工程无用过时的脚本与冗余文档（删除 nacos 下 7 个脚手架旧脚本及 mybatis 冗余的旧说明）

### 2. 基础框架底座层四大核心加固 (已 100% 验收交付)
- [x] 数据权限 UPDATE / DELETE 行级 AST 改写闭环与 1=0 恒假防穿透（`starter-datascope` + `starter-mybatis`）
- [x] Spring Data JPA 国密存储加密对称性实现与四大生产级特性（`starter-jpa` + `starter-crypto`）
- [x] 统一安全上下文智能自适应互通与 Redis 分布式 Token 黑名单（`starter-secure` + `starter-sso-client`）
- [x] 核心通用单数工具类独立高覆盖率单测构建（`zhkj-framework-core`）
- [x] 通用单点登录客户端 SDK 研制（`starter-sso-client`：自动拦截未登录、OAuth2 授权码自动换 Token、Cookie/Header 双向同步、`SsoContextHolder` 线程上下文自动化注入）
- [x] 生产级优雅可重入分布式锁切面与注解 `@DistributedLock`（`starter-redis`：Redis Hash 原子 Lua 脚本、SpEL 动态 Key、锁在外事务在内外包设计、抢锁异常/返回null/方法降级三重失败策略）

### 3. DevOps、CI/CD 与全链路可观测性 (已 100% 验收交付)
- [x] 全链路分布式 TraceId 隐式自动染色与透传闭环（`core` 的 `TraceUtil` / `TraceConstant` + `starter-web` 的 `TraceIdFilter` / `MdcTaskDecorator` + `starter-http` 的 `HttpClientHeaderInterceptor` + `starter-kafka` 的 `ZhkjKafka` / `KafkaTraceRecordInterceptor`）
- [x] 多数据库方言兼容性验证套件（`starter-mybatis` 的 `DatabaseDialectCompatibilityTest` 覆盖 MySQL 8、PostgreSQL、DM8 达梦分页与 AST 方言转义改写）
- [x] 开箱即用轻量级演示示例工程（`samples/sample-monolith-boot` 单体架构与 `samples/sample-cloud-service` 微服务架构，端到端自动化测试全部绿灯）
- [x] GitHub Actions CI/CD 自动化流水线（`.github/workflows/ci.yml` 自动化执行测试与构建打包校验）

---

## 🚧 正在进行中 (In Progress)
- [ ] 多持久层自动化方言矩阵扩展与压测基准套件

---

## 📝 规划路线图 (Roadmap)

### 近期演进 (v2026.4.2)
- **多数据库适配测试**: 覆盖 PostgreSQL、Oracle、达梦（DM8）测试套件
- **链路监控增强**: 整合 SkyWalking / Micrometer Prometheus 自动装配

### 中远期演进 (v2026.5.x)
- **插件体系扩展**: 设计动态插件加载与热插拔 SPI 机制
- **代码生成器**: 基于 AST 与 FreeMarker 的企业级代码生成器
- **开发者脚手架**: 提供在线快速生成微服务模板的 CLI 脚手架
