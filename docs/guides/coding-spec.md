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
6. **零业务概念反向渗透 (Zero Business Penetration)**:
   - 底层框架严禁硬编码具体业务系统代号（如 `ncap`）或具体业务字段，业务扩展点必须通过标准 SPI 接口（如 `DeptAncestorProvider`）暴露。
7. **高频路径 GC 防护与预编译正则**:
   - 严禁在 SQL 拦截器或查询主链路中动态拼接编译正则表达式；所有正则必须提取为 `private static final Pattern` 静态预编译常量复用，消除 Minor GC 堆内存抖动。
8. **跨平台字符集锁死 (StandardCharsets.UTF_8)**:
   - 涉及字节数组与字符串互转（加解密、网络、Redis 监听）时，必须显式指定 `StandardCharsets.UTF_8`，严禁依赖 JVM 默认编码引发 Windows/Linux 乱码。
9. **绝对禁止代码行内全限定类名 (Anti-FQCN Rule - 绝对铁律)**:
   - 严禁在类成员变量、局部变量、方法参数、返回值、强制类型转换、静态方法调用或注解声明中使用带包名的全路径（如 `@org.springframework...`、`com.zhkj...`、`java.util.Base64` 等）；
   - **所有依赖项必须且只能在文件头部（package 声明下方）集中统一显式 `import`**，保持代码现代工业级整洁度。

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
| **`DigestUtil`** | MD5、SHA256、国密 SM3 消息摘要 (支持 String 与 byte[] 字节流) | ~~`DigestUtils`~~ |
| **`BeanUtil`** | 高性能字节码 Bean 属性拷贝与 Map 互转 | ~~`BeanUtils`~~ |
| **`PropUtil`** | 属性配置文件加载与读取 | ~~`PropsUtil`~~ |

### 2.2 工具类架构级防御与不可实例化约束
- **强制 `final` 修饰与私有构造器**: 全框架所有 `*Util` 类必须声明为 **`public final class`**，并显式提供私有构造器 `private XxxUtil() {}`，杜绝任何外部实例化或不安全继承。
- **常量池类标准封装**: `StringPool` 与 `CharPool` 统一声明为带有私有构造器的 `public final class`，严禁以 `implements StringPool` 反模式使用。

### 2.3 优先使用现代 Java 标准能力
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

---

## 🏛️ 5. 实体架构与泛型规范 (Entity Architecture & Generic Specification)

### 5.1 泛型实体主键规范
框架全面采用强类型泛型接口 `IEntity<ID extends Serializable>` 与 `IAuditEntity<ID extends Serializable>`，杜绝历史 `Object id` 弱类型设计与方法重载歧义。在实体继承时，严格遵循以下继承标准：

1. **默认 Long 雪花算法主键（绝大多数业务实体）**：
   - 直接继承 **`MybatisBaseEntity`**（内部已特化实现 `MybatisAuditEntity<Long>`，自动装配 `@TableId(type = IdType.ASSIGN_ID)`、审计字段与默认逻辑删除）。
   - 示例：
     ```java
     @Data
     @EqualsAndHashCode(callSuper = true)
     @TableName("biz_order")
     public class Order extends MybatisBaseEntity {
         private String orderNo;
         private BigDecimal amount;
     }
     ```
2. **自定义主键类型（如 String 租户/机构编码、UUID 等）**：
   - 继承 **`MybatisAuditEntity<ID>`**，并在泛型参数中明确指定主键类型（如 `String` 等）。
   - 子类自行声明 `@TableId` 物理主键字段及具体生成策略。
   - 示例：
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

### 5.2 维度标记接口正交解耦 (Orthogonal Dimension Markers)
框架将实体能力划分为正交维度，`ILogicDeleteEntity<T>`、`ITenantEntity`、`IVersionEntity<T>` 均直接继承 `Serializable`，彻底从 `IEntity` 依赖树中解耦：
- **设计原则**：维度标记接口与实体的物理主键类型无关，各维度职责完全正交。
- **自由组装**：业务实体可根据自身持久化特征自由组合（例如仅需逻辑删除的实现 `ILogicDeleteEntity<Integer>`；多租户实体实现 `ITenantEntity`；需要乐观锁并发控制的实现 `IVersionEntity<Integer>`），杜绝由于单一根继承链导致的虚假继承和冗余方法重载。

### 5.3 Jackson 3 架构纪律与序列化安全 (Rule 20)
在实体层与框架通用层**严禁引入 `com.fasterxml.jackson.*` 注解**（如 `@JsonIgnore`、`@JsonProperty`、`@JsonSetter`）：
- **架构根因**：OpenAPI / SpringDoc（基于 Jackson 2 内核）在扫描实体属性时，若基类与子类存在 `setId(Object)` 与 `setId(Long)` 双重定义，会抛出 `Conflicting setter definitions for property "id"` 异常。
- **Javac 桥接解法**：通过泛型继承 `MybatisAuditEntity<ID>`，Javac 编译器在编译期会自动生成带有 `ACC_BRIDGE` / `ACC_SYNTHETIC` 标记的桥接字节码。SpringDoc、Swagger 以及 Jackson 3 会自动忽略合成桥接方法，仅识别唯一的强类型 setter，既保证了实体层遵循纯粹的 Jackson 3 体系（Rule 20），又从底层天然规避了任何第三方文档/序列化工具的 Setter 冲突。
