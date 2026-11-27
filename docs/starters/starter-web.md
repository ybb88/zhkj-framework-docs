# Web MVC 增强组件：zhkj-framework-starter-web

> **模块定位**: Servlet 环境下 Spring MVC 运行时增强，提供统一响应包装、全局异常映射、Jackson 序列化规则、XSS 过滤、TraceId 与上下文清理过滤器、内部 RPC 脱敏旁路、接口审计注解与请求日志切面。  
> **适用基准**: OpenJDK 21 (LTS), Spring Boot 4.1.1, Spring Framework 7.x  
> **模块坐标**: `io.github.ybb88:zhkj-framework-starter-web`

---

## 📋 1. 架构定位与核心职责

1. **统一响应包装**: `ZhkjResponseBodyAdvice` 对未标注 `@BaseResponseIgnore` 的 Controller 返回值包装为 `R<T>`。
2. **全局异常映射**: `GlobalExceptionHandler` 将框架异常、参数校验异常、HTTP 协议异常与未捕获异常映射为 `R` 与 HTTP 状态码（见第 5 节）。
3. **Jackson 序列化规则**: `JacksonConfiguration` 注册 `JsonMapper`，统一 `Long` 转字符串、时间格式与 `@Desensitize` 内省器。
4. **XSS 清洗**: 默认关闭，需显式开启 `zhkj.xss.enabled=true`。
5. **链路与上下文**: `TraceIdFilter` 维护 `X-Trace-Id`；`FrameworkContextClearFilter` 在请求结束时清空 `FrameworkContext`。
6. **内部 RPC 脱敏旁路**: `InternalRpcDesensitizeFilter` 仅在服务端 `RpcRequestAttribute.DESENSITIZE_AUTHORIZED` 请求属性为 `Boolean.TRUE` 时临时关闭展示态脱敏，`X-Internal-RPC` 不提供授权。
7. **审计与请求日志**: `@AuditLog` 切面发布 `AuditLogEvent`；`RequestLogAspect` 仅在 `dev` 与 `test` Profile 下生效。
8. **异步线程池**: `ZhkjExecutorConfiguration` 提供 `taskExecutor`，并透传 MDC 与 `FrameworkContext` 快照。
9. **接口报文加解密**: `ApiCryptoConfiguration` 装配 `@ApiEncrypt` / `@ApiDecrypt` 相关 Advice 与参数解析器（依赖 `starter-crypto`）。

> [!CAUTION]
> 本模块仅适用于 Servlet WebMVC 应用。Spring Cloud Gateway（WebFlux）工程不应引入本模块。

---

## 📦 2. 依赖坐标

```xml
<dependency>
    <groupId>io.github.ybb88</groupId>
    <artifactId>zhkj-framework-starter-web</artifactId>
</dependency>
```

依赖要点（来自 `pom.xml`）:

| 依赖 | 范围 | 说明 |
| :--- | :--- | :--- |
| `zhkj-framework-core` | compile | 基础模型与工具 |
| `spring-boot-starter-web` | compile | Spring MVC |
| `spring-boot-starter-aspectj`、`spring-boot-starter-validation`、`spring-boot-starter-actuator` | compile | 切面、校验、监控 |
| `zhkj-framework-starter-crypto` | optional | 报文加解密注解与 Advice 所需 |
| `zhkj-framework-starter-tenant` | optional | 可选 |

---

## 🧱 3. 包结构

```
io.github.ybb88.framework.web
├── advice/ZhkjResponseBodyAdvice.java         # 统一响应包装
├── annotation/BaseResponseIgnore.java         # 跳过统一包装
├── audit/
│   ├── annotation/AuditLog.java               # 审计注解
│   ├── aspect/AuditLogAspect.java             # 审计切面
│   ├── enums/BusinessType.java                # 业务操作类型
│   └── event/AuditLogEvent.java               # 审计事件
├── config/
│   ├── WebAutoConfiguration.java              # 主自动配置 (@Import 其它配置)
│   ├── JacksonConfiguration.java
│   ├── XssConfiguration.java
│   ├── ZhkjExecutorConfiguration.java
│   └── ZhkjWebMvcConfiguration.java
├── crypto/
│   ├── config/ApiCryptoConfiguration.java     # 报文加解密自动配置
│   ├── core/                                  # RequestBodyAdvice、ResponseBodyAdvice、参数解析器
│   └── util/ApiCryptoUtil.java
├── filter/
│   ├── FrameworkContextClearFilter.java
│   ├── InternalRpcDesensitizeFilter.java
│   └── TraceIdFilter.java
├── handler/GlobalExceptionHandler.java
├── jackson/ZhkjJavaTimeModule.java
├── logger/RequestLogAspect.java
├── props/
│   ├── ZhkjAsyncProperties.java
│   └── ZhkjDesensitizeProperties.java
├── support/xss/
│   ├── HtmlFilter.java
│   ├── XssFilter.java
│   ├── XssHttpServletRequestWrapper.java
│   ├── XssProperties.java
│   └── XssUrlProperties.java
├── task/MdcTaskDecorator.java
└── util/
    ├── MultipartFileUtil.java
    ├── SpringWebUtil.java
    └── WebUtil.java
```

---

## 🔁 4. 自动装配与过滤器链

`META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` 注册两个类:

- `io.github.ybb88.framework.web.config.WebAutoConfiguration`: `@Import` 了 `JacksonConfiguration`、`XssConfiguration`、`ZhkjExecutorConfiguration`、`ZhkjWebMvcConfiguration`、`GlobalExceptionHandler`、`AuditLogAspect`、`RequestLogAspect`，并注册三个 `FilterRegistrationBean`。
- `io.github.ybb88.framework.web.crypto.config.ApiCryptoConfiguration`: 见第 9 节。

| 过滤器 | 注册 Bean 名 | 顺序 | 作用 |
| :--- | :--- | :--- | :--- |
| `FrameworkContextClearFilter` | `frameworkContextClearFilterRegistration` | `HIGHEST_PRECEDENCE` | `finally` 中调用 `FrameworkContext.clearAll()` |
| `TraceIdFilter` | `traceIdFilterRegistration` | `HIGHEST_PRECEDENCE + 10` | 读取或生成 `X-Trace-Id`，写入 MDC 与响应头，请求结束清理 MDC |
| `InternalRpcDesensitizeFilter` | `internalRpcDesensitizeFilterRegistration` | `HIGHEST_PRECEDENCE + 20` | 仅在服务端 `DESENSITIZE_AUTHORIZED` 请求属性为 `Boolean.TRUE` 时关闭脱敏；否则显式启用，`finally` 恢复进入时的开关状态 |
| `XssFilter` | `xssFilterRegistration`（仅 XSS 开启时） | `LOWEST_PRECEDENCE` | 包装请求做 XSS 清洗 |

```
请求 → FrameworkContextClearFilter → TraceIdFilter → InternalRpcDesensitizeFilter → [XssFilter] → DispatcherServlet
                                                                                                 ↓
                                                                        Controller / GlobalExceptionHandler / ZhkjResponseBodyAdvice
```

---

## ⚙️ 5. 配置项

### 5.1 XSS（`XssConfiguration`、`XssProperties`、`XssUrlProperties`）

| 键 | 类型 | 默认值 | 说明 |
| :--- | :--- | :--- | :--- |
| `zhkj.xss.enabled` | Boolean | `true`（属性类） | 装配条件为 `havingValue = "true"` 且**无** `matchIfMissing`，**未显式配置时 XSS 过滤器不会注册** |
| `zhkj.xss.skip-url` | List<String> | 空 | 跳过清洗的路径（Ant 风格） |
| `zhkj.xss.url.exclude-patterns` | List<String> | 空 | 同上，另一组配置项 |

`XssFilter` 按 `getServletPath()` 与上述两组模式做匹配。非 JSON 请求的参数与报文经 `HtmlFilter` 清洗；JSON 报文体保持原文，不做 HTML 清洗。

### 5.2 异步线程池（`ZhkjAsyncProperties`，前缀 `zhkj.async`）

| 键 | 默认值 |
| :--- | :--- |
| `zhkj.async.core-pool-size` | `2` |
| `zhkj.async.max-pool-size` | `50` |
| `zhkj.async.queue-capacity` | `10000` |
| `zhkj.async.keep-alive-seconds` | `300` |

线程名前缀固定为 `async-executor-`，拒绝策略为 `CallerRunsPolicy`，任务装饰器为 `MdcTaskDecorator`。

### 5.3 脱敏（`ZhkjDesensitizeProperties`，前缀 `zhkj.desensitize`）

| 键 | 默认值 | 说明 |
| :--- | :--- | :--- |
| `zhkj.desensitize.enabled` | `true` | 总开关，作用于请求日志脱敏 |
| `zhkj.desensitize.log.enabled` | `true` | 请求日志脱敏开关 |
| `zhkj.desensitize.log.ignore-fields` | 空列表 | 日志脱敏白名单字段名 |
| `zhkj.desensitize.api.enabled` | `false` | 接口动态脱敏配置，当前代码中无读取点 |

### 5.4 报文加解密（前缀 `zhkj.api.crypto`，定义于 `starter-crypto`）

| 键 | 默认值 | 说明 |
| :--- | :--- | :--- |
| `zhkj.api.crypto.enabled` | `true` | 装配条件 `matchIfMissing = true` |
| `zhkj.api.crypto.param-name` | `data` | URL 参数解密时的参数名 |
| `zhkj.api.crypto.aes-key` / `des-key` / `sm4-key` | — | 默认密钥（按算法） |

### 5.5 示例

```yaml
zhkj:
  xss:
    enabled: true
    skip-url:
      - /doc.html
      - /webjars/**
      - /v3/api-docs/**
      - /actuator/**
  async:
    core-pool-size: 10
    max-pool-size: 50
    queue-capacity: 1000
  desensitize:
    log:
      ignore-fields:
        - mobile
```

---

## 🛡️ 6. 异常到响应的映射（`GlobalExceptionHandler`）

| 异常 | HTTP 状态 | 响应体 |
| :--- | :--- | :--- |
| `ServiceException` | 400（`@ResponseStatus`） | `R.fail(resultCode, msg)` |
| `BaseFrameworkException`（含 `DistributedLockException`） | `resultCode.getCode()`（100–599 时），否则 400 | `R.fail(resultCode, msg)` |
| `MethodArgumentNotValidException` | 400 | `PARAM_VALID_ERROR`，首个字段错误，格式 `[field] message` |
| `BindException` | 400 | `PARAM_BIND_ERROR`，首个字段错误，格式 `[field] message` |
| `ConstraintViolationException` | 400 | `PARAM_VALID_ERROR`，首个违例，格式 `[属性路径] message` |
| `MissingServletRequestParameterException` | 400 | `PARAM_MISS` |
| `MethodArgumentTypeMismatchException` | 400 | `PARAM_TYPE_ERROR` |
| `HttpMessageNotReadableException` | 400 | `MSG_NOT_READABLE` |
| `HttpRequestMethodNotSupportedException` | 405 | `METHOD_NOT_SUPPORTED` |
| `HttpMediaTypeNotSupportedException` | 415 | `MEDIA_TYPE_NOT_SUPPORTED` |
| `NoResourceFoundException` | 404 | `NOT_FOUND` |
| `NoHandlerFoundException` | 404 | `NOT_FOUND`（无自定义消息） |
| `IllegalArgumentException` | 400 | `PARAM_VALID_ERROR`，消息为异常消息 |
| `RuntimeException`（唯一键冲突） | 409 | `数据已存在，请勿重复提交` |
| `RuntimeException`（含 `getResultCode()` 方法） | `resultCode.getCode()`（100–599），否则 400 | `R.fail(resultCode, msg)` |
| 其它 `RuntimeException` | 500 | `系统繁忙，请稍后重试` |
| `Throwable` | 500 | `系统发生未知异常，请联系管理员` |

> 代码中未定义 `CryptoException` 的专门处理器，加解密异常由通用 `RuntimeException` 或 `Throwable` 分支兜底。

---

## 📄 7. 统一响应包装（`ZhkjResponseBodyAdvice`）

- `supports`: 类或方法标注 `@BaseResponseIgnore` 时不包装。
- `beforeBodyWrite`:
  - `String` 返回 `JsonUtil.toJson(R.data(body))`；
  - `Boolean` 返回 `R.status(body)`；
  - 已是 `R` 直接返回；
  - 其它返回 `R.data(body)`。

```java
@RestController
@RequestMapping("/api/users")
public class UserController {

    // 返回值由 Advice 包装为 R<User>
    @GetMapping("/{id}")
    public User getUser(@PathVariable Long id) {
        return userService.getById(id);
    }

    // 不包装，原样返回
    @BaseResponseIgnore
    @GetMapping(value = "/raw", produces = MediaType.TEXT_PLAIN_VALUE)
    public String getRaw() {
        return "raw string";
    }
}
```

---

## 🧾 8. Jackson 与时间序列化（`JacksonConfiguration`）

- Bean: `JsonMapper`（`@Primary`，缺失时注册）、`ObjectMapper`（指向同一实例）、`JacksonJsonHttpMessageConverter`。
- 规则:
  - `Long` / `long` 序列化为 JSON 字符串（防止 JavaScript 精度丢失）；
  - `Locale.CHINA`，时区为系统默认，日期格式 `yyyy-MM-dd HH:mm:ss`，禁用时间戳输出；
  - 关闭 `FAIL_ON_EMPTY_BEANS`、`FAIL_ON_UNKNOWN_PROPERTIES`；
  - 开启 `ALLOW_SINGLE_QUOTES`、`ALLOW_UNESCAPED_CONTROL_CHARS`、`ALLOW_BACKSLASH_ESCAPING_ANY_CHARACTER`；
  - 注册 `ZhkjAnnotationIntrospector`（`@Desensitize` 支持）与 `ZhkjJavaTimeModule`，并执行 `findAndAddModules()`。

---

## 🔍 9. 报文加解密（`ApiCryptoConfiguration`，需 `starter-crypto`）

装配条件: Servlet Web 应用、类路径存在 `io.github.ybb88.framework.crypto.transport.annotation.ApiDecrypt`、`zhkj.api.crypto.enabled` 为 `true`（缺省视为 `true`）。

| 组件 | 作用 |
| :--- | :--- |
| `ApiDecryptRequestBodyAdvice` | 对标注 `@ApiDecrypt` 的方法解密请求体 |
| `ApiEncryptResponseBodyAdvice` | 对标注 `@ApiEncrypt` 的方法加密响应体 |
| `ApiDecryptParamResolver` | 对标注 `@ApiDecrypt` 的参数从 `zhkj.api.crypto.param-name` 指定的请求参数解密 |

```java
@ApiEncrypt
@ApiDecrypt
@PostMapping("/secure/orders")
public R<OrderVo> create(@RequestBody OrderDto dto) {
    return R.data(orderService.create(dto));
}
```

---

## 🧭 10. 链路与上下文

- `TraceIdFilter`: 入站 `X-Trace-Id` 为空时生成 32 位 Hex 字符串；写入 MDC（键 `traceId`）与响应头 `X-Trace-Id`；请求结束清理 MDC。
- `FrameworkContextClearFilter`: 请求结束时清空 `FrameworkContext`，防止线程池复用污染。
- `MdcTaskDecorator`: 异步任务提交前捕获 MDC 与 `FrameworkContext.capture()`，执行时恢复并在结束后还原。

---

## 🛡️ 11. 内部 RPC 脱敏旁路

1. **默认脱敏**：普通 RPC 与公共展示端点继续执行 `@Desensitize`。`starter-http` 的 `DefaultRpcHeaderProvider` 自动添加的 `X-Internal-RPC: true`（`HeaderConstant.X_INTERNAL_RPC`）只是传输标记，不提供明文权限。
2. **服务端赋权**：宿主必须先完成调用方认证与目标内部端点授权，再在当前 ServletRequest 设置 `RpcRequestAttribute.DESENSITIZE_AUTHORIZED` 为 `Boolean.TRUE`。该常量位于 `io.github.ybb88.framework.rpc.RpcRequestAttribute`；不得从 HTTP Header、ThreadLocal 或异步上下文推断授权。
3. **顺序与宿主接入**：脱敏过滤器维持 `HIGHEST_PRECEDENCE + 20`。宿主认证过滤器必须先于此顺序执行，完成调用方认证与目标内部端点授权后，仅为确需明文的请求赋权；公共展示 API 不因机器凭据正确而获得明文权限，没有明文业务需求的内部端点不应赋权。宿主认证过滤器不由本 starter 自动提供。
4. **受控序列化**：仅当 `request.getAttribute(RpcRequestAttribute.DESENSITIZE_AUTHORIZED) == Boolean.TRUE` 时，`InternalRpcDesensitizeFilter` 才调用 `DesensitizeContext.disable()`，使当前受授权请求的 Jackson 序列化不再对 `@Desensitize` 字段打码。
5. **异常同样清理**：未授权请求显式启用脱敏；脱敏过滤器在 `finally` 恢复进入过滤器前的开关状态以支持嵌套调用，而非无条件 `enable()`。授予权限的宿主过滤器在包围下游过滤器链的 `finally` 中调用 `request.removeAttribute(RpcRequestAttribute.DESENSITIZE_AUTHORIZED)`；外层 `FrameworkContextClearFilter` 兜底清理上下文。
6. **委托权限独立**：机器认证、脱敏授权、`TENANT_DELEGATION_AUTHORIZED` 请求标志和可信 JWT 委托声明均不提供资源/行为级跨租户授权；当前通用委托请求默认拒绝。

**安全行为变更与迁移指导**：依赖旧标记头获取明文的调用不再获得明文。确需明文的调用须迁移到具有认证与目标端点授权的内部路径，由宿主在 `+20` 过滤器之前受控设置脱敏请求属性；禁止把 Header 转换成授权属性、对所有 RPC 或公共展示 API 统一赋权。跨租户访问须由具体受保护的业务操作另行验证，不能通过通用委托标志放行。

网关在认证路径、匿名白名单及 `enabled=false` 分支均剥离 `X-Internal-RPC`、`X-Delegated-Tenant-Id`、`X-Grant-Ticket`。此契约不引入共享万能密钥或自制签名，也不替代敏感操作审批、审计及实际部署的入口隔离验证。

---

## 📝 12. 审计注解（`@AuditLog`）

```java
@AuditLog(module = "订单", value = "创建订单", businessType = BusinessType.INSERT, recordResult = true)
@PostMapping("/orders")
public R<OrderVo> create(@RequestBody OrderDto dto) { ... }
```

| 属性 | 默认值 | 说明 |
| :--- | :--- | :--- |
| `module` | `""` | 模块名 |
| `value` | `""` | 功能描述 |
| `businessType` | `BusinessType.OTHER` | `OTHER`、`INSERT`、`UPDATE`、`DELETE`、`QUERY`、`EXPORT`、`GRANT`、`SHRED` |
| `recordParams` | `true` | 记录请求参数（JSON） |
| `recordResult` | `false` | 记录返回结果 |

`AuditLogAspect` 对标注方法执行环绕通知，采集 `traceId`、`FrameworkContext` 快照中的租户与操作人、客户端 IP、请求 URI、方法签名、耗时与成功或失败状态，封装为 `AuditLogEvent` 并通过 `ApplicationEventPublisher` 发布；不记录委托票据原文。本模块不包含事件监听与持久化实现，落库由业务侧监听器完成（待确认：各业务工程的监听实现）。

---

## 🪵 13. 请求日志（`RequestLogAspect`）

- 生效条件: `@Profile({"dev", "test"})`（`AppConstant.DEV_CODE` 与 `AppConstant.UAT_CODE`）。
- 切入点: `@Controller` / `@RestController` 下的方法（返回 `R` 的方法除外）。
- 行为: 记录请求参数，`@PathVariable` 参数跳过；`@RequestBody` 按 JSON 处理。
- 脱敏: 名称含 `token`、`password`、`secret`、`authorization`、`cookie`、`captcha` 等凭据类字段**无条件**打码；其它字段在 `zhkj.desensitize.enabled` 与 `zhkj.desensitize.log.enabled` 均为 `true` 时，经 `DesensitizeUtil.autoDesensitize` 处理，`zhkj.desensitize.log.ignore-fields` 中的字段除外。

---

## 🧰 14. 工具类

| 类 | 说明 |
| :--- | :--- |
| `WebUtil` | `getRequest()`、`getIP()` / `getIP(HttpServletRequest)`（依次检查 `X-Forwarded-For`、`X-Real-IP` 等头）、`getCookie` / `getCookieVal` / `setCookie`（`HttpOnly`，Path `/`）、`isBody(HandlerMethod)` |
| `SpringWebUtil` | `getMethodParameter`、`getAnnotation`（方法优先，再类，支持组合注解）、`isAnnotated` |
| `MultipartFileUtil` | 上传文件的扩展名、空判断与保存辅助 |

---

## 🧪 15. 典型控制器

```java
@RestController
@RequestMapping("/api/v1/orders")
public class OrderController {

    @PostMapping
    public OrderVo createOrder(@Valid @RequestBody OrderDto dto) {
        if (dto.getAmount() == null) {
            throw new ServiceException(ResultCode.PARAM_VALID_ERROR, "订单金额不能为空");
        }
        return orderService.create(dto);
    }
}
```

> `ServiceException` 位于 `io.github.ybb88.framework.exception`（`zhkj-framework-core`）。`ResultCode` 位于 `io.github.ybb88.framework.api`。
