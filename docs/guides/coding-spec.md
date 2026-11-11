# ZHKJ Framework 开发者编码规范与技术红线

> **文档定位**: 统一团队编码习惯、约束历史反模式、保障代码质量与现代 Java 工业级标准对齐。  
> **设计与创作者**: Jason.Yang（274107098@qq.com）  
> **存量说明**: 各条规则的代码现状核对结果见第 6 节；未达标项标注为“存量待治理”，不代表允许新增违规代码。

---

## 🚫 1. 架构级“绝对红线”

1. **依赖方向与反向引用**:
   - `core` 不依赖任何框架模块，`data-api` 仅依赖 `core`；
   - `core` 与 `data-api` 严禁引入 Spring Web、Servlet、MyBatis 或具体中间件依赖；
   - 工具类严禁反向依赖高层业务类。
2. **根除常量接口反模式**:
   - 严禁声明 `public interface *Constant`；
   - 常量类采用 **`public final class` + `private Constructor`** 封装（如 `HeaderConstant`、`CryptoConstant`、`TraceConstant`）。
3. **请求头与加密字符串统一常量化**:
   - 跨服务请求头统一引用 `HeaderConstant`（如 `HeaderConstant.X_TENANT_ID`、`HeaderConstant.AUTHORIZATION`）；
   - 链路追踪统一引用 `TraceConstant.TRACE_ID`；
   - 算法名统一引用 `CryptoConstant`（如 `ALGO_SM4`、`ALGO_HMAC_SM3`）。
4. **持久层国密加密与盲索引**:
   - 不在业务 Service 中手工计算盲索引哈希后赋值；
   - 通过 `@EncryptField(blindIndexField = ...)` 与 `@BlindIndex(source = ...)` 声明，由 `ZhkjMetaObjectHandler` 注入。
5. **自动装配配置类注解**:
   - 作为自动配置入口的类（在 `AutoConfiguration.imports` 中注册）标注 `@AutoConfiguration`；
   - 由主配置类通过 `@Import` 或嵌套方式引入的子配置类使用 `@Configuration(proxyBeanMethods = false)`。
6. **零业务概念渗透**: 底层框架不硬编码业务系统代号或业务字段，业务扩展点通过 SPI 接口（如 `DeptAncestorProvider`）暴露。
7. **高频路径正则预编译**: 查询主链路与拦截器中的正则统一为 `private static final Pattern`，禁止动态拼接编译。
8. **字符集显式指定**: 字节与字符串互转（加解密、网络、Redis 监听）必须显式指定 `StandardCharsets.UTF_8`，不依赖平台默认编码。
9. **Anti-FQCN**: 正文中不使用行内全限定类名，依赖统一在文件头 `import`。

---

## 🛠️ 2. 工具类使用与命名规约

### 2.1 单数命名规范
框架工具类使用单数命名，以下为现行类（均位于 `zhkj-framework-core`）：

| 规范单数工具类 | 核心功能说明 | 不应新增的复数写法 |
| :--- | :--- | :--- |
| **`JsonUtil`** | Jackson 统一 JSON 序列化与反序列化（异常统一为 `JsonParseException`） | `JsonUtils` |
| **`IdUtil`** | UUID 生成（`randomUUID`、`simpleUUID`、`fastUUID`、`fastSimpleUUID`） | `IdUtils` |
| **`DateUtil`** | 日期时间格式化与解析（`dateToString` 为推荐方法名，`DateToString` 已 `@Deprecated(forRemoval = true)`） | `DateTimeUtil`, `ConcurrentDateFormat` |
| **`StringUtil`** | 字符串判空、清洗、标识符安全化等 | `StringUtils` |
| **`DigestUtil`** | MD5（`md5Hex`）、SHA256（`sha256`）消息摘要，支持 `String` 与 `byte[]` 入参 | `DigestUtils` |
| **`BeanUtil`** | Bean 属性拷贝与 Map 互转 | `BeanUtils` |
| **`PropUtil`** | `Properties` 属性读写（`getProperty` / `setProperty`，不含文件加载） | `PropsUtil` |

国密 SM3 摘要不在 `DigestUtil` 中，位于 `zhkj-framework-starter-crypto` 的 `SM3Util`。

> 源码中不存在 `*Utils` 复数工具类、`DateTimeUtil` 与 `ConcurrentDateFormat`（已删除）。`DateUtil.DateToString` 为 `@Deprecated(since = "2026.4.1.0", forRemoval = true)` 兼容转发方法（`DateUtil.java:604-607`），新代码使用 `dateToString`。

### 2.2 工具类不可实例化约束
- `*Util` 类应声明为 **`public final class`** 并提供 **`private XxxUtil() {}`**；
- `StringPool` 与 `CharPool` 为带私有构造器的 `public final class`，以静态字段方式引用（如 `StringPool.DOT`）。

### 2.3 优先使用现代 Java 标准能力
- **Base64**: 直接使用 JDK 8+ `java.util.Base64`，不新建包装类；
- **时间格式化**: 使用 `DateUtil` 或 `DateTimeFormatter`，不在并发环境中共享 `SimpleDateFormat`。

---

## ⚡ 3. 现代化 API 调用清单

1. **JJWT 0.12+**（`SecureUtil` 已采用 `Jwts.parser().verifyWith(...)`）:
   - 不使用 `setSigningKey` 与 `SignatureAlgorithm` 等废弃 API；
   - 构造示例：
     ```java
     Jwts.builder().header().type(...).and().issuer(...).claims(...).signWith(secretKey).compact();
     ```
2. **Spring Data Redis**:
   - 不使用带 `(key, value, timeout, TimeUnit)` 的重载；
   - 使用 `Duration`：`redisTemplate.opsForValue().set(key, val, Duration.ofMinutes(10));`。
3. **Spring JDBC (`JdbcTemplate`)**:
   - 不使用带 `(sql, Object[] args, RowMapper)` 的重载；
   - 使用变长参数：`jdbcTemplate.query(sql, rowMapper, arg1, arg2);`。
4. **Spring Batch 6+**（业务工程场景，框架 pom 不引入 Spring Batch）:
   - 显式声明并注入 `JobRegistry`；Chunk 使用 `StepBuilder.chunk(n).transactionManager(tm)`。

---

## 🧵 4. 统一上下文传递规范 (`FrameworkContext`)

不依赖隐式 `InheritableThreadLocal`。异步线程（`@Async`、线程池）、批处理或消息消费者中，显式传递并受控释放上下文：

```java
import io.github.ybb88.framework.context.ContextScope;
import io.github.ybb88.framework.context.ContextSnapshot;
import io.github.ybb88.framework.context.FrameworkContext;

// 1. 在请求线程中捕获完整上下文快照
ContextSnapshot snapshot = FrameworkContext.capture();

// 2. 在子线程中执行
executor.execute(() -> {
    try (ContextScope scope = FrameworkContext.openScope(snapshot)) {
        // 作用域内 TenantContext、SecurityContextHolder 等已还原
        orderService.processAsyncOrder();
    } // 离开作用域时自动清理，防止线程池复用污染
});
```

---

## 🏛️ 5. 实体架构与泛型规范

### 5.1 主键与实体基类
框架使用泛型接口 `IEntity<ID extends Serializable>` 与 `IAuditEntity<ID extends Serializable>`（`zhkj-framework-data-api`）。

1. **默认 Long 主键（雪花 ID）**: 继承 **`MybatisBaseEntity`**（其继承 `MybatisAuditEntity<Long>`，并声明 `@TableId(type = IdType.ASSIGN_ID)`）。
   ```java
   @Data
   @EqualsAndHashCode(callSuper = true)
   @TableName("biz_order")
   public class Order extends MybatisBaseEntity {
       private String orderNo;
       private BigDecimal amount;
   }
   ```
2. **自定义主键类型（如 String 编码）**: 继承 **`MybatisAuditEntity<ID>`**，由子类声明 `@TableId` 字段与生成策略。
   ```java
   @Data
   @EqualsAndHashCode(callSuper = true)
   @TableName("sys_tenant")
   public class SysTenant extends MybatisAuditEntity<String> {
       @TableId(type = IdType.INPUT)
       private String id;

       private String tenantName;
   }
   ```
3. 逻辑删除与租户能力由 `MybatisLogicEntity`、`MybatisTenantLogicEntity`、`MybatisTenantEntity` 等基类提供（`starter-mybatis/.../base/`）。

### 5.2 维度标记接口
`ILogicDeleteEntity<T>`、`ITenantEntity`、`IVersionEntity<T>` 均直接继承 `Serializable`，与实体主键类型无关，可按需组合实现。

### 5.3 Jackson 与实体注解
实体与框架通用层不引入 `com.fasterxml.jackson.*` 注解（当前框架源码中未发现该类引用）。框架使用 Jackson 3（`tools.jackson.*`）。

---

## 6. 代码现状核对（存量待治理）

以下为对当前工作树的抽样核对结果，用于说明规范与代码的差距；新增代码应遵循上文规约。

| 规则 | 现状 | 位置 |
| :--- | :--- | :--- |
| 工具类私有构造器（第 2.2 节） | `SpringUtil` 未声明私有构造器 | `zhkj-framework-core/src/main/java/io/github/ybb88/framework/util/SpringUtil.java:15` |
| Anti-FQCN（第 1.9 条） | 行内全限定类名 2 处（已复核） | `zhkj-framework-core/.../util/crypto/BlindIndexUtil.java:75-77`（`java.security.*`）；`zhkj-framework-starter-mybatis/.../config/MybatisPlusConfiguration.java:101`（`com.baomidou.*`） |
| 自动配置注解（第 1.5 条） | 切面类 `RequestLogAspect` 标注了 `@AutoConfiguration`，待确认是否应改为其他注解 | `zhkj-framework-starter-web/.../logger/RequestLogAspect.java:54` |
| 框架层不硬编码业务表名（第 1.6 条） | 数据权限默认表名 `sys_org`（:39）、`zhkj_scope_data`（:64）、`zhkj_role_scope`（:69） | `zhkj-framework-starter-datascope/.../props/DataScopeProperties.java:39,64,69` |
| 盲索引不回退（`docs/architecture/security.md` 第 2.3.2 节） | SPI 提供者失败时回退默认算法 | `zhkj-framework-core/.../util/crypto/BlindIndexUtil.java:119,126,209` |
| 日期格式化线程安全（第 2.3 节） | Jackson 配置中构造 `SimpleDateFormat` 实例（需确认 Jackson 是否克隆） | `zhkj-framework-starter-web/.../config/JacksonConfiguration.java:52` |

已符合规约的项（抽样）: `getBytes()` 无参调用在 `zhkj-framework-*/src/main` 中未发现；`StringPool`/`CharPool` 为 `final class`；`StringUtil`、`JsonUtil` 等 `*Util` 类均为 `final` 且（除 `SpringUtil` 外）具备私有构造器；Jackson 注解未在源码中引用；框架源码未发现 `zhkj-ncap` 字样。
