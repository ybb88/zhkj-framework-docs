# 核心基础模块：zhkj-framework-core

> **模块定位**: 框架通用基础模块，提供与容器无关的工具类、统一响应协议 `R<T>`、上下文门面 `FrameworkContext`、脱敏与校验注解，以及 Spring 静态入口 `SpringUtil`。  
> **适用基准**: OpenJDK 21, Spring Boot 4.1.1（根 pom `spring-boot.version`）, Jackson 3（`tools.jackson`，根 pom `tools.jackson.version`）  
> **模块坐标**: `io.github.ybb88:zhkj-framework-core`

---

## 📋 1. 架构定位与职责

`zhkj-framework-core` 是依赖拓扑的最底层模块，其他 Starter 与 `zhkj-framework-data-api` 均依赖它。

- **依赖范围**: compile 范围依赖 `slf4j-api`、`jackson-databind` / `jackson-core`（`tools.jackson`）。`spring-core`、`spring-context`、`spring-boot-autoconfigure`、`jakarta.validation-api`、`jspecify` 均为 `optional`，仅供工具类与自动装配按需使用。Lombok 为 `provided` 编译期依赖。
- **无容器耦合**: 源码不包含 `jakarta.servlet`、MyBatis、JPA、Hibernate 的 import。租户与安全上下文通过类名反射探测（见 `FrameworkContext`），不产生编译期依赖。
- **自动装配**: 仅 `CoreAutoConfiguration` 一个配置类，注册于 `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`，注册 `SpringUtil` Bean。开关为 `zhkj.core.spring-util-enabled`（默认 `true`）。

---

## 📦 2. 依赖坐标

```xml
<dependency>
    <groupId>io.github.ybb88</groupId>
    <artifactId>zhkj-framework-core</artifactId>
</dependency>
```

---

## 🧱 3. 包结构

```
io.github.ybb88.framework
├── api/                        # 统一响应与错误码
│   ├── R.java                  # 统一响应载体 R<T>
│   ├── IResultCode.java        # 错误码接口 (getCode / getMessage)
│   └── ResultCode.java         # 通用状态码枚举
├── config/
│   └── CoreAutoConfiguration.java
├── constant/                   # 常量类 (public final class + 私有构造器)
│   ├── AppConstant.java        # 环境、系统属性键名
│   ├── CommonConstant.java
│   ├── CryptoConstant.java     # 算法名、密文版本前缀
│   ├── HeaderConstant.java     # HTTP 请求头名称
│   ├── RoleConstant.java       # 角色编码
│   ├── TokenConstant.java      # Token 载荷字段名
│   ├── TraceConstant.java      # TraceId 键名
│   └── ZhkjConstant.java       # 通用业务常量与默认消息
├── context/                    # 上下文门面
│   ├── ContextSnapshot.java    # 上下文快照 (Lombok @Builder)
│   ├── ContextSnapshotProvider.java  # 领域上下文 SPI
│   ├── ContextScope.java       # AutoCloseable 作用域
│   └── FrameworkContext.java   # 捕获 / 恢复 / 清理门面
├── desensitize/                # 展示态脱敏
│   ├── annotation/Desensitize.java
│   ├── introspect/ZhkjAnnotationIntrospector.java
│   ├── serializer/DesensitizeJsonSerializer.java
│   ├── DesensitizeContext.java # 线程级脱敏开关
│   ├── DesensitizeRule.java
│   ├── DesensitizeRuleProvider.java  # 规则 SPI
│   ├── DefaultStaticRuleProvider.java
│   ├── DesensitizeType.java
│   └── DesensitizeUtil.java
├── exception/
│   ├── BaseFrameworkException.java   # 携带 IResultCode 的运行时异常基类
│   ├── ServiceException.java         # 业务异常
│   └── JsonParseException.java
├── model/                      # 通用模型
│   ├── BaseModel.java / BaseDto.java / BaseVo.java
│   ├── PageQuery.java          # 分页请求 (pageNo / pageSize，pageSize 上限 1000)
│   ├── BasePage.java           # 带创建、更新时间区间的分页请求
│   └── PageResult.java         # 分页结果
├── rpc/RpcRequestAttribute.java  # 服务端授权请求属性键 (DESENSITIZE_AUTHORIZED、TENANT_DELEGATION_AUTHORIZED)
├── rpc/header/
│   ├── RpcHeaderProvider.java  # 跨服务请求头供给 SPI
│   └── CompositeRpcHeaderProvider.java
├── spi/
│   └── BlindIndexProvider.java # 盲索引算法 SPI
├── support/
│   ├── BaseBeanCopier.java     # 基于 Spring 重打包 CGLIB 的 Bean 拷贝器
│   ├── BeanProperty.java
│   ├── FastStringWriter.java
│   ├── Kv.java                 # 链式键值对 (LinkedCaseInsensitiveMap)
│   └── StrFormatter.java
├── validation/                 # Bean Validation 注解
│   ├── Mobile.java / IdCard.java / CreditCode.java
│   └── validator/MobileValidator.java, IdCardValidator.java, CreditCodeValidator.java
└── util/                       # 单数命名工具类
    ├── BeanUtil, CacheUtil, ClassUtil, DigestUtil, IdUtil, IoUtil, JsonUtil,
    │   NumberUtil, ObjectUtil, PlaceholderUtil, PropUtil, RoleUtil, SpringUtil,
    │   StringUtil, TraceUtil, UrlUtil, ValidatorUtil
    ├── codec/HexUtil.java
    ├── collection/CollectionUtil.java
    ├── crypto/AesUtil.java, DesUtil.java (@Deprecated), BlindIndexUtil.java
    ├── date/DateUtil.java
    ├── exception/ExceptionUtil.java
    ├── io/FileUtil.java
    ├── lang/CharPool.java, Charsets.java, Holder.java, StringPool.java
    ├── random/RandomType.java
    └── tree/TreeAncestorResolver.java
```

---

## 🔑 4. 核心 API 清单

### 4.1 统一响应 `R<T>`（`io.github.ybb88.framework.api.R`）

字段：`int code`、`boolean success`、`T data`、`String msg`。

| 静态方法 | 说明 |
| :--- | :--- |
| `R.data(T data)` / `R.data(T data, String msg)` | 成功响应，code = 200 |
| `R.data(int code, T data, String msg)` | 指定 code；`data == null` 时 msg 替换为 `ZhkjConstant.DEFAULT_NULL_MESSAGE` |
| `R.success(String msg)` / `R.success(IResultCode)` / `R.success(IResultCode, String msg)` | 成功响应，无数据 |
| `R.fail(String msg)` / `R.fail(int code, String msg)` | 失败响应；`fail(String)` 使用 `ResultCode.FAILURE` |
| `R.fail(IResultCode)` / `R.fail(IResultCode, String msg)` | 失败响应，携带错误码 |
| `R.status(boolean flag)` | `true` 返回默认成功消息，`false` 返回默认失败消息 |
| `R.isSuccess(R<?>)` / `R.isNotSuccess(R<?>)` | 按 code 判断，入参为 `null` 时 `isSuccess` 返回 `false` |

### 4.2 错误码

`IResultCode` 接口（继承 `Serializable`）：`int getCode()`、`String getMessage()`。

`ResultCode` 枚举（`io.github.ybb88.framework.api.ResultCode`）：

| 常量 | code | 默认消息 |
| :--- | :--- | :--- |
| `SUCCESS` | 200 | 操作成功 |
| `PART_SUCCESS` | 200 | 部分操作成功 |
| `FAILURE` | 400 | 业务异常 |
| `UN_AUTHORIZED` | 401 | 请求未授权 |
| `REQ_REJECT` | 403 | 请求被拒绝 |
| `NOT_FOUND` | 404 | 404 没找到请求 |
| `METHOD_NOT_SUPPORTED` | 405 | 不支持当前请求方法 |
| `MEDIA_TYPE_NOT_SUPPORTED` | 415 | 不支持当前媒体类型 |
| `INTERNAL_SERVER_ERROR` | 500 | 服务器异常 |
| `MSG_NOT_READABLE` | 400 | 消息不能读取 |
| `PARAM_MISS` | 400 | 缺少必要的请求参数 |
| `PARAM_TYPE_ERROR` | 400 | 请求参数类型错误 |
| `PARAM_BIND_ERROR` | 400 | 请求参数绑定错误 |
| `PARAM_VALID_ERROR` | 400 | 参数校验失败 |

业务错误码由业务模块实现 `IResultCode` 定义（参见各业务工程的错误码枚举）。

### 4.3 异常

| 类型 | 说明 |
| :--- | :--- |
| `BaseFrameworkException` | `RuntimeException` 基类，`getResultCode()` 返回 `IResultCode`（为空时回落 `ResultCode.FAILURE`） |
| `ServiceException extends BaseFrameworkException` | 业务异常，构造器接受 `String` 或 `IResultCode`（可附加 message、cause） |
| `JsonParseException extends ServiceException` | JSON 解析异常 |

`starter-web` 的 `GlobalExceptionHandler` 按 `BaseFrameworkException` 的错误码映射 HTTP 状态（见 `starter-web.md`）。

### 4.4 上下文门面 `FrameworkContext`

`ContextSnapshot` 字段（均可通过 Lombok `@Builder(toBuilder = true)` 构造）：

| 分组 | 字段 |
| :--- | :--- |
| 租户与维度 | `tenantId`、`dimensionId`、`delegatedTenantId`、`grantTicket` |
| 客户端 | `clientId` |
| 操作人 | `userId`（Long）、`userAccount`、`userName`、`deptId`（Long） |
| 权限 | `roles`、`permissions`（`List<String>`，默认空列表） |
| 扩展 | `attributes`（`Map<String, Object>`），`addAttribute` / `getAttribute` |

`FrameworkContext` 静态方法：

| 方法 | 说明 |
| :--- | :--- |
| `capture()` | 按 `ContextSnapshotProvider` 顺序采集快照；若 SPI 未采集到租户或用户身份，则回落到按类名反射探测 `starter-tenant`（`TenantContext`）、`starter-secure`（`SecurityContextHolder`）的上下文；随后与当前线程快照合并（仅补充为空的字段）。快照为空时返回 `null` |
| `restore(ContextSnapshot)` | 恢复快照到当前线程；`null` 或空快照等同 `clearAll()` |
| `openScope(ContextSnapshot)` → `ContextScope` | 保存当前快照并恢复目标快照；`close()` 时还原 |
| `runWith(snapshot, Runnable)` / `callWith(snapshot, Callable<V>)` | 在目标快照下执行，结束后还原 |
| `clearAll()` / `clear()` | 清空线程本地快照；依次调用各已注册 Provider 的 `clear()`，清理 `DesensitizeContext`，反射清理 `starter-secure`、`starter-tenant`、`starter-sso-client`、`starter-datascope` 的上下文持有器，最后执行自定义清理器。不移除已注册的 Provider 与清理器 |
| `registerProvider(ContextSnapshotProvider)` / `registerSnapshotProvider(...)` | 注册 Provider，按 `getOrder()` 升序 |
| `registerCleaner(Runnable)` | 注册清理回调 |
| `getSnapshot()` | 返回线程本地快照（不做 SPI 采集） |
| `getTenantId()` / `setTenantId(String)` 等 | 租户、维度、委托、票据、clientId 的读写门面 |

`ContextSnapshotProvider` SPI 方法：`void capture(ContextSnapshot.ContextSnapshotBuilder)`、`void restore(ContextSnapshot)`、`void clear()`、`default int getOrder()`（默认 0）。实现类通过 `META-INF/services/io.github.ybb88.framework.context.ContextSnapshotProvider` 注册，由 `FrameworkContext` 静态初始化时经 `ServiceLoader` 加载。

### 4.5 常量

| 类 | 主要常量 |
| :--- | :--- |
| `HeaderConstant` | `X_TRACE_ID`(`X-Trace-Id`)、`X_TENANT_ID`(`X-Tenant-Id`)、`X_DIMENSION_ID`、`X_DELEGATED_TENANT_ID`、`X_GRANT_TICKET`、`X_USER_ID`、`X_USER_ACCOUNT`、`X_USER_NAME`、`X_DEPT_ID`、`X_INTERNAL_RPC`(`X-Internal-RPC`)、`AUTHORIZATION`、`ZHKJ_AUTH`、`BLADE_AUTH`、`BEARER_PREFIX`、`X_CLIENT_ID`、`X_TICKET_NO`、`X_APP_KEY`、`X_TIMESTAMP`、`X_NONCE`、`X_SIGN`、`COOKIE_SSO_SESSION`、`COOKIE_SSO_TOKEN`。历史兼容别名：`TENANT_ID`(`tenant-id`)、`ZHKJ_TENANT_ID`、`ZHKJ_USER_ID`、`ZHKJ_USER_ACCOUNT`、`ZHKJ_USER_NAME`、`X_TENANT_ID_UPPER`(`X-Tenant-ID`) |
| `TraceConstant` | `TRACE_ID`(MDC 键 `traceId`)、`HTTP_HEADER_TRACE_ID`、`KAFKA_HEADER_TRACE_ID`（均为 `X-Trace-Id`）、`TRACE_ID_LENGTH = 32` |
| `CryptoConstant` | `ALGO_SM4`、`ALGO_AES`、`ALGO_SM3`、`ALGO_HMAC_SM3`、`ALGO_HMAC_SHA256`、`PREFIX_SM4`(`sm4:`)、`PREFIX_V1`(`v1:`)、`PREFIX_SM4_V1`(`sm4:v1:`)、`SUFFIX_HASH`(`Hash`)、`SUFFIX_CIPHER`(`Cipher`) |
| `AppConstant` | `DEV_CODE`(`dev`)、`UAT_CODE`(`test`)、`PRD_CODE`(`prod`)、`SYSTEM_ENV`(`zhkj.env`)、`SYSTEM_NAME`(`zhkj.name`)、`SYSTEM_GROUP`(`zhkj.group`)、`SYSTEM_ZHKJ`(`zhkj`)、`OS_NAME_LINUX`(`LINUX`)、`APPLICATION_VERSION`(`3.0.0`) |
| `ZhkjConstant` | `DEFAULT_SUCCESS_MESSAGE`(`操作成功`)、`DEFAULT_FAILURE_MESSAGE`、`DEFAULT_NULL_MESSAGE`(`暂无承载数据`)、`ADMIN_TENANT_ID`(`00000000`)、`TOP_PARENT_ID`(0L)、`DB_IS_DELETED`(1) 等 |
| `RoleConstant` | `ADMIN`(`administrator`)、`USER`(`user`)、`TEST`(`test`)、`SUPER_ADMIN`(`super_admin`)、`SYS_ADMIN`(`sys_admin`)、`SEC_ADMIN`(`sec_admin`)、`AUDIT_ADMIN`(`audit_admin`) |

### 4.6 展示态脱敏

- `@Desensitize`（`io.github.ybb88.framework.desensitize.annotation.Desensitize`）：属性 `type`（默认 `CUSTOM`）、`rule`（默认 `CUSTOM`）、`prefixKeep`、`suffixKeep`（仅 `CUSTOM` 生效）、`mask`（默认 `*`）。注解标注 `@JsonSerialize(using = DesensitizeJsonSerializer.class)`。
- `DesensitizeType` 共 7 种：`CHINESE_NAME`、`ID_CARD`、`MOBILE_PHONE`、`BANK_CARD`、`EMAIL`、`PASSWORD`、`CUSTOM`。
- `DesensitizeContext`：线程级开关，`disable()`、`enable()`、`isDisabled()`、`clear()`。`starter-web` 的 `InternalRpcDesensitizeFilter` 使用该开关。
- `DesensitizeUtil`：`mobilePhone`、`idCard`、`bankCard`、`password`、`chineseName`、`email`、`maskAround(str, prefixKeep, suffixKeep)`、`desensitize(origin, DesensitizeType)`、`autoDesensitize(fieldName, value)`。
- `DesensitizeRuleProvider` SPI 与 `DefaultStaticRuleProvider`（默认实现）负责按字段解析规则。

### 4.7 校验注解

| 注解 | 校验器 | 默认消息 |
| :--- | :--- | :--- |
| `@Mobile` | `MobileValidator` | 手机号码格式不合法 |
| `@IdCard` | `IdCardValidator` | 身份证号码格式不合法或校验码错误 |
| `@CreditCode` | `CreditCodeValidator` | 统一社会信用代码格式不合法或校验码错误 |

### 4.8 工具类（单数命名）

| 类 | 常用方法 |
| :--- | :--- |
| `JsonUtil` | `toJson(Object)`、`toJson(Map)`、`parse(String, Class<T>)`、`toBean`、`toListBean`、`toMap`、`readTree`、`toJsonNode`、`getJsonMapper()` |
| `IdUtil` | `randomUUID()`、`simpleUUID()`、`fastUUID()`、`fastSimpleUUID()` |
| `DigestUtil` | `md5Hex(String / byte[])`、`sha1`、`sha256(String)`（返回十六进制字符串）、`sha256(byte[])`（返回 `byte[]`）、`sha384`、`sha512`、`hash(algorithm, str)`、`toHex`、`slowEquals`、`encrypt` |
| `BeanUtil` | `copy(source, Class<T>)`、`copy(source, target)`、`copyProperties`、`toMap`、`toBean`、`getProperty`、`setProperty`、`newInstance` |
| `StringUtil` | `isBlank`、`isNotBlank`、`hasText`、`isEmpty`、`join`、`cleanChars`、`cleanIdentifier`、`random(count[, RandomType])`、`underlineToHump`、`humpToUnderline`、`toStrArray` 等 |
| `ObjectUtil` | `isEmpty`、`isNotEmpty`、`nullSafeEquals` |
| `CollectionUtil` | `isEmpty`、`isNotEmpty`（Collection / Map）、`contains` |
| `NumberUtil` | `toInt`、`toLong`、`toDouble`、`toBoolean`、`to62String`、`toLongList` |
| `DateUtil`（`util.date`） | `PATTERN_DATETIME`、`PATTERN_DATE`、`now`、`plusDays` 等、`getDateTimeFormatter(pattern)` |
| `SpringUtil` | `getBean(Class)`、`getBean(String)`、`getBean(String, Class)`、`getContext()`、`getProperty(key[, default])`、`publishEvent` |
| `CacheUtil` | `get`、`put`、`evict`、`clear`（基于 Spring `CacheManager`，按缓存名访问；`CacheUtil.SYS_CACHE` 常量值为 `sys`） |
| `TraceUtil` | `getTraceId`、`setTraceId`、`generateTraceId`、`getOrCreateTraceId`、`clearTraceId` |
| `AesUtil`（`util.crypto`） | `genAesKey`、`encryptToHex`、`encryptToBase64`、`decryptFormHexToString`、`decryptFormBase64ToString`、`encrypt`/`decrypt`（`byte[]`） |
| `DesUtil`（`util.crypto`，**废弃兼容类**，`@Deprecated(since = "2026.4.1.0")`，不推荐用于新代码） | `genDesKey`、`encryptToHex`、`decryptFormHex`、`encryptToBase64`、`decryptFormBase64`、`encrypt`/`decrypt`（`byte[]`） |
| `BlindIndexUtil`（`util.crypto`） | `generateIndex(plainText[, salt])`、`generateSm3Index`、`generateFuzzyIndices`、`registerProvider`。未传入盐值时经 `SpringUtil.getProperty` 读取配置键 `zhkj.storage.crypto.blind-index-salt`；该键未配置或为空时抛出 `IllegalStateException`（源码无内置默认盐）。配置说明见 `starter-crypto` |
| `HexUtil`（`util.codec`） | `encodeToString`、`decode`、`decodeToString` |
| `FileUtil`（`util.io`） | `list`、`getFileExtension`、`readToString`、`writeToFile`、`toFile` |
| `ExceptionUtil`（`util.exception`） | `unchecked`、`unwrap`、`getStackTraceAsString` |
| `TreeAncestorResolver`（`util.tree`） | `parseAncestors`、`resolveBottomUp`、`resolveTopDown`、`clearCache` |
| `RoleUtil` | `isPrivileged(String roleCodes)` |
| `PropUtil` | `getProperty(Properties, key, defaultValue)`、`setProperty(Properties, key, value)`（仅当原值为空白时写入） |

---

## 🛡️ 5. 编码规约（本模块约束）

### 5.1 工具类单数命名与私有构造器
- 工具类统一单数命名（`JsonUtil`、`StringUtil`、`IdUtil`、`DigestUtil`），不新增 `*Utils` 类。
- 规约要求工具类声明为 `public final class` 并提供私有构造器。
- 例外：`SpringUtil` 声明为 `public final class`，但源码未提供私有构造器，由 `CoreAutoConfiguration` 通过 `new SpringUtil()` 实例化为 Bean [待确认：该例外是否需在规约中豁免]。

### 5.2 静态正则预编译
- `StringUtil.cleanChars` 与 `StringUtil.cleanIdentifier` 使用静态 `Pattern` 常量，方法体内不动态编译正则。
- 纯字面量替换优先使用 `String.replace`。

### 5.3 字符集
- 涉及字节与字符串互转、加解密、网络传输时，显式指定 `StandardCharsets.UTF_8`（例如 `AesUtil` 的 `DEFAULT_CHARSET`）。

---

## 💡 6. 典型使用范式

### 6.1 统一响应
```java
import io.github.ybb88.framework.api.R;
import io.github.ybb88.framework.api.ResultCode;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class UserController {

    @GetMapping("/api/users/{id}")
    public R<UserVo> getUser(@PathVariable Long id) {
        UserVo vo = new UserVo();
        vo.setId(id);
        return R.data(vo);
    }

    @GetMapping("/api/users/check")
    public R<Void> checkStatus() {
        return R.fail(ResultCode.UN_AUTHORIZED);
    }
}
```

> 在 Servlet 应用中引入 `zhkj-framework-starter-web` 后，`ZhkjResponseBodyAdvice` 会自动包装非 `R` 返回值，此时可直接返回业务对象（见 `starter-web.md`）。

### 6.2 上下文跨线程传递
```java
import io.github.ybb88.framework.context.ContextScope;
import io.github.ybb88.framework.context.ContextSnapshot;
import io.github.ybb88.framework.context.FrameworkContext;

// 1. 在提交任务的线程中捕获快照
ContextSnapshot snapshot = FrameworkContext.capture();

// 2. 在工作线程中恢复，作用域结束自动还原
executor.execute(() -> {
    try (ContextScope scope = FrameworkContext.openScope(snapshot)) {
        bizService.processData();
    }
});
```

### 6.3 JSON、ID、摘要、拷贝
```java
String json = JsonUtil.toJson(user);
UserVo userVo = JsonUtil.parse(json, UserVo.class);

String traceId = IdUtil.simpleUUID();

String md5Hex = DigestUtil.md5Hex("rawPassword");
String sha256Hex = DigestUtil.sha256("rawContent");
byte[] fileHash = DigestUtil.sha256(fileBytes);

UserVo targetVo = BeanUtil.copy(sourceEntity, UserVo.class);
```

### 6.4 对称加密与盲索引
```java
String cipher = AesUtil.encryptToHex(plainText, aesKey);
String plain = AesUtil.decryptFormHexToString(cipher, aesKey);

String blindHash = BlindIndexUtil.generateIndex(plainText);  // 需已配置 zhkj.storage.crypto.blind-index-salt，否则抛出 IllegalStateException
```

### 6.5 字符串清洗
```java
String cleanText = StringUtil.cleanChars("  张 三 \t \v ");       // "张三"
String safeColumn = StringUtil.cleanIdentifier("user_name; DROP TABLE--"); // "user_nameDROPTABLE"
```

### 6.6 链式键值对
```java
Kv param = Kv.create()
        .set("userId", 1001L)
        .set("roleCode", "ADMIN");
String json = JsonUtil.toJson(param);
```

### 6.7 脱敏与校验
```java
import io.github.ybb88.framework.desensitize.DesensitizeType;
import io.github.ybb88.framework.desensitize.annotation.Desensitize;
import io.github.ybb88.framework.validation.Mobile;
import lombok.Data;

@Data
public class CustomerVo {

    @Desensitize(type = DesensitizeType.CHINESE_NAME)
    private String name;

    @Desensitize(type = DesensitizeType.MOBILE_PHONE)
    private String mobile;

    @Mobile
    private String contactMobile;
}
```

---

## 🔗 7. 与其它模块的关系

- 上层 `zhkj-framework-data-api`、`starter-web`、`starter-http`、`starter-redis`、`starter-kafka`、`starter-jpa`、`starter-datasource`、`starter-openapi` 等均依赖本模块。
- `FrameworkContext` 在运行期探测 `io.github.ybb88.framework.tenant.context.TenantContext`（`starter-tenant`）与 `io.github.ybb88.framework.secure.context.SecurityContextHolder`（`starter-secure`），两者不存在时探测静默跳过。
- `FrameworkContext.clearAll()` 另反射清理 `io.github.ybb88.framework.sso.client.context.SsoContextHolder`（`starter-sso-client`）与 `io.github.ybb88.framework.datascope.context.DataScopeContext`（`starter-datascope`），类不存在时静默跳过。
- `RpcHeaderProvider` SPI 由 `starter-http` 的 `HttpClientHeaderInterceptor` 消费。
