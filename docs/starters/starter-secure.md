# 安全鉴权中心：zhkj-framework-starter-secure

> **模块定位**: 微服务鉴权 Starter，提供 JWT 签发与验签、令牌黑名单、权威会话认证 SPI（`TokenAuthenticationProvider`）、方法级权限注解（`@PreAuth`）、Controller 参数注入（`ZhkjUser`）、Actuator 端点保护、以及跨线程安全上下文快照。  
> **适用基准**: OpenJDK 21, Spring Boot 4.x, Spring Security 7.x（可选）, JJWT 0.12.x  
> **模块坐标**: `io.github.ybb88:zhkj-framework-starter-secure`

---

## 📋 1. 架构定位与核心设计

1. **JWT 验签（JJWT 0.12）**：签发使用 `Jwts.builder()`，验签使用 `Jwts.parser().verifyWith(key)`。签名密钥取自 `zhkj.secure.token.sign-key`。
2. **统一认证流水线（`TokenAuthenticator`）**：黑名单检查 → JWT 验签与时效 → 签发者白名单 → 权威会话 Provider 或纯 JWT 回退。失败统一抛出 `SecureException(ResultCode.UN_AUTHORIZED)`。
3. **请求凭据提取（`TokenCredentialUtil`）**：仅接受请求头，不接受 URL 参数与 Cookie（见第 5.2 节）。
4. **方法级授权（`@PreAuth`）**：基于 SpEL，根元对象为 `AuthFun`，可访问 Spring Bean 与方法参数。
5. **上下文生命周期**：`SecureInterceptor` 每次受保护请求清空并重新认证；`afterCompletion` 清理 `SecurityContextHolder`；跨线程通过 `FrameworkContext` 快照恢复。

---

## 📦 2. 依赖坐标

```xml
<dependency>
    <groupId>io.github.ybb88</groupId>
    <artifactId>zhkj-framework-starter-secure</artifactId>
</dependency>
```

Redis 黑名单需要 `spring-boot-starter-data-redis` 在类路径中；`ActuatorSecurityConfiguration` 需要 `spring-boot-starter-security`。

---

## 🧱 3. 核心包结构与组件

```
io.github.ybb88.framework.secure
├── ZhkjUser.java, AuthInfo.java, TokenInfo.java    # 用户模型与令牌模型
├── aot/SecureRuntimeHints.java                     # AOT 运行时提示（META-INF/spring/aot.factories）
├── annotation/PreAuth.java                         # 方法/类级授权注解
├── aspect/AuthAspect.java                          # @PreAuth 切面
├── auth/
│   ├── AuthFun.java                                # SpEL 根对象函数
│   ├── TokenAuthenticator.java                     # 统一认证流水线
│   └── TokenAuthenticationProvider.java            # 权威会话 SPI
├── blacklist/
│   ├── TokenBlacklistService.java                  # 黑名单 SPI
│   ├── RedisTokenBlacklistService.java             # Redis 实现（需 StringRedisTemplate）
│   └── InMemoryTokenBlacklistService.java          # 进程内实现
├── config/
│   ├── RegistryConfiguration.java                  # SecureRegistry Bean（before SecureConfiguration）
│   ├── SecureConfiguration.java                    # 主装配
│   └── ActuatorSecurityConfiguration.java          # /actuator/** 保护
├── constant/SecureConstant.java
├── context/SecureContextSnapshotProvider.java, SecurityContextHolder.java
├── exception/SecureException.java
├── interceptor/SecureInterceptor.java, ClientInterceptor.java
├── jwt/ToolsJacksonSerializer.java, ToolsJacksonDeserializer.java   # JJWT 序列化桥接
├── props/ZhkjSecureProperties.java, ClientSecure.java
├── provider/
│   ├── IClientDetails.java, ClientDetails.java
│   ├── IClientDetailsService.java, ClientDetailsServiceImpl.java    # 默认 JDBC 查询 client 表
│   └── SecureAuditUserProvider.java                                 # AuditUserProvider 实现
├── registry/SecureRegistry.java                    # 放行路径注册表
├── resolver/TokenArgumentResolver.java             # Controller 参数 ZhkjUser 解析
├── rpc/SecurityRpcHeaderProvider.java              # RPC 身份头透传
└── util/
    ├── SecureUtil.java                             # 工具入口（public final class）
    ├── SecureWebUtil.java                          # 客户端 IP 等 Web 辅助
    └── TokenCredentialUtil.java                    # 请求头凭据解析
```

自动装配声明：`SecureConfiguration`、`RegistryConfiguration`、`ActuatorSecurityConfiguration`（`META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`）。

---

## ⚙️ 4. 核心配置清单 (`application.yml`)

```yaml
zhkj:
  secure:
    enabled: true                       # 默认 true，同时控制 SecureRegistry 与 Servlet 拦截器
    token:
      sign-key: "${JWT_SIGN_KEY}"       # 必填，原始字符串长度 >= 32
      accepted-issuers: [business-provider]   # 默认空列表（不限制签发者）
      allow-pure-jwt: false             # 默认 true；受管会话应设为 false
    blacklist:
      enabled: true                     # 默认 true
      key-prefix: "zhkj:secure:token:blacklist:"   # 默认值
    exclude-patterns:                   # 默认空；加入 SecureInterceptor 放行路径
      - "/doc.html"
      - "/webjars/**"
      - "/v3/api-docs/**"
    skip-url: []                        # 默认空；与 exclude-patterns 合并使用
    client-table: zhkj_client           # 默认 zhkj_client，JDBC 客户端表名
    client:                             # 默认空；配置客户端拦截器
      - client-id: "zhkj-web"
        path-patterns: ["/open-api/**"]
```

### 4.1 配置键说明

| 键 | 默认值 | 说明 |
|----|--------|------|
| `zhkj.secure.enabled` | `true` | `false` 时不装配 `SecureConfiguration`，也不注册拦截器 |
| `zhkj.secure.token.sign-key` | 无 | 启动时校验，长度不足抛出 `IllegalStateException`。解析为 Base64 失败则按 UTF-8 字节使用 |
| `zhkj.secure.token.accepted-issuers` | 空 | 非空时，`iss` 缺失或不在列表中的令牌被 `SecureUtil.parseJWT` 返回 `null` |
| `zhkj.secure.token.allow-pure-jwt` | `true` | 无匹配 Provider 时是否接受已验签的纯 JWT |
| `zhkj.secure.token.access-token-validity` | `7200` | 已绑定；当前签发路径未读取 |
| `zhkj.secure.token.refresh-token-validity` | `604800` | 已绑定；当前签发路径未读取 |
| `zhkj.secure.blacklist.enabled` | `true` | 是否执行黑名单检查 |
| `zhkj.secure.blacklist.key-prefix` | `zhkj:secure:token:blacklist:` | Redis 键前缀，实际键为前缀 + SHA-256(token) |
| `zhkj.secure.exclude-patterns` / `skip-url` | 空 | 追加到拦截器排除路径 |
| `zhkj.secure.client[]` | 空 | `client-id`、`path-patterns`，注册 `ClientInterceptor` |
| `zhkj.secure.client-table` | `zhkj_client` | 客户端查询表，列：`client_id`、`client_secret`、`access_token_validity`、`refresh_token_validity` |

### 4.2 默认放行路径（代码现状）

`SecureRegistry` 构造时固定加入以下默认排除模式，`SecureInterceptor` 对其不做认证：

`/actuator/health/**`、`/actuator/info`、`/ping`、`/v3/api-docs/**`、`/doc.html`、`/webjars/**`、`/swagger-ui/**`、`/swagger-resources/**`、`/v2/api-docs/**`、`/auth/**`、`/token/**`、`/log/**`、`/user/user-info`、`/user/user-info-by-id`、`/menu/auth-routes`、`/error/**`、`/assets/**`

这些模式仅在认证拦截器注册（`SecureRegistry.enabled=true`，默认与 `zhkj.secure.enabled` 一致）时生效。`/auth/**`、`/token/**`、`/log/**` 等通配模式覆盖范围较大，业务服务中同名路径将不经过令牌校验（代码未提供开关关闭这部分默认项，待确认是否需要调整）。

---

## 🔐 5. 认证流程与凭据约定

### 5.1 请求认证（`SecureInterceptor.preHandle`）

1. 清空当前线程安全上下文（不信任缓存用户）。
2. `SecureUtil.getToken(request)` 提取令牌。
3. `TokenAuthenticator.authenticate(token)`：
   - 令牌为空 → 拒绝；
   - 黑名单启用且命中 → 拒绝；
   - `SecureUtil.parseJWT` 返回 `null`（签名错误、过期、时效不符、签发者不匹配）→ 拒绝；
   - 遍历所有 `TokenAuthenticationProvider`，`supports(claims)` 命中多个 → 拒绝；命中一个则调用 `authenticate(token, claims)`；
   - 无命中：`allow-pure-jwt=true` 时由 `SecureUtil.userFromClaims(claims)` 映射用户，否则拒绝；
   - 返回 `null` → 拒绝。
4. 成功：写入 `SecurityContextHolder`，若用户携带租户 ID 则桥接到 `FrameworkContext`。
5. 失败：HTTP 状态 `401`，响应体为 `R.fail(401, ...)`。

### 5.2 请求凭据（`TokenCredentialUtil.resolve`）

| 请求头 | 格式 |
|--------|------|
| `Authorization` | 必须为 `Bearer <token>` |
| `zhkj-auth` | 裸令牌或 `Bearer <token>` |
| `Blade-Auth` | 裸令牌或 `Bearer <token>` |

- 同一请求头出现多个值 → 拒绝；逗号合并值 → 拒绝。
- 不同请求头携带相同令牌可以共存；携带不同令牌 → 拒绝。
- 令牌含空白、逗号或非 ASCII 可见字符 → 拒绝。
- URL 参数（包括 `token`、`Authorization`）与 Cookie 不参与认证。

### 5.3 权威会话 Provider

```java
public interface TokenAuthenticationProvider {
    boolean supports(Claims claims);
    ZhkjUser authenticate(String token, Claims claims);   // 返回 null 或抛异常即拒绝
}
```

- 以 Spring Bean 形式注册，`TokenAuthenticator` 延迟解析（`ObjectProvider.orderedStream()`），避免启动期循环依赖。
- 多个 Provider 同时 `supports` 为拒绝；返回的 `ZhkjUser` 为权威身份，不以 JWT 中旧权限覆盖。

### 5.4 黑名单

- 类路径存在 `StringRedisTemplate` 且容器中存在该 Bean 时使用 `RedisTokenBlacklistService`，否则使用 `InMemoryTokenBlacklistService`。
- Redis 读取异常 → 视为黑名单命中（拒绝访问）。
- 写入时 `expireSeconds <= 0` 按 86400 秒处理。

---

## 🧰 6. 工具与 API

### 6.1 `SecureUtil`（`public final class`，私有构造）

| 方法 | 说明 |
|------|------|
| `getUser()` / `setUser(ZhkjUser)` / `clearUser()` | 读写当前线程安全上下文，不重新认证 |
| `getUser(HttpServletRequest)` | 通过统一认证流水线解析请求，**会重新执行黑名单与 Provider 校验** |
| `userFromClaims(Claims)` | 纯 JWT 字段映射（`allow-pure-jwt` 回退路径使用） |
| `getUserId()` / `getUserAccount()` / `getUserName()` / `getUserRole()` / `getTenantId()` / `getClientId()` | 从当前上下文读取；无用户时返回 `-1` 或空串 |
| `isAdministrator()` | 当前角色编码（逗号分隔）包含 `administrator` 或 `super_admin`（`RoleUtil.isPrivileged`，忽略大小写）。数据权限放行由 `AuditUserProvider.isAdmin()` 判定（`SecureAuditUserProvider` 基于同一判定实现），不直接调用本方法 |
| `getToken(HttpServletRequest)` | 按第 5.2 节提取令牌 |
| `getHeader()` / `getHeader(HttpServletRequest)` | 返回 `Bearer <token>` |
| `parseJWT(String)` | 验签与时效校验；不代表会话仍有效 |
| `createJWT(Map<String,Object> claims, String audience, String issuer, String subject, long expiresInSeconds)` | 服务端签发。保留声明（`sub`、`iss`、`aud`、`jti`、`iat`、`nbf`、`exp`）由参数与框架生成，调用方 claims 不能覆盖 |
| `getClaims(HttpServletRequest)` | 解析请求令牌为 `Claims` |
| `init(ZhkjSecureProperties)` | 由 `SecureConfiguration` 启动期调用 |

`createJWT(Map, String, String, String)` 四参重载仍存在：它从当前请求的 `Zhkj-Authorization` 或 `Authorization: Basic` 读取客户端凭据并查询 `IClientDetails` 决定有效期。新代码应使用五参重载。

### 6.2 `SecurityContextHolder`

`setUser`、`getUser`、`getUserId`（无用户返回 `-1`）、`getTenantId`、`hasUser`、`clear`、`runWithUser`、`callWithUser`、`callWithUserChecked`。线程本地存储，不使用 `InheritableThreadLocal`。

### 6.3 `ZhkjUser`

字段：`clientId`、`userId`（Long）、`tenantId`、`dimensionId`、`deptId`（String）、`userName`、`account`、`roleId`、`roleName`、`userType`、`userCode`、`deptCode`、`permissions`（List&lt;String&gt;）、`extMapList`。

### 6.4 `AuthFun`（SpEL 根对象）

| 函数 | 语义 |
|------|------|
| `permitAll()` / `denyAll()` | 恒真 / 恒假 |
| `hasRole(String)` / `hasAnyRole(String...)` | `roleName`（逗号分隔）精确匹配 |
| `hasPermission(String)` / `hasAnyPermission(String...)` | `permissions` 精确匹配；`*` 不作为通配符 |

`admin` 账号名或角色不隐式获得资源权限。

---

## 💡 7. 典型使用范式

### 7.1 方法级授权

```java
@RestController
@RequestMapping("/api/v1/admin")
public class AdminController {

    @DeleteMapping("/users/{id}")
    @PreAuth("hasRole('admin') or hasPermission('sys:user:delete')")
    public R<Void> deleteUser(@PathVariable Long id) {
        return R.success("删除成功");
    }
}
```

`@PreAuth` 可标注在类上（类级表达式对所有方法生效），方法级注解优先。表达式不满足时抛出 `SecureException(UN_AUTHORIZED)`，HTTP 状态为 `401`。

### 7.2 获取当前用户

```java
ZhkjUser user = SecureUtil.getUser();
if (user != null) {
    Long userId = user.getUserId();
    String tenantId = user.getTenantId();
}
```

Controller 参数直接声明 `ZhkjUser user`，由 `TokenArgumentResolver` 注入当前上下文用户。

### 7.3 服务端签发令牌（受管会话）

```java
Map<String, Object> claims = Map.of("permissions", List.of("order:export"), "roleName", "ops");
String token = SecureUtil.createJWT(claims, "order-service", "business-provider", "user-1001", 7200);
```

签发方须在调用前完成凭据校验与授权。签发者须出现在 `accepted-issuers` 中，否则其他服务的 `parseJWT` 会拒绝该令牌。

### 7.4 自定义权威会话 Provider

```java
@Component
public class SessionAuthProvider implements TokenAuthenticationProvider {

    @Override
    public boolean supports(Claims claims) {
        return "business-provider".equals(claims.getIssuer());
    }

    @Override
    public ZhkjUser authenticate(String token, Claims claims) {
        // 查询会话存储与当前授权，返回权威用户；会话失效返回 null
        return sessionService.loadUser(claims.getId());
    }
}
```

### 7.5 白名单

```java
@Bean
public SecureRegistry secureRegistry() {
    SecureRegistry registry = new SecureRegistry();
    registry.setEnabled(true);   // 默认 false：未设置时认证拦截器不会注册
    registry.excludePathPatterns("/open-api/**", "/notify/pay-callback");
    return registry;
}
```

自定义 `SecureRegistry` Bean 时，`RegistryConfiguration` 不再创建默认 Bean，`SecureRegistry.enabled` 字段默认为 `false`，必须由应用显式 `setEnabled(true)`（或使用默认 Bean），否则 `SecureConfiguration` 不会注册认证拦截器。默认放行项仍由 `SecureRegistry` 构造函数加入。

### 7.6 跨线程传递

```java
ContextSnapshot snapshot = FrameworkContext.capture();
executor.execute(() -> FrameworkContext.runWith(snapshot, this::doWork));
```

`SecureContextSnapshotProvider` 随快照恢复角色、权限、部门与用户元数据；作用域退出后恢复原状态，无原用户时清空。

### 7.7 RPC 身份透传

`SecurityRpcHeaderProvider` 在 `SecurityContextHolder` 有用户时注入：`X-User-Id` / `zhkj-user-id`、`X-User-Account` / `zhkj-user-account`、`X-User-Name` / `zhkj-user-name`（URL 编码）、`X-Tenant-Id` / `X-Tenant-ID` / `zhkj-tenant-id`。

---

## 🔌 8. Actuator 保护

`ActuatorSecurityConfiguration` 在类路径存在 `SecurityFilterChain` 时生效：`/actuator/**` 中仅 `/actuator/health`、`/actuator/info` 允许匿名，其余要求认证（HTTP Basic，CSRF 关闭）。若应用已定义同名 `actuatorSecurityFilterChain` Bean，则不再创建。

---

## 🛡️ 9. 常见问题

1. **启动报错 `zhkj.secure.token.sign-key must be configured and at least 32 characters`**：签名密钥未配置或长度不足。
2. **401 但令牌看似有效**：检查是否为 `Authorization` 缺少 `Bearer ` 前缀；是否存在 `zhkj-auth` 与 `Authorization` 指向不同令牌；`accepted-issuers` 是否包含 `iss`；`allow-pure-jwt=false` 时是否有 Provider 匹配。
3. **`SecureUtil.getUser()` 返回 null**：仅在经过 `SecureInterceptor` 的请求中有值；RPC、异步线程需通过快照恢复。

---

## 📎 10. 相关模块

- `zhkj-framework-starter-web`：全局异常处理与请求日志凭据脱敏
- `zhkj-framework-starter-sso-client`：基于 `TokenAuthenticator` 的单点登录过滤器与网关过滤器
- `zhkj-framework-starter-mybatis`：审计字段经 `AuditUserProvider` SPI 获取操作人，本模块提供实现 `SecureAuditUserProvider`
