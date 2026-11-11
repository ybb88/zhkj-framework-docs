# 通用单点登录客户端：zhkj-framework-starter-sso-client

> **模块定位**: SSO 客户端 Starter。提供 Servlet 过滤器（未登录重定向或 401、授权码换取令牌）、响应式网关过滤器（请求头校验与身份头转发）、认证中心 HTTP 客户端 `SsoAuthClient`，并把解析结果以 `SsoUserContext` 写入安全上下文。  
> **适用基准**: OpenJDK 21, Spring Boot 4.x, Spring Cloud Gateway（可选）, OAuth 2.x 授权码模式  
> **模块坐标**: `io.github.ybb88:zhkj-framework-starter-sso-client`

---

## 📋 1. 架构定位与核心设计

1. **Servlet 过滤器（`SsoClientFilter`）**：对 `/*` 生效，顺序为 `Ordered.HIGHEST_PRECEDENCE + 50`。
   - 白名单路径直接放行；
   - 请求含 `code` 参数时，调用认证中心换取令牌；
   - 否则从请求头读取访问令牌，经 `TokenAuthenticator` 认证；失败时按 AJAX 规则返回 401 JSON 或重定向到认证中心。
2. **网关过滤器（`SsoGatewayGlobalFilter`）**：仅在类路径存在 `GlobalFilter` 时装配，顺序同上。校验请求头令牌后，向下游注入标准身份头并替换为单一 `Authorization`。
3. **统一认证**：Servlet 与网关共用 `TokenCredentialUtil` 与 `TokenAuthenticator`（来自 `starter-secure`）。令牌只接受请求头；URL 参数与 Cookie 不参与认证。
4. **身份模型**: `SsoTokenParser` 将 JWT 声明映射为 `SsoUserContext`（继承 `ZhkjUser`）；`SsoClientFilter` 认证成功后将其写入 `SsoContextHolder` 与 `SecurityContextHolder`，并经 `FrameworkContext` 同步租户、维度与委托。
5. **会话与权威校验**：认证调用 `TokenAuthenticator`，黑名单与 `TokenAuthenticationProvider` 生效；受管会话服务应设置 `zhkj.secure.token.allow-pure-jwt: false`。

---

## 📦 2. 依赖坐标

```xml
<dependency>
    <groupId>io.github.ybb88</groupId>
    <artifactId>zhkj-framework-starter-sso-client</artifactId>
</dependency>
```

本模块依赖 `zhkj-framework-starter-secure`（提供 `TokenAuthenticator` Bean，Servlet 过滤器注册时必需）与 `zhkj-framework-starter-http`（提供 `RestClient.Builder` 时使用）。网关过滤器需要 `spring-cloud-starter-gateway-server-webflux`。

---

## 🧱 3. 核心包结构与组件

```
io.github.ybb88.framework.sso.client
├── api/
│   ├── SsoAuthClient.java                # @HttpExchange 认证中心客户端
│   ├── dto/TokenRequestDTO.java          # 换令牌请求
│   └── vo/TokenResponseVO.java           # 令牌与身份摘要
├── config/ZhkjSsoClientAutoConfiguration.java
├── context/
│   ├── SsoContextHolder.java             # ThreadLocal<SsoUserContext>
│   ├── SsoContextSnapshotProvider.java   # 快照属性 SSO_USER_CONTEXT
│   └── SsoUserContext.java               # 继承 ZhkjUser
├── filter/
│   ├── SsoClientFilter.java              # Servlet 过滤器
│   └── SsoGatewayGlobalFilter.java       # Spring Cloud Gateway 全局过滤器
├── parser/SsoTokenParser.java            # JWT 声明 → SsoUserContext
└── props/SsoClientProperties.java        # zhkj.sso.client.*
```

自动装配（`META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`）：`ZhkjSsoClientAutoConfiguration`。

| Bean | 条件 |
|------|------|
| `SsoAuthClient` | `@ConditionalOnMissingBean` |
| `SsoContextSnapshotProvider` | `@ConditionalOnMissingBean` |
| `FilterRegistrationBean<SsoClientFilter>` | Servlet Web 应用 |
| `SsoGatewayGlobalFilter` | 类路径存在 `org.springframework.cloud.gateway.filter.GlobalFilter` |

`SsoContextSnapshotProvider.capture` 除保存 `SSO_USER_CONTEXT`、租户与用户信息外，还将当前 `SsoUserContext` 的 `dimensionId`、`delegatedTenantId`、`grantTicket` 写入统一上下文快照，使已建立的维度与委托上下文可通过 `FrameworkContext` 读取。快照携带的是上下文数据，不包含 `RpcRequestAttribute` 服务端请求授权属性，不能据此授予新的脱敏或跨租户权限。

---

## ⚙️ 4. 核心配置清单 (`application.yml`)

```yaml
zhkj:
  sso:
    client:
      enabled: true                                 # 默认 true
      auth-server-url: "http://127.0.0.1:8088"      # 默认值，别名 server-url
      client-id: "demo-business-service"
      client-secret: "${SSO_CLIENT_SECRET}"
      redirect-uri: "https://app.example.com/callback"   # 可选，默认当前请求 URL
      exclude-paths:                                # 默认值见下
        - "/actuator/**"
        - "/error"
        - "/favicon.ico"
        - "/doc.html"
        - "/webjars/**"
        - "/v3/api-docs/**"
        - "/swagger-ui/**"
        - "/swagger-resources/**"
        - "/oauth2/callback"
        - "/static/**"
        - "/public/**"
  secure:
    token:
      sign-key: "${JWT_SIGN_KEY}"
      accepted-issuers: [business-provider]
      allow-pure-jwt: false
```

| 键 | 默认值 | 说明 |
|----|--------|------|
| `zhkj.sso.client.enabled` | `true` | `false` 时不装配本模块 Bean |
| `zhkj.sso.client.auth-server-url` | `http://127.0.0.1:8088` | 认证中心基础地址；`server-url` 为同一属性的别名 |
| `zhkj.sso.client.client-id` / `client-secret` | 无 | 换令牌与授权跳转使用 |
| `zhkj.sso.client.redirect-uri` | 无 | 未配置时，授权码请求与跳转使用当前请求 URL（去除 `code`、`state` 后）或当前完整 URL |
| `zhkj.sso.client.exclude-paths` | 见示例 | Ant 风格，匹配前对路径做 `StringUtils.cleanPath` |
| `zhkj.sso.client.login-page-url` | 无 | 已绑定，当前实现不读取 |

`cookie-name`、`token-header` 不再存在。

---

## 🔐 5. 请求处理流程

### 5.1 Servlet 过滤器

1. URI 命中 `exclude-paths` → 放行。
2. 请求参数存在 `code`：
   - 调用 `SsoAuthClient.exchangeToken`，`grantType=authorization_code`，`clientId`、`clientSecret`、`redirectUri` 随请求发送；
   - 成功：响应头 `Authorization: Bearer <accessToken>`、`Cache-Control: no-store`，正文为 `R<TokenResponseVO>` JSON；
   - 失败：记录 ERROR 日志，302 重定向到去除 `code`、`state` 的当前 URL。
3. 否则通过 `SecureUtil.getToken(request)` 取令牌（仅请求头），调用 `TokenAuthenticator.authenticate(token, claims -> SsoTokenParser.buildUserContext(claims, token))`。
4. 认证失败：清理 `FrameworkContext`。若为 AJAX（`X-Requested-With: XMLHttpRequest`、`Accept` 含 `application/json`，或路径以 `/api/` 开头）返回 401 JSON；否则 302 到 `{auth-server-url}/oauth2/authorize?response_type=code&client_id=...&redirect_uri=...`。
5. 认证成功：读取 `X-Dimension-Id`；仅当当前请求的 `RpcRequestAttribute.TENANT_DELEGATION_AUTHORIZED` 属性为 `Boolean.TRUE`，且同时存在 `X-Delegated-Tenant-Id` 与 `X-Grant-Ticket` 时，才接受委托头；写入 `SsoContextHolder` 与 `SecurityContextHolder`；经 `FrameworkContext` 同步租户、维度与委托；执行后续过滤器；`finally` 中调用 `FrameworkContext.clearAll()`，清理 `SsoContextHolder`、`SecurityContextHolder`、`TenantContext` 及脱敏与数据权限上下文。该 `finally` 仅包围 `chain.doFilter`；身份写入发生在其之前，异常路径由外层 `FrameworkContextClearFilter`（`starter-web`）兜底清理。

**委托迁移**：`X-Internal-RPC: true` 不再授予委托权限。宿主须先完成调用方认证、目标端点及租户委托权限校验，再受控设置上述请求属性，并在包围下游处理的 `finally` 中删除；不得从 Header 或 ThreadLocal 推断授权。机器凭据验证与 `DESENSITIZE_AUTHORIZED` 均不能替代委托授权，可信 JWT 声明建立的现有上下文不受本次变更影响。普通 RPC 与公共展示端点仍默认脱敏。

### 5.2 网关过滤器

1. 入口先移除 `X-Internal-RPC`、`X-Delegated-Tenant-Id`、`X-Grant-Ticket`；即使 `enabled=false` 或路径命中匿名白名单，也仅放行清理后的请求，不透传这三个内部头。
2. `TokenCredentialUtil.resolve` 解析请求头，异常 → 401 JSON。
3. 在 `boundedElastic` 线程执行 `TokenAuthenticator.authenticate`，异常 → 401 JSON。
4. 下游请求头：
   - 设置：`X-User-Id`/`zhkj-user-id`、`X-User-Account`/`zhkj-user-account`、`X-User-Name`/`zhkj-user-name`（URL 编码）、`X-Tenant-Id`/`tenant-id`/`zhkj-tenant-id`、`X-Client-Id`、`X-Dept-Id`、`X-Dimension-Id`；
   - 删除：上述同名头、`X-Delegated-Tenant-Id`、`X-Grant-Ticket`、`X-Internal-RPC`、`zhkj-auth`、`Blade-Auth`；
   - `Authorization` 统一为 `Bearer <token>`。

### 5.3 令牌声明映射（`SsoTokenParser.buildUserContext`）

| 字段 | 依次尝试的声明名 |
|------|------------------|
| `accountId` | `accountId`（解析为 Long） |
| `accountName` | `account`、`accountName` |
| `targetUserId` | `userId`、`targetUserId`、`user_id` |
| `targetUserName` | `userName`、`targetUserName`、`user_name` |
| `tenantId` | `tenantId`、`tenant_id` |
| `dimensionId` | `dimensionId`、`deptId`、`dept_id` |
| `clientId` | `clientId`、`client_id` |
| `roles` | `roles`（数组）；缺失时取 `role_name` / `roleName` 按逗号拆分 |
| `permissions` | `permissions`（数组），忽略 `*` |

字符串声明为空或 `"null"` 时视为缺失。

---

## 🧩 6. `SsoAuthClient`

```java
@HttpExchange
public interface SsoAuthClient {
    @PostExchange("/oauth2/token")
    R<TokenResponseVO> exchangeToken(@RequestBody TokenRequestDTO request);

    @PostExchange("/oauth2/logout")
    R<Boolean> logout(@RequestHeader(value = "Authorization", required = false) String token);
}
```

- `TokenRequestDTO`：`grantType`、`code`、`redirectUri`、`clientId`、`clientSecret`、`refreshToken`。
- `TokenResponseVO`：`accessToken`、`tokenType`、`expiresIn`、`refreshToken`、`accountId`、`accountName`、`targetUserId`、`targetUserName`、`tenantId`。
- 过滤器自动调用 `exchangeToken`；`logout` 不由过滤器调用，需业务自行调用。

---

## 💡 7. 典型使用范式

### 7.1 启动类

```java
@SpringBootApplication
public class OrderBusinessApplication {
    public static void main(String[] args) {
        ZhkjApplication.run("order-business-service", OrderBusinessApplication.class, args);
    }
}
```

无需额外 `@Enable*` 注解。

### 7.2 读取当前用户

```java
@GetMapping("/my-list")
public R<List<OrderVo>> getMyOrders() {
    SsoUserContext user = SsoContextHolder.get();
    if (user == null) {
        return R.fail(ResultCode.UN_AUTHORIZED);
    }
    Long accountId = user.getAccountId();
    String tenantId = user.getTenantId();
    String account = user.getAccount();
    return R.data(orderService.listByAccount(account, tenantId));
}
```

Controller 参数声明 `ZhkjUser`（或 `SsoUserContext`）同样可注入当前用户。

### 7.3 授权码回调

应用自有回调端点若在 `exclude-paths` 中，过滤器不处理其 `code` 参数。默认白名单包含 `/oauth2/callback`；若回调路径不同，须自行加入白名单并实现换令牌逻辑，或从 `exclude-paths` 中移除以让过滤器接管。

---

## 🛡️ 8. 注意事项

- 本模块不写 Cookie。换取成功后，浏览器需在后续请求中携带 `Authorization` 头。
- IdP 的 SSO 会话 Cookie 不被转换为业务访问令牌。
- 业务系统若接收 IdP 签发的自然人令牌，应通过 `accepted-issuers` 与权威 Provider 限定，避免自然人令牌直接成为业务授权。
- `SsoUserContext.getRoleName()` 在 `roleName` 为空时由 `roles` 拼接生成。
- 网关过滤器对 Redis 或 Provider 的阻塞调用放在 `boundedElastic` 上执行。

---

## 🚨 9. 已知问题（代码现状）

- `SsoTokenParser.buildUserContext` 将 `deptId` / `dept_id` 声明映射到 `dimensionId`（`SsoTokenParser.java` 第 47 行），未为 `deptId` 字段赋值。网关 `X-Dept-Id` 取自认证结果的 `deptId`（`SsoGatewayGlobalFilter.java` 第 124、169–170 行），因此仅当 Provider 返回的 `ZhkjUser` 携带 `deptId` 时才会转发，纯 JWT 声明映射路径下为空。

---

## 📎 10. 相关文档

- `starter-secure`：`TokenAuthenticator`、`TokenAuthenticationProvider`、请求凭据规则
- `starter-tenant`：租户上下文与委托头语义
