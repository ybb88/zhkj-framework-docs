# ZHKJ Framework 版本演进与升级迁移指南

> **目标版本**: 2026.4.1.0-SNAPSHOT  
> **更新日期**: 2026-09-08 08:30  
> **核心升级**: OpenJDK 21 LTS, Spring Boot 4.1.1, Spring Cloud 2025.1.2, Spring Batch 6.0.5, Spring Security 7.x, JJWT 0.12.5

---

## 📋 1. 版本重大变更一览

从早期版本（2.x / Spring Boot 2.x/3.x）升级至当前 `2026.4.1.0-SNAPSHOT`，框架在底层运行时、架构分层以及安全性方面进行了系统化重构：

| 变更维度 | 旧版本行为 | 2026.4.1.0-SNAPSHOT 现代基准 | 影响级别 |
|:---|:---|:---|:---|
| **语言运行时** | JDK 8 / 11 / 17 | **OpenJDK 21 LTS 强制要求**（支持虚拟线程与现代语法） | 🔴 阻塞级 |
| **Spring 基线** | Spring Boot 2.7.x / 3.2.x | **Spring Boot 4.1.1 + Spring Framework 7.x** | 🔴 阻塞级 |
| **RPC 远程通信** | Spring Cloud OpenFeign（已停滞维护） | **Spring 6+ 原生 HTTP Interfaces (`@HttpExchange`) via `starter-http`** | 🔴 阻塞级 (Feign 彻底下线) |
| **Spring Security** | `WebSecurityConfigurerAdapter` | **`SecurityFilterChain` + Lambda DSL** | 🔴 阻塞级 |
| **JWT 安全套件** | JJWT 0.9.x / `parserBuilder()` | **JJWT 0.12.5 + `Jwts.parser().verifyWith()`** | 🟡 代码适配 |
| **模块拓扑** | 单体 `common` / `launch` 杂糅 | **17 个单向依赖框架组件 + 示例工程** | 🟡 坐标调整 |
| **单点登录 (SSO)** | 各业务端自行管理 Session / Token | **统一认证中心 + 通用客户端 `starter-sso-client`** | 🟡 架构升级 |
| **多租户与数据权限** | 深度耦合在 MyBatis 内部 | **独立为 `starter-tenant` 与 `starter-datascope` SPI** | 🟡 依赖解耦 |
| **配置安全解密** | Jasypt 第三方依赖 | **`starter-crypto` 原生 `EnvironmentPostProcessor` (`ENC(...)`)** | 🟡 架构收敛 |
| **在线接口文档** | Swagger 2 (`@Api`, `@ApiOperation`) | **Knife4j 4.5.0 + OpenAPI 3.0 (`@Tag`, `@Operation`)** | 🟢 平滑兼容 |

---

## 🧱 2. 模块重组与 Maven 坐标变迁

### 2.1 模块映射对照
```
旧版本模块名                           新版本模块名                          说明
─────────────────────────────────────────────────────────────────────────────
zhkj-framework-common          →  zhkj-framework-core              纯纯核心工具库与响应封装
zhkj-framework-launch          →  zhkj-framework-launcher          启动器核心驱动类
                                  zhkj-framework-starter-web       Web MVC 核心运行时增强底座
zhkj-framework-starter-datascope  zhkj-framework-starter-datascope 框架无关数据权限抽象与注解
(原与 MyBatis 混杂)            →  zhkj-framework-starter-mybatis   MyBatis 数据权限拦截器实现
                                  zhkj-framework-starter-tenant    独立解耦的租户核心
(新增数据访问抽象)              →  zhkj-framework-data-api          ORM 无关数据访问与 SPI
(新增字段加解密与配置解密)      →  zhkj-framework-starter-crypto    字段级脱敏、密文存储与配置解密
spring-cloud-starter-openfeign →  zhkj-framework-starter-http      Spring 6+ 原生 HTTP Interfaces 客户端
(新增单点登录接入客户端)        →  zhkj-framework-starter-sso-client 通用 SSO 客户端防护与上下文注入
```

### 2.2 包名调整
```java
// 旧包名引用
import io.github.ybb88.framework.common.utils.*;
import io.github.ybb88.framework.launch.*;

// 新规范包名引用
import io.github.ybb88.framework.utils.*;
import io.github.ybb88.framework.launcher.annotation.ZhkjApplication;
import io.github.ybb88.framework.launcher.launch.ZhkjApplication;
```

---

## 🔐 3. 安全体系迁移

### 3.1 Spring Security 7.x `SecurityFilterChain` 迁移
Spring Security 已彻底废弃 `WebSecurityConfigurerAdapter`。请将旧版配置迁移为声明式 `SecurityFilterChain` Bean：

```java
// ❌ 旧版本（已移除）
@Configuration
public class SecurityConfiguration extends WebSecurityConfigurerAdapter {
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.csrf().disable()
            .authorizeRequests()
            .antMatchers("/actuator/**").permitAll()
            .anyRequest().authenticated();
    }
}

// ✅ 新版本（Spring Security 7.x 规范）
@Configuration
public class SecurityConfiguration {
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .csrf(AbstractHttpConfigurer::disable)
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/actuator/**", "/doc.html", "/v3/api-docs/**").permitAll()
                .anyRequest().permitAll() // 或按业务要求配置
            );
        return http.build();
    }
}
```

### 3.2 JJWT 0.12.5 Token 解析 API 变迁
旧版的 `Jwts.parserBuilder().setSigningKey(key).build()` 在 JJWT 0.12.x 中已标记废弃，框架底层已全面统一为：

```java
// ✅ JJWT 0.12.5 标准写法
SecretKey key = Keys.hmacShaKeyFor(secret.getBytes(StandardCharsets.UTF_8));
Claims claims = Jwts.parser()
    .verifyWith(key)
    .build()
    .parseSignedClaims(token)
    .getPayload();
```

---

## ⚡ 4. Spring Boot 4.x 缓存适配变化

Spring Boot 4 中调整了缓存自动配置类的包路径：
- `org.springframework.boot.autoconfigure.cache.CacheProperties` 迁移至 `org.springframework.boot.cache.autoconfigure.CacheProperties`
- `org.springframework.boot.autoconfigure.cache.CacheManagerCustomizers` 迁移至 `org.springframework.boot.cache.autoconfigure.CacheManagerCustomizers`

在 `zhkj-framework-starter-redis` 中已完成此升级。如果您自定义了 Redis 缓存配置，请同步更新导入的包路径。

---

## 🏢 5. 多租户与数据权限解耦迁移

### 5.1 数据权限使用迁移
若仅需使用数据权限注解，直接引入 `zhkj-framework-starter-datascope`。底层持久层不管是 MyBatis 还是 JPA，均能无缝桥接：

```xml
<dependencies>
    <!-- 数据权限注解与 SPI -->
    <dependency>
        <groupId>io.github.ybb88</groupId>
        <artifactId>zhkj-framework-starter-datascope</artifactId>
    </dependency>
    <!-- MyBatis 实现（内置 AST 拦截器） -->
    <dependency>
        <groupId>io.github.ybb88</groupId>
        <artifactId>zhkj-framework-starter-mybatis</artifactId>
    </dependency>
</dependencies>
```

在方法级别标注：
```java
@DataAuth(type = DataScopeEnum.DEPT, column = "dept_id")
public List<Order> queryOrders() {
    return orderMapper.selectList(null);
}
```

### 5.2 多租户使用迁移
配置项统一规范为 `zhkj.tenant.*`：
```yaml
zhkj:
  tenant:
    enabled: true
    column: tenant_id
    ignore-tables:
      - sys_tenant
      - sys_dict
```

---

## 🌐 6. OpenFeign 迁移至 Spring 6+ HTTP Interfaces

Spring Cloud OpenFeign 已进入停滞维护阶段。框架全面基于 Spring 6+ 原生推出的 **HTTP Interfaces** 建设微服务 RPC 通信能力：

### 6.1 依赖替换
移除 `spring-cloud-starter-openfeign`，在 API 契约包或消费方工程引入：
```xml
<dependency>
    <groupId>io.github.ybb88</groupId>
    <artifactId>zhkj-framework-starter-http</artifactId>
</dependency>
```

### 6.2 注解与代码迁移
- 将启动类上的 `@EnableFeignClients` 替换为 `@EnableHttpClients(basePackages = "...")`；
- 将客户端接口上的 `@FeignClient` 替换为 Spring 原生 `@HttpExchange`：

```java
// ❌ 旧版本 Feign 写法
@FeignClient(name = "user-service", fallbackFactory = UserFallbackFactory.class)
public interface UserClient {
    @GetMapping("/users/{id}")
    R<UserVO> getUserById(@PathVariable("id") Long id);
}

// ✅ 新版本 Spring 6+ HTTP Interface 标准写法
@HttpExchange(url = "http://user-service/users")
public interface UserClient {
    @GetExchange("/{id}")
    R<UserVO> getUserById(@PathVariable("id") Long id);
}
```

框架内置的 `HttpClientHeaderInterceptor` 会自动透传当前请求的认证 Token、租户 ID 及链路追踪 ID，并支持动态优雅降级。

---

## 🔑 7. 统一单点登录与全生命周期数据安全迁移

### 7.1 下游微服务接入统一 SSO 客户端
无需在每个微服务单独编写登录认证逻辑，直接引入 `zhkj-framework-starter-sso-client`：
1. 启动类添加 `@EnableSsoClient`；
2. 在 `application.yml` 中配置认证中心与客户端凭证：
```yaml
zhkj:
  sso:
    client:
      enabled: true
      auth-server-url: "http://124.221.100.243:8050"
      client-id: "your-service-name"
      client-secret: "your-secret-key"
```
3. 在业务方法中直接通过 `SsoContextHolder.getContext()` 提取自然人主账号与所属租户业务身份。

### 7.2 敏感配置与持久层字段加密迁移
- **配置解密**: 移除历史引入的 Jasypt 依赖。使用 `zhkj-framework-starter-crypto` 统一提供的 `ENC(...)` 透明解密。
- **字段存储加密**: 实体敏感字段配置 `@TableField(typeHandler = EncryptTypeHandler.class)`（SM4-CBC/AES），配合盲索引辅助列存储 `BlindIndexUtil.hash(...)`（HMAC-SM3）进行精准与模糊匹配。

---

## 📝 8. 升级验证清单

升级完成后，请依次执行以下检查：
1. **JDK 版本校验**: 运行 `java -version` 确保输出为 `21.0.x`。
2. **构建测试**: 执行 `.\build.ps1` 验证所有 27 个模块是否能 `100% BUILD SUCCESS`。
3. **接口文档访问**: 启动应用后访问 `http://localhost:8080/doc.html` 验证 Knife4j 4.5.0 文档页面是否正常展现。
4. **Token 与上下文提取**: 验证携带 `Bearer <token>` 请求受保护接口时能否通过 `SecureUtil.getUser()` 或 `SsoContextHolder.getContext()` 正常提取身份。
5. **配置解密校验**: 验证包含 `ENC(...)` 的数据库密码或中间件连接串在应用启动时能否自动解密成功建联。

