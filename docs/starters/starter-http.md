# 声明式 HTTP 通信组件：zhkj-framework-starter-http

> **模块定位**: 基于 Spring 6+ 原生 HTTP Interfaces（`@HttpExchange`）与 `RestClient` 的声明式 HTTP 客户端组件，提供服务名负载均衡、跨服务请求头透传与降级工厂。  
> **适用基准**: OpenJDK 21 (LTS), Spring Boot 4.1.1, Spring Framework 7.x  
> **模块坐标**: `io.github.ybb88:zhkj-framework-starter-http`

---

## 📋 1. 架构定位与选型背景

本模块使用 Spring 原生 HTTP Interfaces，不依赖 OpenFeign:

1. **标准注解**: 契约接口使用 `@HttpExchange`、`@GetExchange`、`@PostExchange` 等 Spring 注解声明；
2. **统一客户端底座**: 底层为 `RestClient`，请求工厂为 JDK `HttpClient`（`JdkClientHttpRequestFactory`）；
3. **服务名寻址**: 自动装配的 `RestClient.Builder` 带 `@LoadBalanced`，可直接使用注册中心中的服务名作为主机；
4. **请求头透传**: `HttpClientHeaderInterceptor` 从入站 Servlet 请求与 `RpcHeaderProvider` SPI 补充下游请求头。

---

## 📦 2. 依赖坐标

```xml
<dependency>
    <groupId>io.github.ybb88</groupId>
    <artifactId>zhkj-framework-starter-http</artifactId>
</dependency>
```

依赖要点（来自 `pom.xml`）:

| 依赖 | 范围 | 说明 |
| :--- | :--- | :--- |
| `zhkj-framework-core` | compile | `R`、`HeaderConstant`、`TraceUtil`、`RpcHeaderProvider` |
| `spring-web` | compile | `RestClient`、`@HttpExchange` |
| `spring-cloud-starter-loadbalancer` | compile | `@LoadBalanced` 支持 |
| `jakarta.servlet-api` | optional | 仅在 Servlet 环境中提取入站请求头 |

---

## 🧱 3. 核心组件

```
io.github.ybb88.framework.http.client
├── annotation/EnableHttpClients.java          # 启用契约接口扫描
├── config/
│   ├── HttpClientRegistrar.java               # ImportBeanDefinitionRegistrar，扫描并注册 FactoryBean
│   ├── HttpClientFactoryBean.java             # 为每个契约接口创建代理
│   └── HttpClientAutoConfiguration.java       # 自动装配
├── fallback/
│   ├── HttpClientFallbackFactory.java         # 降级代理工厂 (手动使用)
│   └── HttpClientFallback.java                # 降级行为实现
├── header/
│   ├── HttpClientHeaderInterceptor.java       # ClientHttpRequestInterceptor
│   ├── DefaultRpcHeaderProvider.java          # 默认 RpcHeaderProvider (X-Internal-RPC、X-Trace-Id)
│   └── ServletHeaderExtractor.java            # 从 Servlet 请求提取允许透传的头 (包级可见)
└── props/
    ├── HttpClientProperties.java              # zhkj.http.client.*
    └── HttpClientHeadersProperties.java       # zhkj.http.client.headers.*
```

---

## 🔁 4. 自动装配与 Bean

`META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` 注册 `HttpClientAutoConfiguration`，其中：

| Bean | 条件 | 说明 |
| :--- | :--- | :--- |
| `DefaultRpcHeaderProvider` | 缺失时注册 | 提供 `X-Internal-RPC: true` 与 `X-Trace-Id`；RPC 标记不具有认证或授权意义 |
| `HttpClientHeaderInterceptor` | 缺失时注册 | 注入 `List<RpcHeaderProvider>`，组合为 `CompositeRpcHeaderProvider` |
| `loadBalancedRestClientBuilder`（`RestClient.Builder`，`@LoadBalanced`） | 名称缺失时注册 | `JdkClientHttpRequestFactory` + 上述拦截器；连接超时、读超时、重定向策略来自 `zhkj.http.client.*` |

`HttpClientFactoryBean` 在初始化时通过名称 `loadBalancedRestClientBuilder` 获取构建器并 `clone()`，为每个契约接口创建独立 `RestClient`，再由 `HttpServiceProxyFactory` 生成代理。

---

## ⚙️ 5. 配置项

### 5.1 连接参数（`HttpClientProperties`，前缀 `zhkj.http.client`）

| 键 | 类型 | 默认值 | 说明 |
| :--- | :--- | :--- | :--- |
| `zhkj.http.client.connect-timeout` | Duration | `5s` | 连接建立超时 |
| `zhkj.http.client.read-timeout` | Duration | `10s` | 响应读取超时 |
| `zhkj.http.client.follow-redirects` | `HttpClient.Redirect` | `NORMAL` | 重定向策略 |

### 5.2 请求头透传（`HttpClientHeadersProperties`，前缀 `zhkj.http.client.headers`）

| 键 | 类型 | 默认值 | 说明 |
| :--- | :--- | :--- | :--- |
| `zhkj.http.client.headers.enabled` | boolean | `true` | 是否从入站 Servlet 请求复制头 |
| `zhkj.http.client.headers.account` | String | `X-Account-Id` | 账号标识头名称，总是允许复制 |
| `zhkj.http.client.headers.pattern` | String | `x-*` | 通配模式（`PatternMatchUtils.simpleMatch`，同时匹配小写与原始大小写） |
| `zhkj.http.client.headers.allowed` | List<String> | `zhkj-auth`、`Authorization`、`authorization`、`tenant-id`、`x-tenant-id`、`trace-id`、`x-trace-id`、`x-request-id` | 白名单（大小写不敏感） |

### 5.3 示例

```yaml
zhkj:
  http:
    client:
      connect-timeout: 3s
      read-timeout: 8s
      headers:
        enabled: true
        allowed:
          - zhkj-auth
          - Authorization
          - tenant-id
          - trace-id
```

---

## 🧭 6. 请求头透传规则

`HttpClientHeaderInterceptor.intercept` 的顺序：

1. 记录调用方是否已显式设置 `Authorization`。
2. 若 Servlet 环境可用且 `zhkj.http.client.headers.enabled=true`，从当前请求复制满足白名单、`account` 名称或 `pattern` 的头（已存在的头不覆盖，值经 ISO-8859-1 → UTF-8 还原）。
3. 调用 `RpcHeaderProvider`（默认包含 `X-Internal-RPC: true` 与 `X-Trace-Id`）补充缺失的头；值为空白则跳过。
4. 若调用方显式设置了 `Authorization`，则以该值为准，并移除 `zhkj-auth` 与 `Blade-Auth`。

其它模块通过实现 `RpcHeaderProvider` 注入租户、用户等头（例如 `zhkj-framework-starter-tenant` 的 `TenantRpcHeaderProvider`）。

租户请求头标准名为 `X-Tenant-Id`，接收端 `HeaderTenantResolver` 同时兼容 `X-Tenant-ID`、`tenant-id`、`zhkj-tenant-id`。入站复制（仅 Servlet 环境）时，`X-Tenant-Id` 与 `X-Tenant-ID` 由 `pattern`（`x-*`）匹配，`tenant-id` 与 `x-tenant-id` 由 `allowed` 白名单匹配，`zhkj-tenant-id` 既不在默认白名单也不匹配 `x-*`，不会被入站复制；出站时 `TenantRpcHeaderProvider`（由 `zhkj-framework-starter-tenant` 提供）写入 `X-Tenant-Id`、`X-Tenant-ID`、`zhkj-tenant-id`。已存在的请求头不会被覆盖。

| 请求头 | 来源 |
| :--- | :--- |
| `X-Internal-RPC: true` | `DefaultRpcHeaderProvider`，仅标记调用，不触发明文或租户委托授权（见 `starter-web.md`） |
| `X-Trace-Id` | `TraceUtil.getTraceId()` 或入站头 |
| `zhkj-auth`、`Authorization`、`tenant-id` 等 | 入站请求白名单复制 |

**安全行为变更**：普通 RPC 与公共展示端点默认脱敏，携带上述标记不会再关闭 `@Desensitize`。明文取数由接收宿主先完成调用方认证和目标内部端点授权，再受控设置当前 ServletRequest 的 `RpcRequestAttribute.DESENSITIZE_AUTHORIZED` 为 `Boolean.TRUE`，在 `finally` 删除。客户端 provider 不设置也不传播此授权属性；不得将 Header 转换为授权属性。机器认证不是跨租户授权，委托头另需服务端 `TENANT_DELEGATION_AUTHORIZED` 请求属性。迁移步骤见 `starter-web.md` 第 11 节。

---

## 💡 7. 典型使用范式

### 7.1 启用契约扫描
```java
import io.github.ybb88.framework.http.client.annotation.EnableHttpClients;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@EnableHttpClients(basePackages = "io.github.ybb88.demo.client")
@SpringBootApplication
public class DemoApplication {
    public static void main(String[] args) {
        SpringApplication.run(DemoApplication.class, args);
    }
}
```

`@EnableHttpClients` 属性: `value` / `basePackages`（互为别名）、`basePackageClasses`。未指定时使用标注类所在包。

### 7.2 声明契约接口
`HttpClientRegistrar` 仅注册带类级 `@HttpExchange`（或其元注解）的接口。

```java
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

    @GetExchange("/users/{id}")
    R<UserVo> getUserById(@PathVariable("id") Long id);

    @PostExchange("/payments")
    R<PaymentResultVo> submitPayment(@RequestBody PaymentRequestDto request);
}
```

Bean 名称为接口简单名首字母小写，例如 `remoteUserClient`。

### 7.3 注入调用
```java
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
    }
}
```

### 7.4 降级（手动使用）
`HttpClientFallbackFactory` 不会被自动装配。需在业务侧捕获异常后调用：

```java
HttpClientFallbackFactory<RemoteUserClient> factory = HttpClientFallbackFactory.of(RemoteUserClient.class);
RemoteUserClient fallback = factory.create(cause);
```

降级代理行为（`HttpClientFallback`）:

| 返回类型 | 返回值 |
| :--- | :--- |
| `R` 及子类 | `R.fail(code, "远程微服务调用异常: " + cause 消息)`；`code` 为 `RestClientResponseException` 的 HTTP 状态，否则为 `ResultCode.FAILURE` 的 400 |
| `boolean` / `Boolean` | `false` |
| `int` / `Integer` | `-1` |
| 其它 | `null` |

接口类型使用 JDK 动态代理，类类型使用 CGLIB。

---

## 🧷 8. 代理启动与服务发现保障

- `HttpClientRegistrar` 在 BeanDefinition 上声明 `FactoryBean.OBJECT_TYPE_ATTRIBUTE`。Spring 类型扫描无需实例化代理工厂，避免负载均衡 BeanPostProcessor 初始化期间提前创建 `RestClient.Builder`。
- `HttpClientFactoryBean` 必须使用已配置的 `loadBalancedRestClientBuilder`，并通过 `clone()` 为每个代理创建独立构建器。请求头拦截器在自动配置中安装一次，代理工厂不重复追加。
- 获取构建器失败时保留 Spring 启动异常，不再静默使用普通 `RestClient.builder()`。否则服务名会被当作 DNS 主机访问，表现为业务服务可达但声明式 RPC 失败。
- 更新后需重启依赖此工厂的应用以重建已有代理；仅热替换类文件不能修复已创建的客户端实例。使用源码模块运行时重新编译该模块；使用 JAR 时先更新对应 Maven 仓库中的依赖。
- 回归使用实际 Spring 自动配置、服务名负载均衡与本地 HTTP 服务，验证类型扫描后的真实请求及强类型响应；缺失配置必须启动失败。
- 调用方显式设置 `Authorization` 时，该凭据替换整个认证头族：拦截器移除从入站请求复制的 `zhkj-auth`／`Blade-Auth`，避免新自然人凭据与旧业务令牌混用。其余租户与链路请求头仍按既有规则传递。
