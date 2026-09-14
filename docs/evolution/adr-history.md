# ZHKJ Framework 架构演进与决策记录全集 (ADR History)

> **文档定位**: 记录 `zhkj-framework` 从诞生到工业级成熟过程中所做出的全部重大架构决策（Architecture Decision Records, ADR）、设计依据与重构历程。  
> **核心架构师与创作者**: Jason.Yang（274107098@qq.com）  

---

## 🧭 架构演进背景与路线

`zhkj-framework` 经历多轮系统化演进，成功实现了向现代化技术栈（Java 21、Spring Boot 4.1.1、Spring Cloud 2025、Jackson 3）的彻底跨越，并彻底清零了历史架构异味。

```
┌─────────────────┐      ┌─────────────────┐      ┌─────────────────┐
│  Phase 1: 破旧立新 │ ───► │  Phase 2: 安全重构 │ ───► │  Phase 3: 极致纯粹 │
│  - 拥抱 Java 21  │      │  - 国密 SM4/SM3  │      │  - 单向依赖闭环 │
│  - 废弃 Feign   │      │  - 盲索引检索   │      │  - 拦截器竞态根除 │
│  - 升级 Boot 4  │      │  - 统一 SSO Client │      │  - 根 POM 依赖卫生│
└─────────────────┘      └─────────────────┘      └─────────────────┘
```

---

## 📑 重大架构决策记录 (ADR 清单)

### ADR-001: 彻底废弃 Spring Cloud OpenFeign，全面采用 Spring 6+ HTTP Interfaces
- **状态**: 已落地 (Accepted & Implemented)
- **背景**: OpenFeign 维护停滞，且对响应式与 Spring 6 现代通信体系支持薄弱。
- **决策**: 
  1. 废弃 Feign 全套依赖，研发自研声明式客户端 `zhkj-framework-starter-http`；
  2. 基于 Spring 6+ `@HttpExchange`、`@GetExchange`、`@PostExchange` 现代注解构建 RPC；
  3. 内置 `HttpClientHeaderInterceptor`，在 Servlet 请求、Spring Batch、`@Async` 异步线程及 Kafka 消费端透明透传 `X-Tenant-Id`、`X-User-Id` 等全链路身份上下文。

---

### ADR-002: Jackson 3 现代化全面升级与 Jackson 2 兼容层治理
- **状态**: 已落地 (Accepted & Implemented)
- **背景**: 业界 JSON 序列化演进至 Jackson 3 (`tools.jackson.*`)，废弃了原 2.x 的很多低效同步阻塞逻辑。
- **决策**:
  1. 框架底层核心统一基于 `tools.jackson.core:jackson-databind` 3.x 构建；
  2. 废弃已淘汰的旧特性方法（如使用 `DateTimeFeature.WRITE_DATES_AS_TIMESTAMPS` 替代废弃的 `SerializationFeature`）；
  3. `RedisSerializer.json()` 统一使用现代化工厂，全面提升 JSON 解析吞吐量。

---

### ADR-003: 淘汰 Jasypt/RSA，构建全生命周期国密 SM4/AES 与 HMAC-SM3 盲索引安全体系
- **状态**: 已落地 (Accepted & Implemented)
- **背景**: 旧版配置解密采用第三方 Jasypt，算法陈旧且在 Spring Boot 4 启动阶段频发松散绑定 ClassCast 异常；持久层隐私数据明文存储或加密后无法检索。
- **决策**:
  1. 彻底移除 Jasypt 与 RSA，构建专属 `zhkj-framework-starter-crypto`；
  2. **配置态**: 基于 Boot 4 原生 `EnvironmentPostProcessor` 统一解密 `ENC(...)`；
  3. **展示态**: 统一 Jackson `@Desensitize` 动态打码（支持 10 种脱敏规则）；
  4. **存储态**: 采用国密 SM4/AES 透明加密（`EncryptTypeHandler`）+ HMAC-SM3 加盐哈希盲索引列（`*_hash`），由 `ZhkjMetaObjectHandler` 自动计算注入，支持数据库密文精准等值检索与模糊检索。

---

### ADR-004: 统一上下文生命周期门面 (`FrameworkContext`) 与 ThreadLocal 安全治理
- **状态**: 已落地 (Accepted & Implemented)
- **背景**: 原有多租户、安全身份、SSO 凭证分散在各自的 ThreadLocal 中，在线程池并发复用场景下容易出现串号或内存泄漏；使用 `InheritableThreadLocal` 容易在子线程复用时隐式继承污染。
- **决策**:
  1. 在 `core` 模块建立统一门面 `FrameworkContext`，采用显式 `ThreadLocal` 纳管快照；
  2. 跨线程统一使用 `FrameworkContext.capture()` 捕获快照，使用 `try (ContextScope scope = FrameworkContext.openScope(snapshot))` 受控执行；
  3. Web 出口处由 `FrameworkContextClearFilter`、Kafka 消费者出口处在 `finally` 强制兜底清理全部 6 大上下文，杜绝线程污染。

---

### ADR-005: 消除常量接口反模式，全面推行工具类单数命名规范
- **状态**: 已落地 (Accepted & Implemented)
- **背景**: 历史代码中散落了大量 `public interface *Constant`（常量接口反模式）以及 `*Utils` 复数工具类，存在常量污染和第三方废弃代码复制风险。
- **决策**:
  1. 全面根除常量接口，常量统一采用 **`public final class` + `private Constructor`** 模式封装（如 `HeaderConstant`、`CryptoConstant`）；
  2. 强制使用单数命名工具类（`JsonUtil`、`IdUtil`、`DateUtil`、`DigestUtil`、`StringUtil`、`BeanUtil`、`PropUtil`），所有历史复数类已彻底清理。

---

### ADR-006: MyBatis-Plus 拦截器严格拓扑编排与竞态根治
- **状态**: 已落地 (Accepted & Implemented)
- **背景**: 多租户拦截器与数据权限拦截器各自通过 `interceptors.add(0, ...)` 动态注入，执行先后次序受 Spring Bean 初始化顺序影响，存在数据权限排在租户拦截器之前的竞态隐患。
- **决策**:
  1. 在 `MybatisTenantAutoConfiguration` 中强制将 `TenantLineInnerInterceptor` 锚定在链表首位 `index = 0`；
  2. 在 `MybatisDataScopeAutoConfiguration` 中智能探测租户拦截器位置并锚定在 `tenantIndex + 1`，且在 `PaginationInnerInterceptor` 之前；
  3. 最终拦截器拓扑严格保证：`Tenant -> DataScope -> Pagination`，彻底消除初始化竞态。

---

### ADR-007: 根 POM 依赖卫生与外部依赖版本 100% 集中纳管
- **状态**: 已落地 (Accepted & Implemented)
- **背景**: 曾有少量子模块 POM 存在写死的 `<version>`（如 `mybatis-spring:3.0.4`、`jspecify:1.0.0`），违反依赖卫生原则。
- **决策**:
  1. 根 POM `<dependencies>` 仅保留 `lombok` 与 `spring-boot-configuration-processor`；
  2. 所有第三方依赖版本统一定义在根 POM 的 `<properties>` 与 `<dependencyManagement>`，18 个子模块严格遵循零硬编码版本，杜绝版本碎片化。
