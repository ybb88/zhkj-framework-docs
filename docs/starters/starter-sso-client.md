# 通用单点登录客户端：zhkj-framework-starter-sso-client

> **模块定位**: 企业级通用单点登录（SSO）客户端 Starter，为微服务与异构业务应用提供免密认证集成、请求未登录无感 302 拦截重定向、OAuth 2.1 授权码无感换取 Token、Cookie/Header 双向透明映射、URI 规范化防绕过、`SsoContextHolder` 线程上下文自动化绑定及单点登出联动。  
> **适用基准**: OpenJDK 21 (LTS), Spring Boot 4.1.1, OAuth 2.1  
> **模块坐标**: `io.github.ybb88:zhkj-framework-starter-sso-client`

---

## 📋 1. 架构定位与核心设计

在大型企业应用群或多微服务矩阵中，用户在各个业务子系统之间切换时，需要保持一致的无缝单点登录体验：

1. **未登录无感拦截重定向**：基于 Servlet Filter 机制，对未携带有效凭据的 Web 请求自动 302 重定向至统一认证收银台；
2. **OAuth 2.1 授权码自动兑换**：用户在收银台认证完成回跳时，过滤器自动拦截 URL 中的 `code` 参数，在服务端向认证中心兑换 JWT Token 与当前用户身份，前端无需接触或明文存储凭证；
3. **Cookie / Header 双向透明映射**：自动向浏览器下发 HttpOnly Cookie，同时完全兼容前后端分离架构下的 `Authorization: Bearer <token>` 头部传递；
4. **URI 路径规范化清洗防绕过**：在白名单匹配与鉴权前统一调用 `StringUtil.cleanPath(uri)`，彻底防御利用多重斜杠（`//admin`）、分号矩阵参数（`/api;jsessionid=...`）或相对路径符号绕过安全拦截；
5. **零反射用户模型归一化**：SSO 解析所得的 `SsoUserContext` 物理继承自框架核心 `ZhkjUser`，直接装配到 `SecurityContextHolder` 与 `FrameworkContext`，杜绝任何反射性能开销与跨系统模型割裂。

---

## 📦 2. 依赖坐标

```xml
<dependency>
    <groupId>io.github.ybb88</groupId>
    <artifactId>zhkj-framework-starter-sso-client</artifactId>
</dependency>
```

---

## 🧱 3. 核心包结构与组件

```
io.github.ybb88.framework.sso.client
├── api/
│   ├── SsoAuthClient.java            # 基于 Spring 6+ @HttpExchange 的认证中心声明式客户端
│   ├── dto/                          # 授权码换 Token 请求 DTO
│   └── vo/                           # Token 响应与身份档案 VO
├── config/
│   └── ZhkjSsoClientAutoConfiguration.java # 客户端自动装配配置类
├── context/
│   ├── SsoContextHolder.java         # 线程级 SSO 上下文持有者 (ThreadLocal)
│   └── SsoUserContext.java           # 自然人与当前系统业务身份模型 (继承 ZhkjUser)
├── filter/
│   └── SsoClientFilter.java          # 核心认证重定向与 Token 换取过滤器
└── props/
    └── SsoClientProperties.java      # SSO 客户端配置属性 (zhkj.sso.client.*)
```

---

## ⚙️ 4. 核心配置清单 (`application.yml`)

```yaml
zhkj:
  sso:
    client:
      enabled: true                           # 是否开启 SSO 客户端无感拦截 (默认: true)
      server-url: "http://sso-auth-server:8050" # 统一认证中心地址 (或内网微服务名)
      client-id: "order-business-service"     # 在认证中心登记的本应用客户端 ID
      client-secret: "sec_order_secret_2026"  # 客户端通讯密钥
      cookie-name: "ZHKJ_SSO_TOKEN"           # 保存 Token 的浏览器 Cookie 键名
      exclude-paths:                          # 免密放行白名单 (支持 Ant 匹配)
        - "/doc.html"
        - "/webjars/**"
        - "/v3/api-docs/**"
        - "/actuator/**"
        - "/public/**"
```

---

## 💡 5. 典型使用范式

### 5.1 业务微服务极速接入
只需在 `pom.xml` 中引入 `zhkj-framework-starter-sso-client`，并在 `application.yml` 中完成上述配置即可，**无需在启动类标注任何繁琐的注解**：

```java
package io.github.ybb88.demo;

import io.github.ybb88.framework.launcher.launch.ZhkjApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class OrderBusinessApplication {

    public static void main(String[] args) {
        ZhkjApplication.run("order-business-service", OrderBusinessApplication.class, args);
    }
}
```

### 5.2 在业务代码中获取当前用户及业务身份
在任意 Controller 或 Service 中直接调用 `SsoContextHolder` 获取当前登录用户：

```java
package io.github.ybb88.demo.controller;

import io.github.ybb88.framework.api.R;
import io.github.ybb88.framework.api.ResultCode;
import io.github.ybb88.framework.sso.client.context.SsoContextHolder;
import io.github.ybb88.framework.sso.client.context.SsoUserContext;
import io.github.ybb88.demo.vo.OrderVo;
import lombok.extern.slf4j.Slf4j;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import java.util.List;

@Slf4j
@RestController
@RequestMapping("/api/v1/orders")
public class OrderQueryController {

    @GetMapping("/my-list")
    public R<List<OrderVo>> getMyOrders() {
        SsoUserContext user = SsoContextHolder.get();
        if (user == null) {
            return R.fail(ResultCode.UN_AUTHORIZED);
        }
        Long personId = user.getPersonId();       // 自然人主账号 ID
        String tenantId = user.getTenantId();     // 当前租户 ID
        String account = user.getIdentityAccount();// 当前系统中的业务账号
        
        log.info("当前操作人 [{}] 正在查询租户 [{}] 的业务订单", account, tenantId);
        
        // 执行业务查询
        return R.data(List.of());
    }
}
```

---

## 🛡️ 6. 常见问题与排错 (FAQ)

1. **微服务间通过 RPC 互相调用时，是否会被 SSO 重定向拦截？**
   - 不会。微服务间使用 `starter-http` 声明式客户端时，`HttpClientHeaderInterceptor` 会自动透传当前请求的 `Authorization: Bearer <token>` 请求头。`SsoClientFilter` 识别到已存在有效凭据直接放行，不会触发 302 重定向。
2. **测试环境下如何免除单点登录拦截？**
   - 在测试配置 `application-test.yml` 中设置 `zhkj.sso.client.enabled: false`，或者将测试接口加入 `exclude-paths` 白名单列表。
