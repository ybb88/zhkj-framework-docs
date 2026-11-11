# ZHKJ Framework 安全架构与全生命周期数据保护

> **文档定位**: 系统安全防护、全生命周期数据加解密体系、JWT 认证、统一 SSO 单点登录客户端与分布式锁防误删安全指南  
> **设计与创作者**: Jason.Yang（274107098@qq.com）  

---

## 🛡️ 1. 数据安全全生命周期三层防护模型

框架将隐私数据从“配置文件 $\rightarrow$ 网络传输/前端展示 $\rightarrow$ 数据库持久存储与检索”纳入三层管控：

```
┌─────────────────────────────────────────────────────────────────────────┐
│ 1. 配置态 (Configuration Layer)                                         │
│    本地 application.yml 或 Nacos 配置中心中数据库密码等敏感属性通过      │
│    ENC(...) 加密存储；由 EncryptablePropertyEnvironmentPostProcessor     │
│    在环境准备阶段透明解密，严禁明文凭据入库或入 Git。                    │
└────────────────────────────────────┬────────────────────────────────────┘
                                     │
                                     ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ 2. 传输与展示态 (Transmission & Presentation Layer)                     │
│    - 展示态: Jackson 序列化输出时，通过 @Desensitize 对字段做掩码脱敏    │
│      （共 7 种内置规则，含 CUSTOM）；                                   │
│    - 传输态: 通过 @ApiEncrypt / @ApiDecrypt 对 Controller 报文做对称加解密│
└────────────────────────────────────┬────────────────────────────────────┘
                                     │
                                     ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ 3. 存储与检索态 (Persistence & Storage Layer)                           │
│    - 密文落盘: 实体敏感字段通过 EncryptTypeHandler 透明加密入库（SM4 或   │
│      AES），加密失败抛出 CryptoException 阻断持久化，不降级写明文；       │
│    - 盲索引检索: 敏感字段配套 HMAC 加盐哈希辅助列 (*_hash)，由             │
│      ZhkjMetaObjectHandler 在新增/更新时自动注入。                        │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## ⚙️ 2. 全生命周期数据安全实战指南

### 2.1 配置态：敏感属性透明解密 (`starter-crypto`)
在配置文件中写入 `ENC(...)` 密文：
```yaml
spring:
  datasource:
    url: jdbc:mysql://127.0.0.1:3306/zhkj_demo?useUnicode=true&characterEncoding=utf-8
    username: root
    password: "ENC(XyKx5jN5684AxytPGmOTAV7+XSDh/S83BwCb8aCdyGU=)"

zhkj:
  crypto:
    property:
      enabled: true
      password: "${ZHKJ_CRYPTO_PASSWORD}"  # 主密钥由环境变量 ZHKJ_CRYPTO_PASSWORD 注入，严禁明文入库
```

配套命令行密文生成工具（`ZhkjCryptoCli`，参数 `-e` 加密、`-d` 解密、`-p` 主密钥）：
```bash
java -cp zhkj-framework-starter-crypto.jar io.github.ybb88.framework.crypto.property.ZhkjCryptoCli -e "root123456" -p "YourStrongMasterKey"
# 输出为 ENC(...) 形式的密文
```

主密钥与属性前缀配置项：`zhkj.crypto.property.{enabled, password, algorithm, prefix, suffix, cache-enabled}`（`ZhkjPropertyCryptoProperties`）。

---

### 2.2 展示态：字段动态脱敏 (`@Desensitize`)
在返回前端的 VO/DTO 字段上声明 `@Desensitize`，框架在 Jackson 序列化时自动打码：
```java
@Data
public class CustomerVo {
    private Long id;

    @Desensitize(rule = DesensitizeType.CHINESE_NAME)
    private String name;        // 输出: 张*丰

    @Desensitize(rule = DesensitizeType.MOBILE_PHONE)
    private String mobile;      // 输出: 138****8000

    @Desensitize(rule = DesensitizeType.ID_CARD)
    private String idCard;      // 输出: 110101********2345

    @Desensitize(rule = DesensitizeType.BANK_CARD)
    private String bankCardNo;  // 输出: 622202************1234
}
```

内置规则（`DesensitizeType`，共 7 种）：`CHINESE_NAME`、`ID_CARD`、`MOBILE_PHONE`、`BANK_CARD`、`EMAIL`、`PASSWORD`、`CUSTOM`。

#### 2.2.1 内部微服务 RPC 脱敏旁路放行机制 (`InternalRpcDesensitizeFilter`)
> [!IMPORTANT]
> **默认脱敏与受控取数**：
> 普通 RPC 与公共展示端点默认执行 `@Desensitize`。确需明文字段的内部业务必须先完成调用方认证和目标端点授权；传输标记不能替代权限校验，也不能替代敏感操作所需的审批与审计。

`zhkj-framework-starter-web` 的 `InternalRpcDesensitizeFilter` 保持 `Ordered.HIGHEST_PRECEDENCE + 20`，授权契约来自 `framework-core` 的 `io.github.ybb88.framework.rpc.RpcRequestAttribute`：
1. **脱敏授权**：仅当当前 ServletRequest 的 `RpcRequestAttribute.DESENSITIZE_AUTHORIZED` 属性为 `Boolean.TRUE` 时，才在请求线程内调用 `DesensitizeContext.disable()`；请求头、字符串 `"true"` 或 ThreadLocal 均不能作为该授权的来源。
2. **认证先行**：宿主认证过滤器必须先于 `HIGHEST_PRECEDENCE + 20` 执行，在已保护的内部路径完成调用方认证与目标端点授权后，才设置该属性并进入下游过滤器。公共展示 API 不因机器凭据正确而获得明文权限；没有明文业务需求的内部端点不应赋权。
3. **请求内清理**：未授权请求显式启用脱敏；无论是否授权，脱敏过滤器均在 `finally` 恢复进入过滤器前的开关状态以支持嵌套调用，并非无条件调用 `enable()`。设置授权属性的宿主过滤器在 `finally` 删除属性；外层 `FrameworkContextClearFilter` 清理 `FrameworkContext`，不将授权保存到线程或异步快照。
4. **委托单独授权**：接收 `X-Delegated-Tenant-Id` 与 `X-Grant-Ticket` 另需当前请求的 `RpcRequestAttribute.TENANT_DELEGATION_AUTHORIZED == Boolean.TRUE`。宿主必须先认证并校验目标端点及租户委托权限，再受控设置并在 `finally` 删除；机器认证、脱敏授权与 `X-Internal-RPC` 均不等于跨租户授权。基于可信 JWT 声明建立的现有上下文仍按原逻辑处理。
5. **入口剥离**：网关在认证路径、匿名白名单及 `enabled=false` 分支均清理 `X-Internal-RPC`、`X-Delegated-Tenant-Id`、`X-Grant-Ticket`。`starter-http` 默认 provider 仍可生成 RPC 标记，但该标记不具备授权意义。

**安全行为变更与迁移**：旧版依赖 `X-Internal-RPC: true` 获取明文或接受委托头的调用不再获得相应权限。宿主应将确有业务需求的调用迁移到受认证和端点授权保护的内部路径，并在脱敏过滤器之前接入服务端请求属性契约；不得把入站 Header 值复制成授权属性，不得为公共展示端点统一赋权，也不得引入共享万能密钥或自制签名。未接入服务端授权的调用保持默认脱敏、忽略未经授权的委托头。

以上为代码契约，不代表已验证部署入口隔离、服务直连限制或全部宿主授权策略；上线前仍需按实际部署核对这些边界。

---

### 2.3 存储态：国密密文落盘与盲索引自动注入 (`starter-mybatis` + `starter-crypto`)
持久层不在业务 Service 中手工计算盲索引哈希，改由实体注解声明，`ZhkjMetaObjectHandler` 在新增与更新时自动注入：

```java
@Data
@EqualsAndHashCode(callSuper = true)
@TableName(value = "sys_customer", autoResultMap = true)
public class Customer extends MybatisTenantEntity {

    private String customerName;

    /**
     * 手机号敏感字段（存储加密，SM4 或 AES，由 zhkj.storage.crypto.algorithm 决定）
     */
    @EncryptField(blindIndexField = "mobileHash")
    @TableField(typeHandler = EncryptTypeHandler.class)
    private String mobile;

    /**
     * 手机号盲索引哈希列（HMAC 加盐哈希）
     */
    @BlindIndex(source = "mobile")
    private String mobileHash;
}
```

存储加密相关配置（`ZhkjStorageCryptoProperties`，前缀 `zhkj.storage.crypto`）：`enabled`（默认 `false`）、`write-enabled`、`algorithm`（默认 `AES`）、`key`、`version`（默认 `v1`）、`blind-index-salt`。

**业务检索实战**：
```java
// 业务层根据明文查询时，仅需计算一次盲索引哈希后等值匹配辅助列：
String queryMobile = "13800138000";
String queryHash = BlindIndexUtil.generateIndex(queryMobile);

List<Customer> list = customerMapper.selectList(
    Wrappers.<Customer>lambdaQuery().eq(Customer::getMobileHash, queryHash)
);
```

#### 2.3.1 加解密安全阻断
> [!CAUTION]
> `StorageCryptoEngine` 在加密失败时抛出 `CryptoException`，中断持久化。未配置 `zhkj.storage.crypto.key` 时同样抛出 `CryptoException`（`StorageCryptoEngine.java` 第 219 行附近），不使用内置默认密钥。

#### 2.3.2 盲索引哈希与异常阻断
> [!IMPORTANT]
> 1. **算法探测**：`BlindIndexUtil` 通过 `resolveHmacAlgorithm()` 探测运行环境。存在 BouncyCastle（`BC`）Provider 且可用 `HmacSM3` 时使用 `HmacSM3`，否则降级为 `HmacSHA256`。
> 2. **盐值强制配置**：未配置 `zhkj.storage.crypto.blind-index-salt` 时抛出 `IllegalStateException`，拒绝使用内置默认盐值。
> 3. **计算异常阻断**：哈希计算异常时抛出 `IllegalStateException` 阻断写入，不写入缺省值。
> 4. **检索语法约束**：盲索引列仅支持精确等值匹配（`EQ` 或 `IN`），不支持 `LIKE` 模糊检索。
>
> **存量待治理**：`BlindIndexUtil` 的 SPI 提供者（`ZhkjBlindIndex` 相关分支）计算失败时会记录告警并回退到默认算法（`BlindIndexUtil.java` 第 119、126、209 行）。该回退可能导致同一明文在不同算法下产生不一致的哈希，与上述第 3 条约束不一致，需确认后治理。

### 2.4 JPA 租户读写隔离 (`starter-jpa` + `starter-tenant`)

- `JpaTenantEntity` 直接声明 Hibernate 自动过滤器，覆盖查询及触发数据库读取的主键加载；`TenantEntityListener` 拒绝跨租户新增、更新和实体归属改写，不以告警日志代替阻断。
- `JpaTenantIdResolver` 只调用 `TenantRuleProvider.resolveTenantId()`，不读取请求头、不在 Session 中缓存租户参数；受控委托须由上游按第 2.2.1 节授权后建立上下文。
- 启用隔离时，provider 返回 `null` 或空白即拒绝操作；默认 provider 无上下文时返回租户 `0`，仍执行等值过滤，不开放全表。只有启动配置 `zhkj.tenant.enabled=false` 可关闭该隔离分支。
- 无需 MVC 或 OSIV；请求上下文仍由 `FrameworkContext` 清理，Session 由原事务或 OSIV 生命周期管理，不改变 `spring.jpa.open-in-view=false`。
- 不跨租户复用带已加载实体的 EntityManager，不调用 `disableFilter` 或显式覆盖租户参数。一级缓存命中不会重新执行 SQL；原生 SQL 与绕过生命周期的批量 DML 需显式限定租户，不能据此宣称全部 SQL 都被自动防护。详见 [JPA 组件](../starters/starter-jpa.md#8-session-生命周期与租户边界)。

---

## 🔑 3. 统一 SSO 单点登录与安全上下文 (`starter-sso-client` / `starter-secure`)

### 3.1 核心防护机制
1. **URI 安全清洗防绕过**：`SsoClientFilter`（第 103 行）、`SsoGatewayGlobalFilter`（第 197 行）及 `XssFilter`（第 40 行）在路径匹配前调用 Spring 的 `StringUtils.cleanPath(...)` 规范化路径，抵御 `//admin`、分号矩阵参数等绕过手法。
2. **用户模型归一化**：`SsoUserContext` 继承 `ZhkjUser`；`SsoClientFilter` 解析 Token 后调用 `SecurityContextHolder.setUser(...)`（第 172 行）完成装配。
3. **Token 传递方式**：认证仅接受请求头（`TokenCredentialUtil`，见 `starter-sso-client` 文档）。Cookie 名称常量 `zhkj_sso_token`、`zhkj_sso_session`（`HeaderConstant.COOKIE_SSO_TOKEN`、`COOKIE_SSO_SESSION`）已定义，但当前认证代码未读取 Cookie。Header 常量包括 `Authorization`、`Zhkj-Authorization`、`zhkj-auth`（`HeaderConstant`）。`SsoAuthClient` 的 `TokenRequestDTO.grantType` 支持 `refresh_token` 换取新 Token。
4. **认证服务地址默认值**：`zhkj.sso.client.auth-server-url` 默认 `http://127.0.0.1:8088`（`SsoClientProperties`），生产环境需显式覆盖。

### 3.2 安全上下文的生命周期 (`SecurityContextHolder`)
- **存储基石**：`SecurityContextHolder` 基于标准 `ThreadLocal<ZhkjUser>`（`secure/context/SecurityContextHolder.java`），不使用 `InheritableThreadLocal`。
- **自动清理**：类静态块中调用 `FrameworkContext.registerCleaner(SecurityContextHolder::clear)`，由 `FrameworkContext.clearAll()` 统一清理。
- **跨线程透传**：异步线程与批处理使用 `FrameworkContext.capture()` / `FrameworkContext.openScope(snapshot)`，离开 `try` 作用域后自动清理。Kafka 生产端由 `KafkaContextProducerInterceptor` 将快照写入消息 Header，消费端由 `KafkaTraceRecordInterceptor` 还原并在 `afterRecord` 中清理。

---

## 🔒 4. 高可靠分布式锁防御 (`starter-redis`)

分布式锁的安全性不仅在于“能否获取”，还在于“能否安全释放”：
1. **持有者校验防误删**：释放锁通过 Lua 脚本完成，仅当锁中存储的唯一标识与当前持有者一致时才执行删除（`DistributedLockScripts`），防止任务耗时超过租期后误删他人持有的锁。
2. **注解参数**：`@DistributedLock` 的 `key`（必填，支持 SpEL）、`prefix`、`waitTime`（默认 `0`）、`leaseTime`（默认 `30`）、`timeUnit`（默认 `SECONDS`）、`failureStrategy`、`message`、`fallbackMethod`。
3. **失败处理**：加锁失败抛出 `DistributedLockException`，该类继承 `BaseFrameworkException`，由 `GlobalExceptionHandler` 统一处理（响应码以 handler 实现为准）。
