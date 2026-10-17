# 核心基础模块：zhkj-framework-core

> **模块定位**: 框架通用基础核心模块，提供纯纯 Java 工具箱、统一 API 响应协议模型、通用领域基类与对象拷贝设施。  
> **适用基准**: OpenJDK 21 (LTS), Spring Boot 4.1.1, Jackson 3.x  
> **模块坐标**: `io.github.ybb88:zhkj-framework-core`

---

## 📋 1. 架构定位与职责

`zhkj-framework-core` 是整个框架依赖拓扑树的**最底层基石**，坚持以下架构底线：

1. **绝对纯粹性**：仅依赖 JDK 21、Lombok、Jackson 3 及基础加密库，**严禁反向引入任何 WebMVC、Servlet、MyBatis 或具体中间件实现**；
2. **高复用性**：收敛 100+ 高性能工业级单数工具类（`*Util`），彻底杜绝业务层重复造轮子；
3. **协议一致性**：统一定义 API 响应包装 `R<T>`、错误码规范 `IResultCode`、通用领域基类 `BaseDto`/`BaseVo` 与多维度抽象；
4. **统一上下文生命周期门面**：提供 `FrameworkContext` 作为全生命周期上下文总入口，统一纳管租户、安全凭证与操作人身份快照。

```
              ┌────────────────────────┐
              │ zhkj-framework-core    │
              │ - API 统一协议 R<T>     │
              │ - FrameworkContext 门面 │
              │ - 核心单数工具类 (*Util) │
              │ - 通用领域基类 DTO/VO   │
              └───────────┬────────────┘
                          │ (单向依赖基石)
                          ▼
              ┌────────────────────────┐
              │ 上层 Starters / 业务服务│
              └────────────────────────┘
```

---

## 📦 2. 依赖坐标

在 Maven 项目中引入：

```xml
<dependency>
    <groupId>io.github.ybb88</groupId>
    <artifactId>zhkj-framework-core</artifactId>
</dependency>
```

---

## 🧱 3. 核心包结构与组件矩阵

```
io.github.ybb88.framework
├── api/                     # API 统一交互协议
│   ├── R.java               # 统一响应载体 (code, success, data, msg)
│   ├── IResultCode.java     # 业务错误码契约接口
│   ├── ResultCode.java      # 系统通用状态码枚举 (SUCCESS, FAILURE, 401, 403, 500...)
│   └── PageResult.java      # 框架中立的通用分页出参模型
├── constant/                # 核心基础常量 (public final class 封装)
│   ├── AppConstant.java     # 应用通用常量 (Profile, 包路径, 环境标识)
│   ├── HeaderConstant.java  # HTTP 协议请求头规约常量 (X-Tenant-Id, Authorization, X-Trace-Id)
│   ├── TraceConstant.java   # 链路追踪常量
│   ├── CryptoConstant.java  # 加密算法与存储前缀常量 (SM4, AES, v1:)
│   └── CommonConstant.java  # 通用系统常量
├── context/                 # 全生命周期统一上下文门面
│   ├── FrameworkContext.java# 统一快照采集 (capture) 与跨线程传递 (runWith / openScope)
│   └── ContextScope.java    # AutoCloseable 资源作用域门面
├── desensitize/             # 敏感数据动态脱敏规范
│   ├── annotation/          # @Desensitize 注解
│   ├── handler/             # 手机号、身份证、银行卡等 10 种脱敏规则处理器
│   └── json/                # Jackson 脱敏序列化器
├── model/                   # 通用领域模型与基类
│   ├── BaseDto.java         # 数据传输对象基类
│   ├── BaseVo.java          # 视图对象基类
│   ├── BasePage.java        # 分页查询请求基类
│   ├── Kv.java              # 链式键值对模型 (继承 Map)
│   └── INode.java           # 树形节点接口
├── support/                 # 核心支持组件
│   └── BaseBeanCopier.java  # 高性能字节码级 Bean 对象属性复制器
└── util/                    # 核心单数工具类库
    ├── JsonUtil.java        # Jackson JSON 序列化工具
    ├── uuid/IdUtil.java     # 规范 UUID 与唯一 ID 生成工具
    ├── DigestUtil.java      # MD5、SHA256、SM3 摘要计算工具 (支持 byte[] 字节流)
    ├── SpringUtil.java      # Spring 上下文静态获取 Bean 工具
    ├── StringUtil.java      # 高性能字符串与安全清洗工具
    ├── ObjectUtil.java      # 对象判空、对比工具
    ├── CollectionUtil.java  # 集合转换、拆分、笛卡尔积工具
    ├── DateUtil.java        # Java 8 时间与经典 Date 转换工具
    ├── ClassUtil.java       # 反射、类加载与注解提取工具
    ├── BeanUtil.java        # 属性复制与 Map 互转
    ├── FileUtil.java        # 文件与路径高效读写
    └── crypto/              # 安全与加解密 (SM4, AES, MD5, SHA, BlindIndexUtil)
```

---

## 🛡️ 4. 架构级防护与编码规约

### 4.1 工具类单数命名与密封防实例化规约 (Sealed Utility Defense)
- **单数法则**：所有基础工具类严格命名为单数（`JsonUtil`、`DateUtil`、`StringUtil`、`IdUtil`、`BeanUtil`、`DigestUtil`、`PropUtil`），**严禁出现任何 `*Utils` 复数形式**。
- **密封防御**：所有工具类强制声明为 **`public final class`**，并显式声明私有构造器：
  ```java
  public final class DateUtil {
      private DateUtil() {} // 防实例化
  }
  ```
- **彻底根除常量接口反模式**：`StringPool` 与 `CharPool` 均为带有私有构造器的 `public final class`，严禁以 `implements StringPool` 反模式使用。

### 4.2 静态正则常驻预编译与 GC 防护
- 字符串工具（`StringUtil`）及关键字清洗方法内部，一律采用静态私有预编译常量：
  ```java
  private static final Pattern CLEAN_CHARS_PATTERN = Pattern.compile("[ 　`·•\\f\\t\\v\\s]");
  private static final Pattern SAFE_IDENTIFIER_PATTERN = Pattern.compile("[^a-zA-Z0-9_.]");
  ```
- 坚决杜绝在方法体内动态执行 `Pattern.compile(...)` 或滥用 `String.replaceAll(...)`，彻底消除高并发吞吐下的 Minor GC 内存抖动。

### 4.3 跨平台字符集绝对锁死 (`StandardCharsets.UTF_8`)
- 涉及字节数组与字符串转换、网络传输、加解密计算时，必须显式指定 `StandardCharsets.UTF_8`，杜绝在 Windows/Linux 跨平台部署时因操作系统默认字符集不同导致乱码或签名验证失败。

---

## 💡 5. 核心 API 最佳实践与使用范式

### 5.1 统一 API 响应包装 (`R<T>`)
```java
package io.github.ybb88.demo.controller;

import io.github.ybb88.framework.api.R;
import io.github.ybb88.framework.api.ResultCode;
import io.github.ybb88.demo.vo.UserVo;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class UserController {

    @GetMapping("/api/users/{id}")
    public R<UserVo> getUser(@PathVariable Long id) {
        UserVo vo = new UserVo();
        vo.setId(id);
        vo.setUsername("alex");
        // 成功响应带数据
        return R.data(vo);
    }

    @GetMapping("/api/users/check")
    public R<Void> checkStatus() {
        // 失败响应带状态码
        return R.fail(ResultCode.UN_AUTHORIZED);
    }
}
```

### 5.2 全局统一上下文门面 (`FrameworkContext`)
在多线程、异步任务（`@Async`）或消息消费中，严禁依赖原生 `InheritableThreadLocal`。统一采用快照捕获与作用域传递：

```java
// 1. 在主线程抓取上下文快照 (包含租户、安全身份、操作人信息)
FrameworkContext.Snapshot snapshot = FrameworkContext.capture();

// 2. 传递给异步线程池受控执行
executor.execute(() -> {
    try (ContextScope scope = FrameworkContext.openScope(snapshot)) {
        // 在作用域内，当前线程上下文完整还原
        bizService.processData();
    } // 退出作用域时，所有上下文自动受控彻底释放，杜绝常驻线程复用污染
});
```

### 5.3 核心单数工具类使用
```java
// JSON 序列化与反序列化 (Jackson 3 原生)
String json = JsonUtil.toJson(user);
UserVo userVo = JsonUtil.parse(json, UserVo.class);

// 唯一 ID 生成
String uuid = IdUtil.simpleUuid();
long snowflakeId = IdUtil.nextId();

// 摘要计算 (支持 String 及 byte[] 字节流)
String md5Hex = DigestUtil.md5Hex("rawPassword");
String sha256 = DigestUtil.sha256Hex("rawContent");
byte[] fileHash = DigestUtil.sha256(byteBuffer);

// 对象高效拷贝 (基于字节码 BeanCopier 缓存)
UserVo targetVo = BeanUtil.copy(sourceEntity, UserVo.class);
```
