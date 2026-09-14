# ZHKJ Framework 版本演进与升级迁移指南

> **目标版本**: 2026.4.1.0-SNAPSHOT  
> **核心基准**: OpenJDK 21 LTS, Spring Boot 4.1.1, Spring Cloud 2025.1.2, Spring Batch 6.0.5, Spring Security 7.x, JJWT 0.12.5  
> **设计与创作者**: Jason.Yang（274107098@qq.com）  

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
| **模块拓扑** | 单体 `common` / `launch` 杂糅 | **18 个单向依赖框架组件 + 示例工程** | 🟡 坐标调整 |
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
zhkj-framework-common          →  zhkj-framework-core              纯纯核心工具库与统一上下文门面
zhkj-framework-launch          →  zhkj-framework-launcher          启动器核心驱动类
                                  zhkj-framework-starter-web       Web MVC 核心运行时增强底座
zhkj-framework-starter-datascope  zhkj-framework-starter-datascope 框架无关数据权限抽象与注解
(原与 MyBatis 混杂)            →  zhkj-framework-starter-mybatis   MyBatis-Plus AST 数据权限与租户拦截
                                  zhkj-framework-starter-tenant    独立解耦的租户核心
(新增数据访问抽象)              →  zhkj-framework-data-api          ORM 无关数据访问与 SPI
(新增字段加解密与配置解密)      →  zhkj-framework-starter-crypto    字段级脱敏、国密密文存储与配置解密
spring-cloud-starter-openfeign →  zhkj-framework-starter-http      Spring 6+ 原生 HTTP Interfaces 客户端
(新增单点登录接入客户端)        →  zhkj-framework-starter-sso-client 通用 SSO 客户端防护与上下文注入
```

### 2.2 包名调整对照
```java
// 旧包名引用
import io.github.ybb88.framework.common.utils.*;
import io.github.ybb88.framework.launch.*;

// 新规范包名引用
import io.github.ybb88.framework.util.*;
import io.github.ybb88.framework.launcher.launch.ZhkjApplication;
```

---

## 🔐 3. 安全体系迁移

### 3.1 配置文件敏感属性解密迁移
- **旧版方式**: Jasypt 的 `jasypt.encryptor.password`；
- **新版方式**: 彻底移除 Jasypt 依赖，改用 `zhkj-framework-starter-crypto` 原生支持：
```yaml
zhkj:
  crypto:
    property:
      enabled: true
      password: "YourStrongMasterKey"
```

### 3.2 敏感数据存储加密
- **旧版方式**: 业务 Service 手动进行对称加密或明文落盘；
- **新版方式**: 实体字段标注 `@TableField(typeHandler = EncryptTypeHandler.class)`，全自动透明加密，配合 `@BlindIndex` 自动注入加盐盲索引列实现安全检索。

---

## 📡 4. 声明式 HTTP RPC 客户端平滑平移

将原 `@FeignClient` 接口平移为现代标准的 `@HttpExchange`：

```java
// 旧版 Feign
@FeignClient(name = "user-service", path = "/api/user")
public interface UserFeignClient {
    @GetMapping("/{id}")
    R<UserVO> getUser(@PathVariable("id") Long id);
}

// 现代版 HTTP Interfaces
@HttpExchange(url = "http://user-service/api/user")
public interface UserHttpClient {
    @GetExchange("/{id}")
    R<UserVO> getUser(@PathVariable("id") Long id);
}
```
在主启动类标注 `@EnableHttpClients(basePackages = "...")` 即可自动扫描并注入 RestClient 代理 Bean。
