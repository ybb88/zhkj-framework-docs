# 安全鉴权中心：zhkj-framework-starter-secure

> **模块定位**: 轻量级统一鉴权中心 Starter，基于 JJWT 0.12.5 现代安全套件构建无状态安全认证架构，提供 Token 签发与解析、方法级细粒度权限控制（`@PreAuth`）、Actuator 运维端点防护与上下文生命周期治理。  
> **适用基准**: OpenJDK 21 (LTS), Spring Boot 4.1.1, Spring Security 7.x, JJWT 0.12.5  
> **模块坐标**: `io.github.ybb88:zhkj-framework-starter-secure`

---

## 📋 1. 架构定位与核心设计

在企业分布式应用中，轻量、无状态且安全的身份认证是系统的第一道防线：

1. **JJWT 0.12+ 现代流式架构**：
   - 彻底废弃早期 JJWT 的 `setSigningKey` 与 `SignatureAlgorithm` 淘汰语法；
   - 统一采用现代流式语法：`Jwts.builder().claims(...).signWith(secretKey).compact()`；
2. **方法级细粒度权限校验 (`@PreAuth`)**：
   - 支持基于 SpEL 表达式进行角色与权限判断（如 `@PreAuth("hasRole('admin')")` 或 `@PreAuth("hasPermission('order:export')")`）；
3. **运维端点安全隔离 (`ActuatorSecurityConfiguration`)**：
   - 针对 `/actuator/**` 敏感健康与指标端点进行安全权限收敛，防止内部拓扑与配置泄漏；
4. **安全上下文与 ThreadLocal 纳管 (`SecurityContextHolder`)**：
   - 基于标准 `ThreadLocal<ZhkjUser>` 维护当前操作人上下文，严禁使用原生 `InheritableThreadLocal`；
   - 自动挂载至 `FrameworkContext` 清理链路，在请求结束时彻底清空，防止线程池复用污染。

---

## 📦 2. 依赖坐标

```xml
<dependency>
    <groupId>io.github.ybb88</groupId>
    <artifactId>zhkj-framework-starter-secure</artifactId>
</dependency>
```

---

## 🧱 3. 核心包结构与组件

```
io.github.ybb88.framework.secure
├── annotation/
│   └── PreAuth.java                   # 方法级权限校验注解
├── aspect/
│   └── AuthAspect.java                # @PreAuth 权限切面执行器
├── config/
│   ├── SecureAutoConfiguration.java   # 安全模块自动装配
│   ├── ActuatorSecurityConfiguration.java # Actuator 端点保护
│   └── SecureProperties.java          # 安全配置属性
├── provider/
│   └── IClientDetailsProvider.java    # 客户端凭据验证 SPI
└── util/
    ├── SecureUtil.java                # Token 签发、校验与密钥生成工具
    └── SecurityContextHolder.java     # 安全上下文持有者
```

---

## ⚙️ 4. 核心配置清单 (`application.yml`)

```yaml
zhkj:
  secure:
    enabled: true                      # 是否开启安全拦截校验 (默认: true)
    sign-key: "ZhkjSecureTokenMasterKey2026StrongKeyMustBe32Chars!" # JWT 签名主密钥 (建议 256 位以上)
    token-expire: 86400                # AccessToken 过期时间 (秒，默认 24 小时)
    refresh-token-expire: 604800       # RefreshToken 过期时间 (秒，默认 7 天)
    exclude-patterns:                  # 白名单放行路径
      - "/doc.html"
      - "/webjars/**"
      - "/v3/api-docs/**"
      - "/actuator/health"
```

---

## 💡 5. 典型使用范式

### 5.1 签发与解析 JWT Token (`SecureUtil`)
```java
package io.github.ybb88.demo.service;

import io.github.ybb88.framework.secure.util.SecureUtil;
import io.github.ybb88.framework.model.ZhkjUser;
import org.springframework.stereotype.Service;

@Service
public class AuthService {

    public String generateUserToken(Long userId, String username, String tenantId) {
        ZhkjUser user = new ZhkjUser();
        user.setUserId(userId);
        user.setAccount(username);
        user.setTenantId(tenantId);
        user.setRoleName("admin");

        // 签发带有标准 Claim 的 JWT Token
        return SecureUtil.createToken(user);
    }
}
```

### 5.2 获取当前登录操作人身份
在任何业务 Controller 或 Service 中直接读取：

```java
import io.github.ybb88.framework.secure.util.SecurityContextHolder;
import io.github.ybb88.framework.model.ZhkjUser;

ZhkjUser currentUser = SecurityContextHolder.getUser();
if (currentUser != null) {
    Long userId = currentUser.getUserId();
    String tenantId = currentUser.getTenantId();
    String roleName = currentUser.getRoleName();
}
```

### 5.3 方法级权限拦截 (`@PreAuth`)
```java
package io.github.ybb88.demo.controller;

import io.github.ybb88.framework.api.R;
import io.github.ybb88.framework.secure.annotation.PreAuth;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/v1/admin")
public class AdminController {

    /**
     * 仅具有 admin 角色或包含用户删除权限的用户允许访问
     */
    @DeleteMapping("/users/{id}")
    @PreAuth("hasRole('admin') or hasPermission('sys:user:delete')")
    public R<Void> deleteUser(@PathVariable Long id) {
        // 业务逻辑
        return R.success("删除成功");
    }
}
```
若当前登录主体未满足权限表达式，切面自动拦截并抛出受控异常，向前端响应 `403 Forbidden`。
