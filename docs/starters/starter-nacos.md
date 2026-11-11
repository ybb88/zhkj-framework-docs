# 注册与配置中心：zhkj-framework-starter-nacos

> **模块定位**: 对 Spring Cloud Alibaba Nacos 的薄封装。本模块在容器初始化时（`@PostConstruct`）向 `NacosDiscoveryProperties` 的 metadata 补充框架版本、服务版本、运行 Profile 与启动时间；注册中心与配置中心的连接参数由 Spring Cloud Alibaba 标准配置项提供。  
> **适用基准**: OpenJDK 21 (LTS), Spring Boot 4.1.1, Spring Cloud 2025.1.2, Spring Cloud Alibaba 2025.1.0.0  
> **模块坐标**: `io.github.ybb88:zhkj-framework-starter-nacos`

---

## 📋 1. 架构定位与职责

1. **元数据补充**: 当类路径存在 `NacosDiscoveryProperties` 时，`ZhkjNacosAutoConfiguration` 在初始化阶段向 `spring.cloud.nacos.discovery.metadata` 写入以下键（仅在键不存在时写入）:

   | 元数据键 | 值来源 |
   | :--- | :--- |
   | `framework.version` | 本模块 Jar 的 `Implementation-Version`，否则 `Specification-Version`，均缺失时为 `UNKNOWN` |
   | `service.version` | `info.version`（非空时），否则为 `framework.version` |
   | `service.profile` | `spring.profiles.active`，缺省 `default` |
   | `startup.time` | 启动时间，格式 `yyyy-MM-dd HH:mm:ss` |

2. **配置中心与注册中心**: 本模块不定义连接参数，也不引入新的配置键。`spring-cloud-starter-alibaba-nacos-discovery` 与 `spring-cloud-starter-alibaba-nacos-config` 作为依赖由 `pom.xml` 引入。
3. **无 Bean 注册**: `ZhkjNacosAutoConfiguration` 不向容器注册业务 Bean，仅执行元数据补充。
4. **加密解耦**: 敏感配置的 `ENC(...)` 解密由 `zhkj-framework-starter-crypto` 的 `EncryptablePropertyEnvironmentPostProcessor` 统一负责（包装环境中已存在的可包装属性源），本模块不处理解密。框架现行方案不含 Jasypt 或 RSA。Nacos 配置须在该后置处理器执行前已装入环境 `[待确认]`（装入时序由 Spring Cloud Alibaba 的配置导入机制决定，本仓库未验证）。

---

## 📦 2. 依赖坐标

```xml
<dependency>
    <groupId>io.github.ybb88</groupId>
    <artifactId>zhkj-framework-starter-nacos</artifactId>
</dependency>
```

依赖（来自 `pom.xml`）: `zhkj-framework-core`、`spring-boot-starter`、`spring-cloud-starter-alibaba-nacos-discovery`、`spring-cloud-starter-alibaba-nacos-config`。

---

## 🧱 3. 包结构

```
io.github.ybb88.framework.nacos
└── config/
    └── ZhkjNacosAutoConfiguration.java
resources/META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
```

---

## ⚙️ 4. 标准配置示例

以下键属于 Spring Cloud Alibaba，由 `spring-cloud-starter-alibaba-nacos-*` 解析:

```yaml
spring:
  application:
    name: demo-service
  cloud:
    nacos:
      discovery:
        server-addr: 127.0.0.1:8848
        group: DEFAULT_GROUP
        metadata:
          team: order
      config:
        server-addr: 127.0.0.1:8848
        group: DEFAULT_GROUP
        file-extension: yaml
        refresh-enabled: true
        shared-configs:
          - data-id: zhkj-common.yaml
            group: DEFAULT_GROUP
            refresh: true
```

`info.version` 由启动器写入（值为 `AppConstant.APPLICATION_VERSION`，当前为 `3.0.0`，见 [launcher](./launcher.md)），因此通过 `ZhkjApplication` 启动的服务其 `service.version` 元数据为该值。

---

## 💡 5. 使用说明

### 5.1 动态配置热刷新
在 Bean 上标注 `@RefreshScope` 后，Nacos 中的配置变更会刷新该 Bean 的属性（Spring Cloud 标准机制）。

```java
@Data
@Component
@RefreshScope
@ConfigurationProperties(prefix = "biz.feature")
public class FeatureSwitchProperties {

    private boolean orderApprovalEnabled = false;
    private int maxRetryAttempts = 3;
}
```

### 5.2 读取实例元数据
```java
@Autowired
private DiscoveryClient discoveryClient;

public String serviceVersion(String serviceName) {
    return discoveryClient.getInstances(serviceName).get(0).getMetadata().get("service.version");
}
```

---

## 🔗 6. 相关文档

- 启动器: [launcher](./launcher.md)
- 加密配置: `zhkj-framework-starter-crypto`（`ENC(...)` 解密）
