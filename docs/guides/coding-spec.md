# ZHKJ Framework 开发者编码规范与技术红线

> **文档定位**: 统一全团队编码习惯、杜绝历史异味反模式、保障代码质量与现代 Java 工业级标准对齐。  
> **设计与创作者**: Jason.Yang（274107098@qq.com）  

---

## 🚫 1. 架构级“绝对红线”

1. **严禁依赖下沉与反向引用**:
   - `core` 与 `data-api` 严禁引入任何 Spring Web、Servlet、MyBatis 或具体中间件依赖；
   - 工具类严禁反向依赖高层业务类。
2. **彻底根除常量接口反模式 (Eliminate Constant Interface Antipattern)**:
   - 严禁在任何模块中声明 `public interface *Constant`；
   - 所有常量类强制采用 **`public final class` + `private Constructor`** 现代模式封装，杜绝类型污染与不安全继承。
3. **严禁散落硬编码 HTTP 请求头与加密字符串**:
   - 跨服务请求头统一引用 `HeaderConstant`（如 `HeaderConstant.X_TENANT_ID`、`HeaderConstant.AUTHORIZATION`）；
   - 链路追踪统一引用 `TraceConstant.TRACE_ID`；
   - 算法与密文前缀统一引用 `CryptoConstant`。
4. **持久层国密加密与盲索引防漏**:
   - 严禁在业务 Service 中手写计算盲索引哈希（如手工调用 `generateIndex` 赋值）；
   - 必须通过 `@EncryptField` / `@BlindIndex` 配合 `ZhkjMetaObjectHandler` 自动注入，确保入库 100% 防漏。
5. **Spring Boot 4 内部配置类注解规范**:
   - 仅在 `AutoConfiguration.imports` 中作为顶级入口的类标注 `@AutoConfiguration`；
   - 内部由主配置类通过 `@Import` 引入的子模块配置类，必须使用 `@Configuration(proxyBeanMethods = false)`，杜绝注解语义二义性并降低 CGLIB 代理开销。

---

## 🛠️ 2. 工具类使用与命名规约

### 2.1 强制单数命名规范 (Strict Singular Naming)
框架已全面物理清除历史复数工具类，所有工具类必须使用正统单数命名：

| 规范单数工具类 | 核心功能说明 | 严禁使用的旧复数类 |
| :--- | :--- | :--- |
| **`JsonUtil`** | Jackson 统一 JSON 序列化与反序列化 | ~~`JsonUtils`~~ |
| **`IdUtil`** | UUID、雪花算法 ID、序号生成器 | ~~`IdUtils`~~ |
| **`DateUtil`** | Java 8 日期时间、经典 Date 格式化 | ~~`DateTimeUtil`~~, ~~`ConcurrentDateFormat`~~ |
| **`StringUtil`** | 字符串判空、占位符替换、驼峰下划线互转 | ~~`StringUtils`~~ |
| **`DigestUtil`** | MD5、SHA256、国密 SM3 消息摘要 | ~~`DigestUtils`~~ |
| **`BeanUtil`** | 高性能字节码 Bean 属性拷贝与 Map 互转 | ~~`BeanUtils`~~ |
| **`PropUtil`** | 属性配置文件加载与读取 | ~~`PropsUtil`~~ |

### 2.2 优先使用现代 Java 标准能力
- **Base64**: 直接使用 JDK 8+ 原生 `java.util.Base64`，严禁新建浅包装类；
- **时间格式化**: 使用 `DateUtil` 内置的常量与 `DateTimeFormatter`，严禁在并发环境使用非线程安全的 `SimpleDateFormat`。

---

## ⚡ 3. 现代化 API 调用清单

在日常编码中，严格遵守现代框架 API 规约，清零废弃警告：

1. **JJWT 0.12+**:
   - 严禁使用废弃的 `setSigningKey` 与 `SignatureAlgorithm`；
   - 统一采用流式语法：
     ```java
     Jwts.builder().header().type(...).and().issuer(...).claims(...).signWith(secretKey).compact();
     ```
2. **Spring Data Redis**:
   - 严禁使用带 `(key, value, timeout, TimeUnit)` 的废弃重载；
   - 统一传入 `Duration`：`redisTemplate.opsForValue().set(key, val, Duration.ofMinutes(10));`。
3. **Spring JDBC (`JdbcTemplate`)**:
   - 严禁使用带 `(sql, Object[] args, RowMapper)` 的废弃重载；
   - 统一使用原生变长参数：`jdbcTemplate.query(sql, rowMapper, arg1, arg2);`。
4. **Spring Batch 6+**:
   - 显式声明并注入 `JobRegistry`，Chunk 必须使用流式调用 `StepBuilder.chunk(n).transactionManager(tm)`。

---

## 🧵 4. 统一上下文传递规范 (`FrameworkContext`)

严禁依赖隐式 `InheritableThreadLocal`（在线程池复用时极易产生串号与脏数据污染）。在异步线程（`@Async`、线程池）、批处理（Spring Batch）或消息队列消费者中，必须显式传递并受控释放上下文：

```java
// 1. 在主线程捕获完整上下文快照
FrameworkContext.Snapshot snapshot = FrameworkContext.capture();

// 2. 在子线程或异步任务中执行
executor.execute(() -> {
    try (ContextScope scope = FrameworkContext.openScope(snapshot)) {
        // 在作用域内，TenantContext、SecurityContextHolder 等全部自动还原
        orderService.processAsyncOrder();
    } // 退出作用域时，所有上下文自动彻底释放，防止线程池复用污染
});
```
