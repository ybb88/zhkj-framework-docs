# 应用引导启动模块：zhkj-framework-launcher

> **模块定位**: 统一微服务/单体应用引导启动器，提供环境自动探测、多 Profile 自愈装配、启动耗时统计与统一 ASCII Banner。  
> **适用基准**: OpenJDK 21 (LTS), Spring Boot 4.1.1, Spring Cloud 2025.1.2  
> **模块坐标**: `io.github.ybb88:zhkj-framework-launcher`

---

## 📋 1. 架构定位与职责

在企业微服务治理中，每个微服务应用如果在 `main` 方法中自行编写繁琐的启动逻辑，容易造成 Profile 配置不一致、环境识别错误或缺少统一运维诊断机制。

`zhkj-framework-launcher` 提供了统一启动契约：
- **`ZhkjApplication.run(...)` 门面**：替代原生的 `SpringApplication.run`，统一规范服务命名与启动时序；
- **多环境 Profile 智能探测**：根据部署环境变量（`SPRING_PROFILES_ACTIVE`、容器环境变量等）自动补齐并自愈缺省配置；
- **生命周期 SPI 扩展 (`LauncherService`)**：在 Spring 容器启动的极早期（Environment 准备阶段）执行各模块或基础设施的预配置；
- **运维诊断体验**：提供标准 ASCII Banner、JVM 运行参数打印、系统监听端口与启动耗时精准统计。

---

## 📦 2. 依赖坐标

```xml
<dependency>
    <groupId>io.github.ybb88</groupId>
    <artifactId>zhkj-framework-launcher</artifactId>
</dependency>
```

---

## 🧱 3. 核心包结构与类

```
io.github.ybb88.framework.launcher
├── launch/
│   ├── ZhkjApplication.java       # 统一引导启动器核心门面
│   ├── LauncherService.java       # 启动期生命周期回调 SPI 接口
│   └── LauncherConstant.java      # 引导器常量 (环境、默认端口等)
└── util/
    └── PropUtil.java              # 属性与配置文件读取工具
```

---

## 💡 4. 使用范式

### 4.1 标准微服务主类启动
在主类中直接通过 `ZhkjApplication.run` 启动：

```java
package io.github.ybb88.demo;

import io.github.ybb88.framework.launcher.launch.ZhkjApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class DemoApplication {

    public static void main(String[] args) {
        // 参数 1: 微服务或应用名称 (用于日志标识、Nacos 服务发现注册、Trace 追踪名)
        // 参数 2: 启动主类
        // 参数 3: 命令行启动参数
        ZhkjApplication.run("demo-service", DemoApplication.class, args);
    }
}
```

### 4.2 扩展启动期 SPI 回调 (`LauncherService`)
若某个基础 Starter 或企业定制包需要在 Spring Environment 初始化时注入动态配置或执行前置检测，只需实现 `LauncherService`：

```java
package io.github.ybb88.demo.launcher;

import io.github.ybb88.framework.launcher.launch.LauncherService;
import org.springframework.boot.builder.SpringApplicationBuilder;
import org.springframework.core.env.ConfigurableEnvironment;
import org.springframework.stereotype.Component;

import java.util.Properties;

@Component
public class CustomLauncherServiceImpl implements LauncherService {

    @Override
    public void launcher(SpringApplicationBuilder builder, String appName, String profile, boolean isLocalDev) {
        // 在此处为 SpringApplicationBuilder 设置默认属性或挂载自定义初始化器
        Properties props = new Properties();
        props.setProperty("custom.config.initialized", "true");
        builder.properties(props);
    }

    @Override
    public int getOrder() {
        // 定义回调执行顺序，值越小越早执行
        return 10;
    }
}
```

在 `META-INF/spring.factories` 或标准 SPI 中声明该实现即可。

---

## ⚙️ 5. 启动控制台输出示例

服务启动成功后，控制台将输出格式化运维概览：

```text
================================================================================
:: ZHKJ Framework :: (v2026.4.1.0-SNAPSHOT)
Application   : demo-service
Active Profile: dev
Server Port   : 8080
Started In    : 2.148 seconds
================================================================================
```
