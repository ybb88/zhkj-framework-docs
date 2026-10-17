# 多租户治理核心：zhkj-framework-starter-tenant

> **模块定位**: 框架无关多租户治理核心 Starter，提供租户上下文纳管（`TenantContext`）、双模自愈探测机制、租户忽略规则引擎 SPI 及受控切换运行范式。  
> **适用基准**: OpenJDK 21 (LTS), Spring Boot 4.1.1  
> **模块坐标**: `io.github.ybb88:zhkj-framework-starter-tenant`

---

## 📋 1. 架构定位与核心设计

在企业 SaaS 平台与多机构共享体系中，多租户隔离是系统的最高物理安全红线：

1. **框架无关独立抽象**：将租户上下文从具体 ORM 彻底解耦，无论使用 MyBatis-Plus、JPA 还是纯 JDBC，统一依赖 `TenantContext` 调度；
2. **双模自愈探测机制 (Dual-Mode Self-Healing)**：
   - 在批处理、异步任务或 MQ 消费等无 Servlet 请求头的场景下，拦截器首先读取 `TenantContext.getTenantId()`；
   - 若为空，自动动态降级探测安全上下文 `SecurityContextHolder.getTenantId()`；
   - 一旦在安全上下文中捕获到租户身份，立即执行**自动自愈回填**，强制追加 `WHERE tenant_id = 'xxx'`，从根源上杜绝无 Header 时的穿透越权漏洞；
3. **租户规则引擎 SPI (`TenantRuleProvider`)**：
   - 允许业务端通过 SPI 动态定义忽略多租户隔离的全局系统字典表（如 `sys_dict`、`sys_region`）或白名单 URL；
4. **受控临时租户切换 (`runWithTenant`)**：
   - 提供安全的作用域模式切换租户上下文，任务执行完毕后强制自动回滚为原租户，杜绝线程污染。

---

## 📦 2. 依赖坐标

```xml
<dependency>
    <groupId>io.github.ybb88</groupId>
    <artifactId>zhkj-framework-starter-tenant</artifactId>
</dependency>
```

---

## 🧱 3. 核心包结构与类

```
io.github.ybb88.framework.tenant
├── context/
│   ├── TenantContext.java             # 租户上下文持有者 (基于 ThreadLocal)
│   └── TenantContextHolder.java       # 历史兼容持有门面
├── config/
│   ├── TenantAutoConfiguration.java   # 租户核心自动装配
│   └── TenantProperties.java          # 租户全局配置 (列名、忽略表等)
├── provider/
│   └── TenantRuleProvider.java        # 租户规则引擎 SPI 接口
└── util/
    └── TenantUtil.java                # 租户校验与辅助工具
```

---

## ⚙️ 4. 核心配置清单 (`application.yml`)

```yaml
zhkj:
  tenant:
    enabled: true                      # 是否开启全局多租户隔离 (默认: true)
    column: "tenant_id"                # 数据库物理租户隔离列名 (默认: tenant_id)
    ignore-tables:                     # 忽略租户隔离的全局共享表白名单
      - "sys_tenant"
      - "sys_dict"
      - "sys_region"
      - "sys_config"
    ignore-urls:                       # 忽略租户校验的 URL 白名单
      - "/doc.html"
      - "/actuator/**"
```

---

## 💡 5. 典型使用范式

### 5.1 获取与校验当前租户
```java
package io.github.ybb88.demo.service;

import io.github.ybb88.framework.tenant.context.TenantContext;
import org.springframework.stereotype.Service;

@Service
public class TenantDemoService {

    public void processBusiness() {
        // 获取当前请求线程的租户 ID
        String tenantId = TenantContext.getTenantId();
        
        // 执行租户有效性校验
        if (tenantId == null) {
            throw new IllegalStateException("当前未检测到合法租户身份");
        }
    }
}
```

### 5.2 受控跨租户切换运行 (`runWithTenant`)
在系统管理员执行跨租户归档或特权导出时，使用受控闭包执行，执行完毕后原上下文 100% 自动还原：

```java
public void executeForTenant(String targetTenantId) {
    // 在指定租户上下文中执行，无论正常退出还是抛异常，离开后均自动恢复先前状态
    TenantContext.runWithTenant(targetTenantId, () -> {
        // 此处的数据库持久化操作将自动追加 WHERE tenant_id = 'targetTenantId'
        reportService.generateDailySummary();
    });
}
```

### 5.3 扩展租户规则 SPI (`TenantRuleProvider`)
```java
package io.github.ybb88.demo.tenant;

import io.github.ybb88.framework.tenant.provider.TenantRuleProvider;
import org.springframework.stereotype.Component;

import java.util.List;

@Component
public class CustomTenantRuleProvider implements TenantRuleProvider {

    @Override
    public List<String> getIgnoreTables() {
        // 动态追加需要忽略多租户行级过滤的表名
        return List.of("biz_common_metadata");
    }
}
```
