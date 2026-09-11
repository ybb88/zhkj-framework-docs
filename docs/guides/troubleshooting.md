# ZHKJ Framework 常见问题与排错手册 (Troubleshooting FAQ)

> **文档定位**: 汇总在微服务开发、多模块编译、多租户隔离、批处理及安全环境初始化中最常见的异常报错与标准排查解决方案。

---

## 📋 常见故障与排查速查表

| 序号 | 错误现象 / 报错日志 | 根本原因剖析 | 标准排查与解决办法 |
| :--- | :--- | :--- | :--- |
| **1** | `TenantId cannot be null` 或多租户 SQL 过滤报错 | 业务未携带 Header 且 Token 中无租户信息，或异步线程中未注入租户上下文 | 1. 检查是否在 `zhkj.tenant.ignore-tables` 中配置了白名单；<br>2. 在异步线程/MQ/批处理中，使用 `TenantContext.runWithTenant("tenantId", ...)` 或 `FrameworkContext.openScope(snapshot)` 透传上下文。 |
| **2** | `No ScopeModelHandler bean found` | 实体或 Mapper 使用了 `@DataAuth`，但 Spring 容器中未注入 `ScopeModelHandler` 实现 | 在业务应用中实现 `ScopeModelHandler` SPI 接口，并添加 `@Component` 注解注册为 Spring Bean。 |
| **3** | `ClassCastException` 在松散绑定阶段报错 (Spring Boot 4 启动阶段解密) | 自定义 `EnvironmentPostProcessor` 误将内部特殊属性源 `configurationProperties` 替换为普通包，或未重新挂载 | 1. 严禁包装 `ConfigurationPropertySourcesPropertySource`；<br>2. 包装完所有 PropertySource 后，**必须在末尾执行 `ConfigurationPropertySources.attach(environment)`**。 |
| **4** | HTTP 跨服务接口调用报 `401 Unauthorized` | 声明式 HTTP 客户端调用时未自动透传认证 Header 或租户 Header | 确保调用方引入了 `zhkj-framework-starter-http`，并确认配置项 `zhkj.http.client.headers.enabled: true`。 |
| **5** | Swagger / Knife4j 页面报 404 或无数据 | 接口文档静态资源或 API 文档端点被安全拦截器拦截 | 在 `application.yml` 的 `zhkj.secure.exclude-patterns` 中追加配置：<br>`/doc.html`, `/webjars/**`, `/v3/api-docs/**`。 |
| **6** | `JobLocator is required` (Spring Batch 6) | Spring Batch 6 采用 `TaskExecutorJobOperator` 时必须显式提供 `JobRegistry` | 在批处理配置类中声明并注入：<br>`@Bean public JobRegistry jobRegistry() { return new MapJobRegistry(); }`。 |
| **7** | `非法字符: '\ufeff'` (编译期报错) | Windows PowerShell 脚本在生成/修改 Java 源码时写入了带 BOM 的 UTF-8 文件 | 严禁使用带 BOM 的 UTF-8。使用 Node.js (`utf8` 无 BOM) 或在 PowerShell 中通过 `New-Object System.Text.UTF8Encoding($false)` 重写文件。 |
| **8** | OkHttp SSL 握手异常或证书校验失败 | 内部测试或私有云环境使用了自签名证书 | 在配置文件中开启测试信任开关：<br>`zhkj.http.client.disable-ssl-validation: true`，底层会自动注入信任所有证书的 SSLContext。 |
| **9** | `NoClassDefFoundError: R` 或方法签名不匹配 | Maven 多模块跨模块修改后，上游依赖未安装到本地仓库，本地存在增量编译脏类 | 执行 `mvn clean compile -pl <module> -am -DskipTests=true` 重新编译上游，或根目录执行 `mvn clean install -DskipTests=true` 刷新依赖。 |
| **10** | `Spring MVC found on classpath, Spring Cloud Gateway requires WebFlux` | 网关工程误引入了依赖 Servlet/Tomcat 的 Starter | 响应式网关工程（基于 Spring Cloud Gateway WebFlux）只允许引入 `spring-cloud-starter-gateway-server-webflux`、`core`、`starter-nacos`，严禁引入 `starter-web`。 |
