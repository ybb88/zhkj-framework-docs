# 注册与配置中心：zhkj-framework-starter-nacos

> **模块定位**: 微服务服务注册与配置中心 Starter，基于 Alibaba Nacos 深度整合，提供服务自动注册、动态心跳探测、分布式配置集中纳管与热更新刷新机制。  
> **适用基准**: OpenJDK 21 (LTS), Spring Boot 4.1.1, Spring Cloud Alibaba 2025  
> **模块坐标**: `io.github.ybb88:zhkj-framework-starter-nacos`

---

## 📋 1. 架构定位与核心设计

在分布式微服务架构中，服务实例的动态寻址与集中式配置管理是微服务治理的基石：

1. **服务自动注册与健康嗅探**：应用启动后自动将服务名、IP、监听端口及集群元数据上报至 Nacos 注册中心，并保持心跳检测；
2. **配置集中管理与动态热刷新**：支持基于 DataId 与 Group 进行多环境配置隔离（`dev`、`test`、`prod`），配合 Spring Cloud 机制实现配置热刷新（`@RefreshScope`），修改配置无需重启服务；
3. **彻底解耦历史加密逻辑**：完全剔除早期分散在 Nacos 内部的 Jasypt 与 RSA 加密逻辑，所有配置解密统一由 `starter-crypto` 在 `EnvironmentPostProcessor` 阶段收敛承载，Nacos 模块职责纯粹单一；
4. **元数据动态感知**：自动将当前工程构件版本及环境变量写入微服务元数据（Metadata），便于网关执行基于版本的灰度金丝雀发布。

---

## 📦 2. 依赖坐标

```xml
<dependency>
    <groupId>io.github.ybb88</groupId>
    <artifactId>zhkj-framework-starter-nacos</artifactId>
</dependency>
```

---

## 🧱 3. 核心包结构与组件

```
io.github.ybb88.framework.nacos
├── config/
│   ├── ZhkjNacosAutoConfiguration.java # Nacos 增强自动装配类
│   └── NacosProperties.java           # Nacos 参数配置属性
└── constant/
    └── NacosConstant.java             # Nacos 默认配置分组与上下文常量
```

---

## ⚙️ 4. 核心配置清单 (`bootstrap.yml` / `application.yml`)

```yaml
spring:
  application:
    name: demo-service
  cloud:
    nacos:
      # 1. 服务发现与注册中心配置
      discovery:
        server-addr: 127.0.0.1:8848
        namespace: "public"
        group: "DEFAULT_GROUP"
        metadata:
          version: "2026.4.1.0"
      # 2. 分布式配置中心配置
      config:
        server-addr: 127.0.0.1:8848
        namespace: "public"
        group: "DEFAULT_GROUP"
        file-extension: "yaml"
        refresh-enabled: true
        # 共享与扩展配置
        shared-configs:
          - data-id: "zhkj-common.yaml"
            group: "DEFAULT_GROUP"
            refresh: true
```

---

## 💡 5. 典型使用范式

### 5.1 动态配置热刷新 (`@RefreshScope`)
在 Spring Bean 上标注 `@RefreshScope`，当在 Nacos 控制台修改配置并发布时，Spring 会自动更新 Bean 中的属性值：

```java
package io.github.ybb88.demo.config;

import lombok.Data;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.cloud.context.config.annotation.RefreshScope;
import org.springframework.stereotype.Component;

@Data
@Component
@RefreshScope
@ConfigurationProperties(prefix = "biz.feature")
public class FeatureSwitchProperties {

    /**
     * Nacos 控制台动态修改后，无需重启微服务，实时生效
     */
    private boolean orderApprovalEnabled = false;
    private int maxRetryAttempts = 3;
}
```
