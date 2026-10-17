# 声明式 HTTP 通信组件：zhkj-framework-starter-http

> **模块定位**: 现代化声明式 HTTP 客户端组件，基于 Spring 6+ 原生 HTTP Interfaces（`@HttpExchange`）构建，全面替代已废弃停滞的 OpenFeign，集成全链路 Header 透明透传、内部 RPC 脱敏旁路标头与优雅降级。  
> **适用基准**: OpenJDK 21 (LTS), Spring Boot 4.1.1, Spring Framework 7.x  
> **模块坐标**: `io.github.ybb88:zhkj-framework-starter-http`

---

## 📋 1. 架构定位与选型背景

传统微服务大多使用 `spring-cloud-starter-openfeign`，但在现代化微服务体系中存在如下重大弊端：
1. **社区停滞**: OpenFeign 已停止主线演进，不再跟进响应式与轻量化改进；
2. **底层割裂**: Feign 自带一套编码解码体系，与 Spring 6 原生 `RestClient` 存在体系摩擦；
3. **上下文断流**: Feign 的拦截器通常需要手工维护 `RequestInterceptor`，在异步线程或批处理中无法自动捕获上下文。

`zhkj-framework-starter-http` 采用 **Spring 6+ 原生 HTTP Interfaces**：
- **原生拥抱标准**: 使用标准 `@HttpExchange`、`@GetExchange` 等注解声明式定义远程客户端；
- **全链路自动透传**: 内置 `HttpClientHeaderInterceptor`，自动透明透传 `X-Tenant-Id`、`X-Trace-Id`、`Authorization` 以及 `X-Internal-RPC: true`；
- **负载均衡与微服务互通**: 配合 Spring Cloud LoadBalancer 与 Nacos，开箱支持基于微服务服务名的透明寻址与负载均衡。

---

## 📦 2. 依赖坐标

```xml
<dependency>
    <groupId>io.github.ybb88</groupId>
    <artifactId>zhkj-framework-starter-http</artifactId>
</dependency>
```

---

## 🧱 3. 核心组件与机制

```
io.github.ybb88.framework.http.client
├── annotation/
│   └── EnableHttpClients.java         # 扫描并注册声明式客户端注解
├── config/
│   ├── HttpClientAutoConfiguration.java# 客户端自动装配
│   └── HttpClientProperties.java      # 通信参数配置
├── interceptor/
│   └── HttpClientHeaderInterceptor.java# 全链路上下文请求头透传拦截器
└── factory/
    └── HttpServiceProxyFactory.java   # 动态代理生成工厂
```

### 3.1 跨服务请求头自动透传矩阵
当通过 HTTP 客户端发起请求时，`HttpClientHeaderInterceptor` 会自动从 `FrameworkContext` 中抓取当前上下文，并将以下请求头透明写入下游请求：

| 请求头标名 | 取值来源 | 作用说明 |
| :--- | :--- | :--- |
| `X-Tenant-Id` | `TenantContext.getTenantId()` | 保证下游服务在多租户物理/行级隔离中执行 |
| `X-Trace-Id` | `TraceUtil.getTraceId()` | 保持全链路日志 TraceId 连续染色 |
| `Authorization` | `SecurityContextHolder.getToken()` | 透传当前操作人的认证 Token 凭证 |
| `X-Internal-RPC` | 默认注入 `true` | 告知下游服务触发 `InternalRpcDesensitizeFilter` 放行明文数据 |

---

## ⚙️ 4. 核心配置清单 (`application.yml`)

```yaml
zhkj:
  http:
    client:
      enabled: true                   # 是否启用声明式客户端 (默认: true)
      connect-timeout: 5000           # 连接超时时间 (ms)
      read-timeout: 10000             # 读取超时时间 (ms)
      disable-ssl-validation: false   # 是否忽略 SSL 证书校验 (测试环境可置为 true)
      headers:
        enabled: true                 # 是否开启全链路请求头自动透传 (默认: true)
```

---

## 💡 5. 典型使用范式

### 5.1 在主启动类启用客户端扫描
```java
package io.github.ybb88.demo;

import io.github.ybb88.framework.http.client.annotation.EnableHttpClients;
import io.github.ybb88.framework.launcher.launch.ZhkjApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@EnableHttpClients(basePackages = "io.github.ybb88.demo.client")
@SpringBootApplication
public class DemoApplication {

    public static void main(String[] args) {
        ZhkjApplication.run("demo-service", DemoApplication.class, args);
    }
}
```

### 5.2 声明远程微服务通信接口
直接使用 Spring 原生注解声明 HTTP 契约，支持服务名寻址或绝对 URL：

```java
package io.github.ybb88.demo.client;

import io.github.ybb88.framework.api.R;
import io.github.ybb88.demo.dto.PaymentRequestDto;
import io.github.ybb88.demo.vo.PaymentResultVo;
import io.github.ybb88.demo.vo.UserVo;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.service.annotation.GetExchange;
import org.springframework.web.service.annotation.HttpExchange;
import org.springframework.web.service.annotation.PostExchange;

@HttpExchange(url = "http://system-service/api/v1")
public interface RemoteUserClient {

    /**
     * 获取用户信息（下游 Controller 的 @Desensitize 字段会自动被旁路放行明文）
     */
    @GetExchange("/users/{id}")
    R<UserVo> getUserById(@PathVariable("id") Long id);

    /**
     * 提交交易支付
     */
    @PostExchange("/payments")
    R<PaymentResultVo> submitPayment(@RequestBody PaymentRequestDto request);
}
```

### 5.3 业务服务注入与调用
声明式接口无需手工编写实现类，由框架在启动期自动创建并注册为 Spring Bean，业务 Service 直接注入即可调用：

```java
package io.github.ybb88.demo.service;

import io.github.ybb88.framework.api.R;
import io.github.ybb88.demo.client.RemoteUserClient;
import io.github.ybb88.demo.vo.UserVo;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;

@Service
@RequiredArgsConstructor
public class OrderService {

    private final RemoteUserClient remoteUserClient;

    public void verifyOrderUser(Long userId) {
        R<UserVo> result = remoteUserClient.getUserById(userId);
        if (!result.isSuccess() || result.getData() == null) {
            throw new IllegalStateException("用户远程校验失败: " + result.getMsg());
        }
        UserVo user = result.getData();
        // 此处的 user.getMobile() 是下游未打码的真实手机号
    }
}
```
