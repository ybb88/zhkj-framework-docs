# Web MVC 增强组件：zhkj-framework-starter-web

> **模块定位**: Web MVC 核心运行时增强底座，提供全局统一响应自动封装、全局异常强类型映射、XSS 安全清洗防护、全链路 TraceId 染色与微服务内部 RPC 脱敏旁路放行机制。  
> **适用基准**: OpenJDK 21 (LTS), Spring Boot 4.1.1, Spring Framework 7.x  
> **模块坐标**: `io.github.ybb88:zhkj-framework-starter-web`

---

## 📋 1. 架构定位与核心职责

`zhkj-framework-starter-web` 是所有基于 Spring MVC 的微服务和 Web 应用的必备起步依赖：

1. **统一 Controller 响应自动封装**：拦截 Controller 返回值，统一格式化封装为 `R<T>`（无需在每个方法手写 `return R.data(...)`）；
2. **全局异常强类型映射**：拦截业务异常（`ServiceException`）、参数校验异常（`MethodArgumentNotValidException`）、框架基础异常（`BaseFrameworkException`）及未捕获系统异常，输出标准化的 HTTP 响应状态与错误码；
3. **XSS 安全清洗与 URI 规范化**：基于 `XssFilter` 配合 `StringUtil.cleanPath`，自动清洗请求入参中的恶意跨站脚本，防止 XSS 攻击；
4. **全链路 TraceId 染色**：生成或解析请求头 `X-Trace-Id`，绑定至 MDC 及链路上下文，实现跨服务日志全链路染色追踪；
5. **内部微服务 RPC 脱敏旁路放行机制 (`InternalRpcDesensitizeFilter`)**：识别跨微服务通信请求头（`X-Internal-RPC: true`），临时解除展示态数据脱敏打码，保证内部系统数据交互真实完整性；
6. **请求出口兜底清理 (`FrameworkContextClearFilter`)**：在最高优先级的 Filter 出口处清空当前工作线程持有的所有上下文（租户、身份、脱敏、数据权限等），杜绝线程池复用污染。

---

## 📦 2. 依赖坐标

```xml
<dependency>
    <groupId>io.github.ybb88</groupId>
    <artifactId>zhkj-framework-starter-web</artifactId>
</dependency>
```

> [!CAUTION]
> 本模块专用于基于 Servlet 的 WebMVC 应用。在基于 Spring Cloud Gateway WebFlux 的反应式网关工程中，**严禁引入 `starter-web`**，否则会触发类路径冲突（`Spring MVC found on classpath`）。

---

## 🧱 3. 核心组件与过滤器链路拓扑

```
                    ┌──────────────────────────────────────────────┐
                    │            HTTP / RPC 请求到达                │
                    └──────────────────────┬───────────────────────┘
                                           │
                                           ▼
                    ┌──────────────────────────────────────────────┐
                    │ 1. TraceFilter (链路染色，注入 TraceId)        │
                    └──────────────────────┬───────────────────────┘
                                           │
                                           ▼
                    ┌──────────────────────────────────────────────┐
                    │ 2. InternalRpcDesensitizeFilter (脱敏旁路判断)│
                    │    若携带 X-Internal-RPC: true 则临时关闭脱敏 │
                    └──────────────────────┬───────────────────────┘
                                           │
                                           ▼
                    ┌──────────────────────────────────────────────┐
                    │ 3. XssFilter (输入参数清洗防跨站脚本注入)     │
                    └──────────────────────┬───────────────────────┘
                                           │
                                           ▼
                    ┌──────────────────────────────────────────────┐
                    │ 4. Controller 执行 & GlobalExceptionHandler 拦截│
                    └──────────────────────┬───────────────────────┘
                                           │
                                           ▼
                    ┌──────────────────────────────────────────────┐
                    │ 5. FrameworkContextClearFilter (finally 彻底清空)
                    │    清空 MDC、TenantContext、SecurityContext 等│
                    └──────────────────────────────────────────────┘
```

---

## 🛡️ 4. 核心安全机制深度解析

### 4.1 内部微服务 RPC 脱敏旁路放行机制
- **业务痛点**：Controller 返回的用户信息配置了 `@Desensitize`（如手机号脱敏为 `138****8000`）。前端查询时必须打码防泄密，但微服务间通过声明式 RPC（`@HttpExchange`）调用时，必须获取真实手机号进行短信发送或业务计算。
- **旁路设计**：
  1. 调用方在发起 HTTP 请求时携带 Header：`X-Internal-RPC: true`（常量 `HeaderConstant.X_INTERNAL_RPC`）；
  2. `InternalRpcDesensitizeFilter`（Ordered: `HIGHEST_PRECEDENCE + 20`）检测到此标头时，在当前请求线程中调用 `DesensitizeContext.disable()`；
  3. Controller 处理完毕并在 Jackson 序列化输出真实明文字段；
  4. 过滤器在 `finally` 块中立即恢复 `DesensitizeContext.enable()`，并由外层 `FrameworkContextClearFilter` 兜底彻底清空，确保常驻线程池复用绝对无串号与数据泄露隐患。

### 4.2 全局异常强类型映射
框架统一将异常映射为符合规范的 HTTP 响应体：

| 异常类型 | HTTP 状态码 | 业务错误码 | 处理逻辑 |
| :--- | :--- | :--- | :--- |
| `ServiceException` | 200 OK | 自定义 Code | 业务级失败，返回错误提示 |
| `MethodArgumentNotValidException` | 400 Bad Request | 400 | 表单校验不通过，提取首个校验错误字段 |
| `DistributedLockException` | 400 Bad Request | 400 | 分布式抢锁失败，受控友好提示 |
| `CryptoException` | 500 Server Error | 500 | 国密加解密事务安全阻断异常 |
| `Exception` (未捕获兜底) | 500 Server Error | 500 | 输出日志并返回系统未知异常提示 |

---

## ⚙️ 5. 核心配置清单 (`application.yml`)

```yaml
zhkj:
  # XSS 防护配置
  xss:
    enabled: true                 # 是否开启 XSS 输入清洗 (默认: true)
    skip-url:                     # 白名单放行路径 (支持 Ant 风格)
      - "/doc.html"
      - "/webjars/**"
      - "/v3/api-docs/**"
      - "/actuator/**"

  # 链路追踪配置
  trace:
    enabled: true                 # 是否开启 TraceId 自动染色 (默认: true)
    header-name: "X-Trace-Id"     # 请求头键名
```

---

## 💡 6. 典型使用范式

### 6.1 控制器标准开发
```java
package io.github.ybb88.demo.controller;

import io.github.ybb88.framework.api.R;
import io.github.ybb88.framework.exception.ServiceException;
import io.github.ybb88.demo.dto.OrderCreateDto;
import io.github.ybb88.demo.vo.OrderVo;
import jakarta.validation.Valid;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/v1/orders")
public class OrderController {

    @PostMapping
    public R<OrderVo> createOrder(@Valid @RequestBody OrderCreateDto dto) {
        if (dto.getAmount() == null) {
            throw new ServiceException("订单金额不能为空");
        }
        OrderVo vo = new OrderVo();
        vo.setOrderNo("ORD20261017001");
        return R.data(vo);
    }
}
```
