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
- **实施现状（核对于工作树）**: `HttpClientRegistrar`、`HttpClientHeaderInterceptor` 等实现位于 `zhkj-framework-starter-http`；框架 pom 中无 OpenFeign 依赖。
- **实施现状补充（`[待确认]`）**: `HttpClientHeaderInterceptor` 的透传来源是否覆盖 Servlet / Spring Batch / `@Async` / Kafka 消费端，未逐一核实；框架 pom 未引入 Spring Batch；Kafka 上下文由 `starter-kafka` 的 `KafkaContextProducerInterceptor` 与 `KafkaTraceRecordInterceptor` 承担。
- **背景**: OpenFeign 维护停滞，且对响应式与 Spring 6 现代通信体系支持薄弱。
- **决策**: 
  1. 废弃 Feign 全套依赖，研发自研声明式客户端 `zhkj-framework-starter-http`；
  2. 基于 Spring 6+ `@HttpExchange`、`@GetExchange`、`@PostExchange` 现代注解构建 RPC；
  3. 内置 `HttpClientHeaderInterceptor`，在 Servlet 请求、Spring Batch、`@Async` 异步线程及 Kafka 消费端透明透传 `X-Tenant-Id`、`X-User-Id` 等全链路身份上下文。

---

### ADR-002: Jackson 3 现代化全面升级与 Jackson 2 兼容层治理
- **状态**: 已落地 (Accepted & Implemented)
- **实施现状（核对于工作树）**: `JacksonConfiguration` 使用 `DateTimeFeature.WRITE_DATES_AS_TIMESTAMPS`；`RedisTemplateConfiguration` 使用 `RedisSerializer.json()`。
- **实施现状补充（`[待确认]`）**: 决策第 2 项所述“Jackson 2 兼容层”在源码中未找到独立实现（框架源码未引用 `com.fasterxml.jackson`，见 `coding-spec.md` 第 5.3 节）。
- **背景**: 业界 JSON 序列化演进至 Jackson 3 (`tools.jackson.*`)，废弃了原 2.x 的很多低效同步阻塞逻辑。
- **决策**:
  1. 框架底层核心统一基于 `tools.jackson.core:jackson-databind` 3.x 构建；
  2. 废弃已淘汰的旧特性方法（如使用 `DateTimeFeature.WRITE_DATES_AS_TIMESTAMPS` 替代废弃的 `SerializationFeature`）；
  3. `RedisSerializer.json()` 统一使用现代化工厂，全面提升 JSON 解析吞吐量。

---

### ADR-003: 淘汰 Jasypt/RSA，构建全生命周期国密 SM4/AES 与 HMAC-SM3 盲索引安全体系
- **状态**: 已落地 (Accepted & Implemented)
- **实施现状（核对于工作树）**: 盲索引与加解密实现位于 `zhkj-framework-starter-crypto`；框架 pom 中无 Jasypt 依赖。展示态当前 `DesensitizeType` 为 7 种规则（见 `security.md` 第 2.2 节），非本文原述的 10 种。存量差异：`BlindIndexUtil` 的 SPI 提供者失败时回退默认算法（`zhkj-framework-core/.../util/crypto/BlindIndexUtil.java:119,126,209`），与“杜绝哈希漂移”的决策不完全一致，待确认。
- **背景**: 旧版配置解密采用第三方 Jasypt，算法陈旧且在 Spring Boot 4 启动阶段频发松散绑定 ClassCast 异常；持久层隐私数据明文存储或加密后无法检索。
- **决策**:
  1. 彻底移除 Jasypt 与 RSA，构建专属 `zhkj-framework-starter-crypto`；
  2. **配置态**: 基于 Boot 4 原生 `EnvironmentPostProcessor` 统一解密 `ENC(...)`；
  3. **展示态**: 统一 Jackson `@Desensitize` 动态打码（决策时拟支持 10 种规则；已废弃：当前 `DesensitizeType` 为 7 种，见上方实施现状）；
  4. **存储态**: 采用国密 SM4/AES 透明加密（`EncryptTypeHandler`）+ HMAC-SM3 加盐哈希盲索引列（`*_hash`），由 `ZhkjMetaObjectHandler` 自动计算注入，支持数据库密文精准等值检索（EQ/IN）。

---

### ADR-004: 统一上下文生命周期门面 (`FrameworkContext`) 与 ThreadLocal 安全治理
- **状态**: 已落地 (Accepted & Implemented)
- **实施现状（核对于工作树）**: `FrameworkContext.clearAll()`（`FrameworkContext.java` L240-L270）清理快照、SPI 提供者、`DesensitizeContext`、`SecurityContextHolder`、`TenantContext`、`SsoContextHolder`、`DataScopeContext` 及自定义清理器；后四者通过反射按类名调用 `clear`（L257-L260）。Kafka 侧 `KafkaTraceRecordInterceptor` 在 `afterRecord` / `failure` 中经 `clearContext()` 的 `finally` 调用 `clearAll()`（`KafkaTraceRecordInterceptor.java:117-131`）。
- **背景**: 原有多租户、安全身份、SSO 凭证分散在各自的 ThreadLocal 中，在线程池并发复用场景下容易出现串号或内存泄漏；使用 `InheritableThreadLocal` 容易在子线程复用时隐式继承污染。
- **决策**:
  1. 在 `core` 模块建立统一门面 `FrameworkContext`，采用显式 `ThreadLocal` 纳管快照；
  2. 跨线程统一使用 `FrameworkContext.capture()` 捕获快照，使用 `try (ContextScope scope = FrameworkContext.openScope(snapshot))` 受控执行；
  3. Web 出口处由 `FrameworkContextClearFilter`、Kafka 消费者出口处在 `finally` 强制兜底清理全部 6 大上下文，杜绝线程污染。

---

### ADR-005: 消除常量接口反模式，全面推行工具类单数命名规范
- **状态**: 已落地 (Accepted & Implemented)
- **实施现状（核对于工作树）**: `StringPool`、`CharPool` 为 `final class`；`DateUtil.DateToString` 保留为 `@Deprecated(forRemoval = true)` 转发方法（`DateUtil.java:604-607`）。存量例外：`SpringUtil` 未声明私有构造器（`SpringUtil.java:15`）。
- **背景**: 历史代码中散落了大量 `public interface *Constant`（常量接口反模式）以及 `*Utils` 复数工具类，存在常量污染和第三方废弃代码复制风险。
- **决策**:
  1. 全面根除常量接口，常量统一采用 **`public final class` + `private Constructor`** 模式封装（如 `HeaderConstant`、`CryptoConstant`）；
  2. 强制使用单数命名工具类（`JsonUtil`、`IdUtil`、`DateUtil`、`DigestUtil`、`StringUtil`、`BeanUtil`、`PropUtil`），所有历史复数类已彻底清理。

---

### ADR-006: MyBatis-Plus 拦截器严格拓扑编排与竞态根治
- **状态**: 已落地 (Accepted & Implemented)
- **实施现状（核对于工作树）**: 拦截器挂载顺序见 `MybatisPlusConfiguration.java:175-207`（Tenant → DataScope → Custom → Pagination）；数据权限拦截器的插入位置在 `MybatisDataScopeAutoConfiguration.java:86-99`（`tenantIndex + 1`）。决策正文中“在 `MybatisTenantAutoConfiguration` 中锚定 index = 0”的描述与当前实现位置不一致，以此注记为准。
- **背景**: 多租户拦截器与数据权限拦截器各自通过 `interceptors.add(0, ...)` 动态注入，执行先后次序受 Spring Bean 初始化顺序影响，存在数据权限排在租户拦截器之前的竞态隐患。
- **决策**:
  1. 已废弃：“在 `MybatisTenantAutoConfiguration` 中强制将 `TenantLineInnerInterceptor` 锚定在链表首位 `index = 0`”。当前实现为 `MybatisPlusConfiguration.mybatisPlusInterceptor` 中按 `addInnerInterceptor` 顺序首位挂载（见上方实施现状）；
  2. 在 `MybatisDataScopeAutoConfiguration` 中智能探测租户拦截器位置并锚定在 `tenantIndex + 1`，且在 `PaginationInnerInterceptor` 之前；
  3. 最终拦截器拓扑严格保证：`Tenant -> DataScope -> Pagination`，彻底消除初始化竞态。

---

### ADR-007: 根 POM 依赖卫生与外部依赖版本 100% 集中纳管
- **状态**: 已落地 (Accepted & Implemented)
- **实施现状（核对于工作树）**: 根 POM `<dependencies>` 仅含 `lombok` 与 `spring-boot-configuration-processor`；子模块 POM 未发现字面量版本号。经检索框架与 NCAP 全部 POM，引用未定义版本属性的 Batik（`batik-all` / `batik-svgpp`）、XML Graphics Commons、XML Security、XXL-Job、P6Spy 均无消费声明，对应六项依赖管理已删除，不补猜版本。框架模块数为 17（正文中的 18 为历史表述）。
- **背景**: 曾有少量子模块 POM 存在写死的 `<version>`（如 `mybatis-spring:3.0.4`、`jspecify:1.0.0`），违反依赖卫生原则。
- **决策**:
  1. 根 POM `<dependencies>` 仅保留 `lombok` 与 `spring-boot-configuration-processor`；
  2. 所有第三方依赖版本统一定义在根 POM 的 `<properties>` 与 `<dependencyManagement>`，各框架子模块严格遵循零硬编码依赖版本，杜绝版本碎片化（已废弃：“18 个子模块”为历史表述，当前根 POM `<modules>` 为 17 个）。

---

### ADR-008: 全工程系统性架构治理、零业务渗透、GC 性能与上下文内存安全加固 (2026.4.1.0-SNAPSHOT)
- **状态**: 已落地 (Accepted & Implemented)
- **实施现状（逐条核对于工作树）**: (1) 默认种子已移除，框架源码无 `zhkj-ncap` 字样 ✓；(2) `SM2Util.java:124` 使用 `UTF_8`，框架源码无无参 `getBytes()` ✓；(3) `SqlKeyword.java:21` 为静态 `Pattern` ✓；(4) `KafkaTraceRecordInterceptor.java:117` 已实现 `failure` 回调 ✓；(5) `MybatisDataScopeInterceptor.java:49-53` 改为静态 `ThreadLocal` 并注册清理器 ✓，但 `AuditUserProvider` 仍经 `SpringUtil.getBean` 获取（`:217`），未改为 `ObjectProvider`，部分完成；(6) 工具类 `final` 与私有构造器，除 `SpringUtil`（`:15`）外 ✓；(7) `StringPool`/`CharPool` 为 final class ✓。
- **背景**: 在框架 18 模块全景排查中识别出 5 项高危架构硬伤与多项技术债务：加解密种子硬编码业务代号 `zhkj-ncap`、SM2/Redis 字符集依赖平台编码引发乱码、SqlKeyword 每次查询动态编译正则造成 Minor GC 抖动、Kafka 单条消费异常未清理上下文、MybatisDataScopeInterceptor ThreadLocal 内存挂起，以及全工程工具类缺少 private 构造器防线。
- **决策**:
  1. **零业务反向渗透**: 拔除 `starter-crypto` 中硬编码的 `zhkj-ncap` 种子，升级为框架级中立密钥；
  2. **字符集跨平台锁死**: `SM2Util` 与 Redis 监听器强制显式指定 `StandardCharsets.UTF_8`，杜绝跨环境验签与解密失败；
  3. **高频路径 GC 防护**: `SqlKeyword` 正则表达式重构为静态预编译 `Pattern` 复用，消除批量查询下的堆抖动；
  4. **上下文全生命周期闭环**: `KafkaTraceRecordInterceptor` 补齐 `failure` 异常清理回调；`MybatisDataScopeInterceptor` 消除 ThreadLocal 悬挂并引入 `ObjectProvider` 依赖注入优化；
  5. **工程防御加固**: 全框架 30 余个工具类统一补全 `public final class` 与私有防实例化构造器；`StringPool`/`CharPool` 彻底由 interface 升级为标准 final class；清理历史冗余类并平滑过渡。

---

### ADR-009: 内部微服务 RPC 脱敏旁路放行机制与 Anti-FQCN 纯净化治理 (2026.4.1.0-SNAPSHOT)
- **状态**: 已落地 (Accepted & Implemented)
- **实施现状（核对于工作树）**: `InternalRpcDesensitizeFilter` 顺序为 `HIGHEST_PRECEDENCE + 20`（`WebAutoConfiguration.java:69`）；`DigestUtil.sha256(byte[])` 存在（`DigestUtil.java:73`）；Anti-FQCN 存量 2 处，见 `coding-spec.md` 第 6 节。
- **背景**:
  1. **内部调用数据失真痛点**: 原框架展示态 `@Desensitize` 在 Controller 序列化输出时统一对手机号、身份证、银行卡等脱敏打码。但在微服务协同（声明式 RPC 客户端 `@HttpExchange`）场景下，下游系统（如催收作业中心调度外呼拨号、批处理计息跑批、短信服务发送通知）获取到打码的 `138****8000` 会直接引发业务功能熔断或数据污染；
  2. **行内全限定类名反模式 (FQCN)**: 框架及下游业务中偶发出现行内书写长包名全类名（如 `@org.springframework...`、`java.util.Base64` 等），严重违背代码可读性与团队整洁架构规范；
  3. **摘要算法字节流支持缺失**: `DigestUtil` 历史仅支持字符串入参的 `sha256(String)`，业务端在处理字节数组（二进制公私钥、证书、文件流）时被迫自行调用底层 Java 原生 API。
- **决策**:
  1. **内部微服务 RPC 脱敏旁路机制 (`InternalRpcDesensitizeFilter`)**:
     - 在 `zhkj-framework-starter-web` 中实现并装配高优先级 `InternalRpcDesensitizeFilter`（Ordered: `HIGHEST_PRECEDENCE + 20`）；
     - 仅当服务端在调用方认证与目标内部端点授权后，将当前 ServletRequest 的 `RpcRequestAttribute.DESENSITIZE_AUTHORIZED` 设置为 `Boolean.TRUE` 时，才触发 `DesensitizeContext.disable()`；`X-Internal-RPC: true` 仅为传输标记，普通 RPC 与公共展示端点默认脱敏；
     - 未授权请求显式启用脱敏；脱敏过滤器在 `finally` 恢复进入过滤器前的开关状态以支持嵌套调用，而非无条件 `enable()`。授权方在 `finally` 删除请求属性，外层 `FrameworkContextClearFilter` 清理请求上下文；请求属性不得从 Header 或 ThreadLocal 推断。
     - **2026-11-10 安全行为变更**：宿主认证过滤器必须先于脱敏过滤器的 `HIGHEST_PRECEDENCE + 20` 执行，复用宿主认证机制，仅在已保护内部路径完成调用方认证与目标端点授权后赋权；正确机器凭据不使公共展示 API 获得明文权限，没有明文业务需求的内部端点不应赋权。
     - **迁移边界**：旧标记头不再授予明文或委托权限；宿主须接入认证、端点授权及请求属性作用域清理。委托头另需 `RpcRequestAttribute.TENANT_DELEGATION_AUTHORIZED == Boolean.TRUE`，机器认证不是跨租户授权；可信 JWT 声明上下文保留。网关在匿名、disabled 与认证路径均剥离三个内部头（`X-Internal-RPC`、`X-Delegated-Tenant-Id`、`X-Grant-Ticket`）。
  2. **Anti-FQCN 纯净化铁律**:
     - 确立 Anti-FQCN 绝对红线：严禁在类成员、局部变量、方法入参、返回值、泛型、强转或注解声明中使用带包路径的内联全类名；所有依赖必须 100% 提取并在文件头部（package 声明下方）集中显式 `import` 纯类名；
  3. **DigestUtil 字节流与多源摘要扩展**:
     - 在 `zhkj-framework-core` 的 `DigestUtil` 中扩展 `sha256(byte[] data)`（`DigestUtil.java:73`）与 `md5Hex(byte[])`（`DigestUtil.java:46`）字节数组重载（已废弃：源码中不存在 `sha256Hex`）。

