# API 在线接口文档：zhkj-framework-starter-openapi

> **模块定位**: API 在线接口文档 Starter，基于 Knife4j 4.5.0 与 OpenAPI 3.0 (Swagger 3) 构建，提供美观直观的交互式接口调试界面、全局请求头自动挂载与生产环境安全关闭机制。  
> **适用基准**: OpenJDK 21 (LTS), Spring Boot 4.1.1, Knife4j 4.5.x, OpenAPI 3.0  
> **模块坐标**: `io.github.ybb88:zhkj-framework-starter-openapi`

---

## 📋 1. 架构定位与核心设计

在前后端协同与多团队协作中，接口文档的时效性与易用性直接决定了研发交付效率：

1. **全面基于 OpenAPI 3.0 现代规范**：彻底淘汰过时的 Swagger 2 (`@Api`, `@ApiOperation`)，全面采用 OpenAPI 3 标准注解（`@Tag`, `@Operation`, `@Parameter`）；
2. **全局自动挂载鉴权与租户请求头**：
   - 自动在 Knife4j 调试界面中注册 `Authorization` (Bearer Token) 与 `X-Tenant-Id` 全局安全参数，调试接口时无需每次手动复制粘贴 Header；
3. **Jackson 3 架构纪律与冲突免疫**：
   - 针对底层泛型实体通过强类型桥接字节码消除 Setter 冲突，确保文档扫描引擎 100% 稳定生成接口模型；
4. **生产环境安全关闭机制**：
   - 支持通过单一开关（`zhkj.openapi.enabled: false`）一键关闭线上文档与元数据端点，杜绝接口信息暴露隐患。

---

## 📦 2. 依赖坐标

```xml
<dependency>
    <groupId>io.github.ybb88</groupId>
    <artifactId>zhkj-framework-starter-openapi</artifactId>
</dependency>
```

---

## 🧱 3. 核心包结构与组件

```
io.github.ybb88.framework.openapi
├── config/
│   ├── OpenApiAutoConfiguration.java  # Knife4j 与 OpenAPI 自动装配配置
│   └── OpenApiProperties.java         # 文档标题、作者、版本与安全协议配置
└── constant/
    └── OpenApiConstant.java           # 接口文档默认常量
```

---

## ⚙️ 4. 核心配置清单 (`application.yml`)

```yaml
zhkj:
  openapi:
    enabled: true                      # 是否开启在线接口文档 (默认: true，生产环境建议关闭)
    title: "微服务核心业务 API 文档"     # 文档标题
    description: "基于 ZHKJ Framework 企业底座构建的微服务接口中心"
    version: "2026.4.1.0"              # 接口文档版本
    contact-name: "架构组"
    contact-email: "arch@zhkj.internal"
```

---

## 💡 5. 典型使用范式

### 5.1 编写带有 OpenAPI 3.0 注解的 Controller
```java
package io.github.ybb88.demo.controller;

import io.github.ybb88.framework.api.R;
import io.github.ybb88.demo.dto.OrderCreateDto;
import io.github.ybb88.demo.vo.OrderVo;
import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.Parameter;
import io.swagger.v3.oas.annotations.tags.Tag;
import jakarta.validation.Valid;
import org.springframework.web.bind.annotation.*;

@Tag(name = "01. 订单管理模块", description = "提供订单创建、查询与状态变更接口")
@RestController
@RequestMapping("/api/v1/orders")
public class OrderApiController {

    @Operation(summary = "创建订单", description = "提交订单基础信息，自动关联租户与操作人")
    @PostMapping
    public R<OrderVo> createOrder(@Valid @RequestBody OrderCreateDto dto) {
        OrderVo vo = new OrderVo();
        vo.setOrderNo("ORD20261017001");
        return R.data(vo);
    }

    @Operation(summary = "根据订单编号查询详情")
    @GetMapping("/{orderNo}")
    public R<OrderVo> getOrderDetail(
        @Parameter(description = "订单业务编号", required = true) @PathVariable String orderNo
    ) {
        OrderVo vo = new OrderVo();
        vo.setOrderNo(orderNo);
        return R.data(vo);
    }
}
```

### 5.2 在线访问接口文档
服务启动后，在浏览器中访问：
- **Knife4j 美化文档界面**: `http://localhost:8080/doc.html`
- **原生 OpenAPI 3.0 JSON 元数据**: `http://localhost:8080/v3/api-docs`
