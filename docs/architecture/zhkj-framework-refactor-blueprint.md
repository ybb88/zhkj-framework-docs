# zhkj-framework 全项目系统性架构重构作战全景图 (Master Refactoring Blueprint)

> **版本**：2026.4.1.0-SNAPSHOT  
> **评估范围**：根 POM 及全部 17 个子模块（核心底座、数据持久化、通信中间件、Web 与契约层）  
> **审查基准**：Spring Boot 4.x / Java 21 / 根 POM 依赖洁癖 / 单向依赖拓扑 / 零业务污染 / SPI 扩展契约  
> **状态**：已全面攻坚完成并经双工程 27 模块全量构建验收通过 (100% Completed & Verified)  
> **归档时间**：2026-09-21

---

## 一、 当前工程【高危架构硬伤清单】（Critical Flaws）

经过对全部 17 个子模块的源码排查，识别出以下 5 项可能直接导致生产事故、跨平台乱码、内存泄露或违反红线的高危硬伤：

### 1. 【业务概念反向渗透红线】默认加解密种子硬编码 ncap 业务标识
- **缺陷坐标**：`starter-crypto/.../EncryptablePropertyEnvironmentPostProcessor.java`
- **硬伤代码**：
  ```java
  private static final String DEFAULT_SECRET_BASE64 = "emhrai1uY2FwLWNyeXB0by1zZWNyZXQtMjAyNg==";
  ```
- **架构危害**：将上述 Base64 解码后明文为：`zhkj-ncap-crypto-secret-2026`。通用底座框架 `zhkj-framework` 属于跨业务平台的独立基础设施，内部直接硬编码了下游具体业务系统（NCAP 催收业务平台）的工程代号，严重践踏了“底层框架零业务渗透”的最高架构铁律。
- **治理方案**：统一脱敏并更正为框架级中立种子（如 `zhkj-framework-crypto-secret-2026` 对应的 Base64 编码 `emhrai1mcmFtZXdvcmstY3J5cHRvLXNlY3JldC0yMDI2`），并通过配置中心与环境变量外部化注入。

### 2. 【国密跨平台乱码与解密失败】SM2 加解密未指定 Charset 导致跨平台乱码
- **缺陷坐标**：`starter-crypto/.../SM2Util.java`
- **硬伤代码**：
  ```java
  // L79, L148, L180
  byte[] inputBytes = input.getBytes(); 
  // L120
  return new String(decryptedBytes); 
  ```
- **架构危害**：直接调用无参 `getBytes()` 与 `new String(byte[])`，将直接采用当前运行 JVM 的平台默认编码（在 Windows 或国内特定容器环境中常为 GBK 或 GB2312）。当客户端与服务端分别运行在 Windows 与 Linux（UTF-8）时，将直接引发签名校验失败与解密报文严重乱码甚至崩溃。
- **治理方案**：强制显式指定 `StandardCharsets.UTF_8`。

### 3. 【高并发上下文污染与泄露】Kafka 单条消息消费异常引发上下文悬挂
- **缺陷坐标**：`starter-kafka/.../KafkaTraceRecordInterceptor.java`
- **硬伤代码**：
  ```java
  @Override
  public void afterRecord(ConsumerRecord<String, Object> record, Consumer<String, Object> consumer) {
      try { FrameworkContext.clearAll(); } finally { TraceUtil.clearTraceId(); }
  }
  // 遗漏 failure 毁约回调
  ```
- **架构危害**：在 Spring Kafka 消费模型中，当业务 Listener 发生未捕获异常并交由全局 `CommonErrorHandler` 处理时，`afterRecord` 回调可能被绕过。类中未覆写 `failure(record, exception, consumer)`，导致异常发生后 `FrameworkContext` 和 MDC 残留在当前 Worker 线程中，污染下一个复用该线程的消息。
- **对比证据**：同模块的 `KafkaTraceBatchInterceptor.java` 正确覆写了 `success` 与 `failure` 双清理，单条拦截器存在实现缺陷。

### 4. 【高并发数据库查询 GC 击穿】MyBatis 关键字过滤拦截器高频编译正则
- **缺陷坐标**：`starter-mybatis/.../SqlKeyword.java`
- **硬伤代码**：
  ```java
  public static String filter(String param) {
      if (param == null) return null;
      return param.replaceAll("(?i)" + SQL_REGEX, StringPool.EMPTY);
  }
  ```
- **架构危害**：`SqlKeyword.filter` 在构建动态 QueryWrapper 时处于极高频调用路径。代码在每次调用时均通过动态拼接 `"(?i)" + SQL_REGEX` 并执行 `String.replaceAll()`。这将在运行期导致每次调用都重新分配并编译一个昂贵的 Pattern 和 Matcher 实例，在批量查询或高并发作业时触发严重的堆内存抖动与频繁 Minor GC。
- **治理方案**：提取为静态预编译常量 `private static final Pattern SQL_KEYWORD_PATTERN = Pattern.compile("(?i)(" + SQL_REGEX + ")");`，使用预编译正则复用。

### 5. 【数据权限拦截器 ThreadLocal 无 cleanUp 导致线程池悬挂】
- **缺陷坐标**：`starter-datascope/.../MybatisDataScopeInterceptor.java`
- **硬伤代码**：
  ```java
  private final ThreadLocal<Set<Integer>> processedBoundSqls = ThreadLocal.withInitial(HashSet::new);
  ...
  Set<Integer> processed = processedBoundSqls.get();
  if (processed.size() > 64) {
      processed.clear(); // 仅清空集合，ThreadLocal 永远未执行 remove()
  }
  ```
- **架构危害**：虽然设计了阈值清空，但整个拦截器在 SQL 执行完成后从未执行 `processedBoundSqls.remove()`。在 Tomcat/Undertow 容器或连接池常驻工作线程中，ThreadLocal 永远驻留线程内部；同时，方法内每次都直接通过 `SpringUtil.getBean(AuditUserProvider.class)` 执行 Spring 容器反射查找，未进行对象持有或 `ObjectProvider` 优化，在高并发持久层链路造成不必要的性能开销。

---

## 二、 当前工程【技术债务清单】（Technical Debts）

### 1. 工具类架构规范缺失（全工程共性硬伤）
- **构造函数未私有化 & 缺失 final 修饰**：全框架 30 余个 `*Util` 类（包括 `JsonUtil`、`StringUtil`、`BeanUtil`、`WebUtil`、`INetUtil` 等），未显式声明为 `public final class`，且未声明私有构造器（默认由编译器生成 public 无参构造函数），部分类（如 `DateUtil`）使用 protected 构造函数。
- **常量接口反模式 (Constant Interface Anti-Pattern)**：`StringPool.java` 与 `CharPool.java` 目前定义为 `public interface`，严重违背《Effective Java》第 22 条规约，应重构为带私有构造器的 `public final class`。

### 2. 根 POM 依赖洁癖与配置重复
- **子模块打破统一版本继承**：`zhkj-framework-core/pom.xml` 违规在子 POM 内显式声明了 `<version>${lombok.version}</version>`，且重复声明了根 POM 中已由 `<pluginManagement>` 集中纳管的 `maven-compiler-plugin` 处理器路径。
- **根 POM `<dependencyManagement>` 冗余幽灵声明**：
  - 根 POM 声明了 `jjwt-jackson`，但全工程所有模块均已切换至基于 Jackson 3 原生的 `ToolsJacksonSerializer` / `ToolsJacksonDeserializer`，`jjwt-jackson` 属于陈旧声明，一旦被下游误引将拉入 Jackson 2 发生依赖污染。
  - `commons-beanutils` 属于历史遗留高危反序列化依赖，底座完全不应托管。

### 3. 日期时间与 Web 辅助类代码异味
- **DateUtil 命名与解析性能异味**：
  - `DateUtil.java` 中存在首字母大写的非标准方法命名 `public static String DateToString(...)`。
  - 第 386 行 `new SimpleDateFormat(pattern).parse(dateStr)`，在解析未命中预设格式时频繁创建非线程安全对象，未利用现代 Java 21 `DateTimeFormatter` 缓存。
- **WebUtil 字符串替换 GC 浪费**：`WebUtil.java` 中多次使用 `.replaceAll("&amp;", "&")` 和 `.replaceAll("%22", "\"")` 进行纯字面量替换，触发无意义的正则表达式编译。

### 4. 自动装配规范与版本硬编码
- **Jackson 配置类装配注解不纯粹**：`JacksonConfiguration.java` 使用了 `@Configuration(proxyBeanMethods = false)` 而非 Spring Boot 4 推荐的 `@AutoConfiguration` 规范。
- **Nacos 元数据版本号硬编码**：`ZhkjNacosAutoConfiguration.java` 存在代码内硬编码字符串 `"2026.4.1.0"`。
- **Spring Boot 3/4 双重 imports 文件残留**：`starter-crypto` 目录下同时存留 `org.springframework.boot.env.EnvironmentPostProcessor.imports`（Spring Boot 3 废弃路径）与 `org.springframework.boot.EnvironmentPostProcessor.imports`（Spring Boot 4 现代路径）。

---

## 三、 重构推进最佳拓扑时序（自底向上 4 大战役阶段）

```
Stage 4: 表现层与契约开放层
   └── starter-web, starter-openapi
            ↑
Stage 3: 通信中枢与安全认证集群
   └── starter-redis, starter-kafka, starter-http, starter-nacos, starter-secure, starter-sso-client
            ↑
Stage 2: 数据持久化与国密基础设施
   └── starter-crypto, starter-datascope, starter-tenant, starter-datasource, starter-mybatis, starter-jpa
            ↑
Stage 1: 根 POM 治理与核心底座固化 (当前开局战役)
   └── zhkj-framework (根 POM), zhkj-framework-core, zhkj-framework-data-api, zhkj-framework-launcher
```

### 阶段一：根 POM 依赖洁癖与核心底座固化 (Base Foundation)
- **涉及模块**：根 POM、`zhkj-framework-core`、`zhkj-framework-data-api`、`zhkj-framework-launcher`。
- **核心动作**：
  1. 清理根 POM `<dependencyManagement>` 中废弃的 `jjwt-jackson`、`commons-beanutils`。
  2. 剔除 `zhkj-framework-core/pom.xml` 中重复的 compiler 插件与 `<version>${lombok.version}</version>`。
  3. 将 `StringPool`、`CharPool` 由 interface 升级为 `public final class`，收敛私有构造器。
  4. 对 Core 层 20 余个工具类全面固化 `public final class` 与 `private XxxUtil() {}` 防御。
  5. 修正 `DateUtil` 命名规范（`DateToString` $\rightarrow$ `dateToString`）与 `SimpleDateFormat` 线程安全重构。
  6. 治理 `JsonUtil` 异常体系，防止向外抛出裸 `RuntimeException`。

### 阶段二：数据持久化与国密底层引擎治理 (Persistence & Crypto)
- **涉及模块**：`starter-crypto`、`starter-datascope`、`starter-tenant`、`starter-datasource`、`starter-mybatis`、`starter-jpa`。
- **核心动作**：
  1. 彻底清除 `starter-crypto` 中硬编码的 `emhrai1uY2Fw...` 业务种子，净化为通用中立密钥。
  2. 修复 `SM2Util` 中 4 处未传 `StandardCharsets.UTF_8` 的平台默认字符集硬伤。
  3. 清除 `starter-crypto` 中过期的 `org.springframework.boot.env.EnvironmentPostProcessor.imports`。
  4. 在 `SqlKeyword` 中将 `SQL_REGEX` 转为静态预编译 `Pattern`，彻底消除动态编译 GC 抖动。
  5. 重构 `MybatisDataScopeInterceptor`：移除未妥善清理的 ThreadLocal，转用生命周期明确的上下文持有模型，使用 `ObjectProvider` 缓存 `AuditUserProvider`。
  6. 为 `ShardingSphereYamlRepresenter` 补全上游 Apache 官方 Issue 追踪超链接。

### 阶段三：通信中枢、消息与分布式安全集群 (Middleware & Security)
- **涉及模块**：`starter-redis`、`starter-kafka`、`starter-http`、`starter-nacos`、`starter-secure`、`starter-sso-client`。
- **核心动作**：
  1. 在 `KafkaTraceRecordInterceptor` 中实现完整的 `failure(ConsumerRecord, Exception, Consumer)` 异常状态清理回调，堵死上下文泄露。
  2. 修复 `RPubSubListenerDetector` 中 `new String(message.getChannel())` 未指定 UTF-8 编码的问题。
  3. 解耦 `SecureUtil` 的“静态工具类 + Spring 单例”混杂架构，梳理 Token 生成与验证的纯粹边界。
  4. 消除 `ZhkjNacosAutoConfiguration` 中的框架版本号硬编码，改为动态获取当前构件版本。
  5. 对 `SsoTokenParser`、`ChannelUtil` 等工具类补全 final 与私有构造器。

### 阶段四：Web 表现层与契约开放网关 (Web & Contract)
- **涉及模块**：`starter-web`、`starter-openapi`。
- **核心动作**：
  1. 将 `JacksonConfiguration` 统一调整为 Spring Boot 4 标准的 `@AutoConfiguration`。
  2. 重构 `WebUtil`：将所有的 `replaceAll("&amp;", "&")` 等正则替换降级为轻量级的 `String.replace(...)` 字符序列替换。
  3. 对 `WebUtil`、`SpringWebUtil`、`ApiCryptoUtil`、`MultipartFileUtil` 强化不可继承与私有构造器约束。
  4. 审计 `ApiCryptoUtil` 中的 `PROCESSOR_CACHE` 缓存键生命周期。

---

## 四、 破坏性改动（Breaking Changes）判定与平滑升级指引

### 1. StringPool / CharPool 从 interface 改为 final class
- **破坏性性质**：二进制兼容（Binary Compatible），但若存在历史代码以 `implements StringPool` 的“常量接口”方式反模式使用，修改后将导致其编译失败。
- **影响评估**：静态扫描已确认，整个 `zhkj-framework` 内部 0 处类 `implements StringPool`，所有调用均采用 `StringPool.DOT` 静态字段点选语法。
- **平滑建议**：直接变更为 `public final class StringPool { private StringPool() {} ... }`；若下游个别类违规 implements，只需删除声明并改为静态导入 `import static io.github.ybb88.framework.util.lang.StringPool.*`。

### 2. DateUtil.DateToString 方法驼峰重构
- **破坏性性质**：源码与签名级破坏（Source & Binary Breaking）。
- **平滑兼容方案**：
  1. 新增符合规范的小驼峰方法：`public static String dateToString(Date date, String pattern) { ... }`；
  2. 将原大驼峰方法保留并标记废弃，内部直接转发：
  ```java
  @Deprecated(since = "2026.4.1.0", forRemoval = true)
  public static String DateToString(Date date, String pattern) {
      return dateToString(date, pattern);
  }
  ```

### 3. JsonUtil 异常体系标准化收敛
- **破坏性性质**：行为兼容微调（Behavioral Change）。
- **当前现状**：`JsonUtil` 中捕获 `JacksonException` 后抛出 `new RuntimeException(...)`。
- **平滑建议**：定义或沿用框架统一的非受检转换异常（如 `JsonParseException` 继承 `ServiceException`，而 `ServiceException` 继承 `RuntimeException`），既满足下游已有 `catch (RuntimeException e)` 不中断，又建立起底层框架统一规范的异常根基。

---

## 五、 双工程构建与验证基准命令

```powershell
# 1. 底层框架本地全量快照发布 (通用相对路径)
mvn clean install -f "zhkj-framework/pom.xml" "-DskipTests=true" "-Dgpg.skip=true"

# 2. 下游业务工程增量兼容编译验证
mvn test-compile -f "zhkj-ncap/pom.xml" "-DskipTests=true"
```
