# API 在线接口文档：zhkj-framework-starter-openapi

> **模块定位**: 基于 springdoc-openapi 与 Knife4j 的在线接口文档 Starter，提供 `OpenAPI` 元信息、Knife4j 默认属性注入、全局租户相关请求头参数，以及对 `/v3/api-docs` 等端点的响应包装豁免。  
> **适用基准**: OpenJDK 21 (LTS), Spring Boot 4.1.1, springdoc-openapi 2.8.5, Knife4j 4.5.0（版本取自根 `pom.xml`）  
> **模块坐标**: `io.github.ybb88:zhkj-framework-starter-openapi`

---

## 📋 1. 架构定位与职责

1. **OpenAPI 元信息**: 注册 `OpenAPI` Bean（`info`、`license`、`contact`，可选 OAuth2 `SecurityScheme`）。
2. **默认属性**: `OpenApiEnvironmentPostProcessor` 以最低优先级注入 Knife4j 与 springdoc 的默认开关。
3. **全局请求头参数**: 为所有操作追加 `X-Tenant-Id`、`X-Dimension-Id`、`X-Delegated-Tenant-Id`、`X-Grant-Ticket` 四个可选 Header 参数。
4. **响应包装豁免**: 在 Servlet 环境下，用 `OpenApiAwareResponseBodyAdvice` 替换 `starter-web` 的 `zhkjResponseBodyAdvice`，对 springdoc、Actuator、Knife4j 相关类与路径不做 `R` 包装；并为 `byte[]` 的 JSON 文档注册 `ByteArrayHttpMessageConverter`。
5. **开关**: `zhkj.openapi.enabled`（默认 `true`）控制本模块的 Bean 装配。

---

## 📦 2. 依赖坐标

```xml
<dependency>
    <groupId>io.github.ybb88</groupId>
    <artifactId>zhkj-framework-starter-openapi</artifactId>
</dependency>
```

依赖（来自 `pom.xml`）:

| 依赖 | 范围 | 说明 |
| :--- | :--- | :--- |
| `zhkj-framework-core` | compile | 常量 `HeaderConstant`、`AppConstant` |
| `knife4j-openapi3-jakarta-spring-boot-starter` | compile | 传递 springdoc-openapi |
| `zhkj-framework-starter-web` | optional | 提供 `ZhkjResponseBodyAdvice` 基类（`OpenApiAwareResponseBodyAdvice` 继承它） |

---

## 🧱 3. 包结构

```
io.github.ybb88.framework.openapi
├── OpenApiAutoConfiguration.java        # 自动装配（含静态内部类 OpenApiAwareResponseBodyAdvice）
├── OpenApiProperties.java               # openapi.* 配置
└── env/OpenApiEnvironmentPostProcessor.java

com.github.xiaoymin.knife4j.spring.extension.Knife4jOpenApiCustomizer   # @Deprecated(forRemoval)，未注册为 Bean
```

资源文件:

- `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`: `OpenApiAutoConfiguration`
- `META-INF/spring/org.springframework.boot.EnvironmentPostProcessor.imports` 与 `META-INF/spring.factories`: `OpenApiEnvironmentPostProcessor`

---

## ⚙️ 4. 配置项

### 4.1 模块开关（前缀 `zhkj.openapi`）

| 键 | 默认值 | 说明 |
| :--- | :--- | :--- |
| `zhkj.openapi.enabled` | `true` | 为 `false` 时不装配本模块的 Bean（`OpenAPI`、字节数组转换器、响应豁免 Advice、全局 Header Customizer） |

### 4.2 文档元信息（前缀 `openapi`，`OpenApiProperties`）

| 键 | 默认值 | 说明 |
| :--- | :--- | :--- |
| `openapi.title` | `IOBJ接口文档系统` | 标题 |
| `openapi.description` | `IOBJ接口文档系统` | 描述 |
| `openapi.version` | `0.0.1` | 版本 |
| `openapi.license` / `openapi.license-url` | 空 | 许可证名称与地址（空串时不输出） |
| `openapi.terms-of-service-url` | 空 | 服务条款地址 |
| `openapi.host` | 空 | 仅声明于属性类，当前代码未读取 |
| `openapi.contact.name` / `url` / `email` | `Jason.Yang` / 空 / `274107098@qq.com` | 联系人 |
| `openapi.base-packages` | 空列表 | 为空时写入 `zhkj.base-packages`（代码默认值 `io.github.ybb88`，见下） |
| `openapi.base-path` / `openapi.exclude-path` | 空列表 | 兼容字段，当前代码未读取 |

`openapi.base-packages` 为空时，`OpenApiAutoConfiguration` 以 `@Value("${zhkj.base-packages:io.github.ybb88}")` 的值填充。该值仅写入属性对象，不影响扫描范围（扫描由 springdoc 配置决定）。

### 4.3 鉴权（`openapi.authorization.*`）

| 键 | 默认值 | 说明 |
| :--- | :--- | :--- |
| `openapi.authorization.enabled` | `false` | 为 `true` 时注册 OAuth2 `SecurityScheme`（password flow）并全局启用 |
| `openapi.authorization.name` | `Authorization` | SecurityScheme 名称 |
| `openapi.authorization.auth-regex` | `^.*$` | 作为 SecurityScheme 描述输出 |
| `openapi.authorization.token-url-list` | 空 | 取第一个作为 `tokenUrl` |
| `openapi.authorization.authorization-scope-list` | 空 | 元素含 `scope`、`description` |

### 4.4 springdoc / Knife4j 默认值（由 `OpenApiEnvironmentPostProcessor` 注入）

| 键 | 注入值 | 说明 |
| :--- | :--- | :--- |
| `knife4j.enable` | `true` | Knife4j 总开关 |
| `knife4j.production` | 激活 Profile 含 `prod` 时为 `true` | 生产环境屏蔽 Knife4j 增强 UI（源码注释：生产环境关闭 knife4j 增强 UI） |
| `springdoc.api-docs.enabled` | `true` | 文档端点 |
| `springdoc.swagger-ui.enabled` | `true` | Swagger UI |
| `springdoc.enable-data-rest` / `springdoc.enable-spring-data-rest` | `false` | 关闭 Spring Data REST 集成 |

注入的属性源名为 `zhkjOpenApiDefaultProperties`，位于属性源列表末尾，业务配置（`application.yml`、Nacos 等）中的同名键会覆盖这些值。若该名称的属性源已存在则跳过注入。

---

## 🔒 5. 生产环境关闭

> [!IMPORTANT]
> 当前代码中，`zhkj.openapi.enabled=false` 仅使本模块的 Bean 不被装配。springdoc 自身的端点开关由 `OpenApiEnvironmentPostProcessor` 设为 `springdoc.api-docs.enabled=true` 与 `springdoc.swagger-ui.enabled=true`。因此生产环境若需关闭 `/v3/api-docs` 与 Swagger UI，需在业务配置中显式设置 `springdoc.api-docs.enabled: false` 与 `springdoc.swagger-ui.enabled: false`（本模块未提供一键关闭开关）。

```yaml
springdoc:
  api-docs:
    enabled: false
  swagger-ui:
    enabled: false
```

---

## 🧩 6. Bean 说明

| Bean | 条件 | 作用 |
| :--- | :--- | :--- |
| `OpenAPI openAPI(OpenApiProperties)` | 缺失时 | 设置 `Info`、`License`、`Contact`；鉴权启用时注册 `SecurityScheme` |
| `WebMvcConfigurer openApiByteArrayConverterConfigurer()` | Servlet 且缺失时 | 在 `extendMessageConverters` 首位插入 `ByteArrayHttpMessageConverter`（支持 `application/json`、`application/octet-stream`、`*/*`），使 springdoc `openapiJson()` 的 `byte[]` 原样输出 |
| `BeanDefinitionRegistryPostProcessor openApiResponseBodyAdviceRegistrar()` | Servlet 且缺失时 | 移除 `zhkjResponseBodyAdvice`，注册 `openApiAwareResponseBodyAdvice`；移除 `queryDslQuerydslPredicateOperationCustomizer` |
| `GlobalOpenApiCustomizer globalTenantOpenApiCustomizer()` | 缺失时 | 为每个操作追加 `X-Tenant-Id`、`X-Dimension-Id`、`X-Delegated-Tenant-Id`、`X-Grant-Ticket` 可选 Header |

`OpenApiAwareResponseBodyAdvice`（`OpenApiAutoConfiguration` 的静态内部类，`@RestControllerAdvice`）继承 `ZhkjResponseBodyAdvice`，对以下情形直接返回原始响应体（`supports` 返回 false 或 `beforeBodyWrite` 原样返回）:

- 方法声明类的包名以 `org.springdoc.`、`org.springframework.boot.actuate.`、`com.github.xiaoymin.` 开头；
- 请求路径包含 `/v3/api-docs`、`/swagger`、`/actuator`。

---

## 💡 7. 使用示例

### 7.1 Controller 标注
```java
import io.github.ybb88.framework.api.R;
import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.Parameter;
import io.swagger.v3.oas.annotations.tags.Tag;
import org.springframework.web.bind.annotation.*;

@Tag(name = "订单管理", description = "订单创建与查询")
@RestController
@RequestMapping("/api/v1/orders")
public class OrderApiController {

    @Operation(summary = "创建订单", description = "提交订单基础信息")
    @PostMapping
    public R<OrderVo> createOrder(@RequestBody OrderCreateDto dto) {
        return R.data(orderService.create(dto));
    }

    @Operation(summary = "根据订单编号查询详情")
    @GetMapping("/{orderNo}")
    public R<OrderVo> getOrderDetail(
            @Parameter(description = "订单业务编号", required = true) @PathVariable String orderNo) {
        return R.data(orderService.getByNo(orderNo));
    }
}
```

### 7.2 访问地址
- Knife4j 文档页: `http://localhost:8080/doc.html`（Knife4j 默认路径；本模块未覆盖 springdoc / Knife4j 路径配置）
- OpenAPI JSON: `http://localhost:8080/v3/api-docs`
- Swagger UI: `http://localhost:8080/swagger-ui/index.html`

### 7.3 `Knife4jOpenApiCustomizer`
该类位于 `com.github.xiaoymin.knife4j.spring.extension` 包，标注 `@Deprecated(since = "2026.4.1.0", forRemoval = true)`，未标注 `@Component` 且未由自动装配注册，当前不参与文档生成。

---

## 🔗 8. 相关文档

- Web 层与 `ZhkjResponseBodyAdvice`: [starter-web](./starter-web.md)
- 核心响应模型 `R`: [core](./core.md)
