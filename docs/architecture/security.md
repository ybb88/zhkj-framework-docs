# ZHKJ Framework 安全架构与全生命周期数据保护

> **文档定位**: 系统安全防护、全生命周期数据加解密体系、JJWT 0.12+ 认证、统一 SSO 单点登录客户端与分布式锁防误删安全指南  
> **设计与创作者**: Jason.Yang（274107098@qq.com）  

---

## 🛡️ 1. 数据安全全生命周期三层防护模型

框架在业界首创并落地了**全生命周期三层数据安全防护模型**，将隐私数据从“配置文件 $\rightarrow$ 网络传输/前端展示 $\rightarrow$ 数据库持久存储与检索”实施全流程闭环管控：

```
┌─────────────────────────────────────────────────────────────────────────┐
│ 1. 配置态 (Configuration Layer)                                         │
│    本地 application.yml 或 Nacos 配置中心中数据库密码等敏感属性通过      │
│    ENC(...) 加密存储；由 Spring Boot 4 原生 EnvironmentPostProcessor     │
│    在容器环境装配阶段透明解密，严禁明文凭据入库或入 Git。                │
└────────────────────────────────────┬────────────────────────────────────┘
                                     │
                                     ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ 2. 传输与展示态 (Transmission & Presentation Layer)                     │
│    - 展示态: Jackson 序列化输出时，通过 @Desensitize 注解实现手机号、    │
│      身份证、姓名、银行卡等 10 种规则的动态掩码脱敏；                   │
│    - 传输态: 通过 @ApiEncrypt / @ApiDecrypt 实现 Controller 接口报文     │
│      端到端全对称密文交互 (AES / DES)。                                 │
└────────────────────────────────────┬────────────────────────────────────┘
                                     │
                                     ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ 3. 存储与检索态 (Persistence & Storage Layer)                           │
│    - 密文落盘: 实体敏感字段通过 EncryptTypeHandler (SM4/AES) 透明加密入库│
│      具备事务安全阻断机制，加密失败严禁降级明文，坚决抛出 CryptoException│
│    - 盲索引检索: 敏感字段配套生成 HMAC-SM3 加盐哈希盲索引辅助列 (*_hash)│
│      由 ZhkjMetaObjectHandler 在新增/更新时自动注入，强一致散列防漂移。   │
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
      password: "YourStrongMasterKey" # 生产环境推荐通过环境变量 ZHKJ_CRYPTO_PROPERTY_PASSWORD 注入
```

配套命令行密文生成小工具：
```bash
java -cp zhkj-framework-starter-crypto.jar io.github.ybb88.framework.crypto.property.ZhkjCryptoCli -e "root123456" -p "YourStrongMasterKey"
# 输出: ENC(XyKx5jN5684AxytPGmOTAV7+XSDh/S83BwCb8aCdyGU=)
```

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

---

### 2.3 存储态：国密密文落盘与盲索引自动注入 (`starter-mybatis`)
持久层严禁在业务 Service 中散落手工计算盲索引哈希。通过在实体类上采用命名约定或注解声明，由框架自动透明完成计算：

```java
@Data
@EqualsAndHashCode(callSuper = true)
@TableName(value = "sys_customer", autoResultMap = true)
public class Customer extends MybatisTenantEntity {

    private String customerName;

    /**
     * 手机号敏感字段 (国密 SM4 存储加密)
     */
    @EncryptField(blindIndexField = "mobileHash")
    @TableField(typeHandler = EncryptTypeHandler.class)
    private String mobile;

    /**
     * 手机号盲索引哈希列 (HMAC-SM3 国密加盐哈希)
     */
    @BlindIndex(source = "mobile")
    private String mobileHash;
}
```

**业务检索实战**：
```java
// 业务层根据明文查询时，仅需计算一次盲索引哈希后等值匹配辅助列：
String queryMobile = "13800138000";
String queryHash = BlindIndexUtil.generateIndex(queryMobile);

List<Customer> list = customerMapper.selectList(
    Wrappers.<Customer>lambdaQuery().eq(Customer::getMobileHash, queryHash)
);
```

#### 2.3.1 加解密安全阻断与合规铁律
> [!CAUTION]
> **绝对禁止降级明文存储**：底层 `StorageCryptoEngine` 在执行敏感字段加密时具备硬性事务阻断机制。一旦发生任何加密异常（如密钥未就绪、算法提供者异常），坚决抛出 `CryptoException` 强制中断持久化事务并回滚，绝不向数据库降级写入任何明文数据，严防明文脱库合规事故。

#### 2.3.2 盲索引强一致性哈希与防哈希漂移
> [!IMPORTANT]
> **强一致散列与异常阻断**：
> 1. **动态探测与算法自愈**：盲索引哈希由 `BlindIndexUtil.generateIndex(plainText)` 驱动，启动期动态嗅探运行环境优先采用国密 `HmacSM3`（在无 BouncyCastle 依赖的轻量环境平滑降级为 `HmacSHA256`）。
> 2. **杜绝哈希漂移**：散列计算过程若遭遇不可恢复异常，直接抛出 `IllegalStateException` 阻断写入，绝不使用随机或缺省值回退，杜绝同一敏感数据在不同集群节点或时间点生成不一致散列串导致的历史数据检索失效与死数据。
> 3. **检索语法约束**：盲索引列仅支持确定性精确等值匹配（`EQ` 或 `IN`），严禁使用 `LIKE '%...%'` 进行模糊检索。

---

## 🔑 3. 统一 SSO 单点登录与安全上下文 (`starter-sso-client` / `starter-secure`)

### 3.1 核心防护机制
1. **URI 安全清洗防绕过**：
   `SsoClientFilter` 与 `SsoGatewayGlobalFilter` 在白名单匹配前，统一调用 `StringUtil.cleanPath(uri)` 进行规范化，有效防止利用多重斜杠（`//admin`）、分号矩阵参数（`/api;jsessionid=...`）或相对路径符号绕过鉴权。
2. **零反射用户模型归一化**：
   SSO 解析所得的 `SsoUserContext` 物理继承自框架核心 `ZhkjUser`，在过滤器层面直接装配到 `SecurityContextHolder` 与 `FrameworkContext`，杜绝任何反射开销与跨系统模型割裂。
3. **双 Token 静默续期与 Cookie/Header 自由切换**：
   开箱支持浏览器 Cookie 模式与微服务 Header 模式（`Zhkj-Auth` / `Authorization`），支持通过 RefreshToken 自动静默换取新 Token。

### 3.2 安全上下文透传与生命周期治理 (`SecurityContextHolder`)
> [!NOTE]
> **基于标准 ThreadLocal 纳管生命周期**：
> - **存储基石**：`SecurityContextHolder` 基于标准 `ThreadLocal<ZhkjUser>` 维护当前线程的用户身份，严禁使用原生 `InheritableThreadLocal` 避免线程池复用时的线程上下文泄漏与租户身份错乱。
> - **自动清理机制**：`SecurityContextHolder` 在静态块中自动注册到 `FrameworkContext` 清理链路（`FrameworkContext.registerCleaner(SecurityContextHolder::clear)`）。
> - **跨线程全生命周期透传**：在 Web 过滤器（`SsoClientFilter`）与消息监听拦截器（`KafkaTraceRecordInterceptor`）中，统一通过 `FrameworkContext.capture()` 获取快照，并在异步线程池、Spring Batch 批处理中通过 `FrameworkContext.openScope(snapshot)` / `runWith(...)` 建立安全作用域，执行完毕由 AutoCloseable 机制彻底清理，从根本上杜绝跨线程池污染。

---

## 🔒 4. 高可靠分布式锁防御 (`starter-redis`)

在分布式并发场景下，锁的安全性不仅在于“能否获取锁”，更在于“能否安全释放锁”：
1. **持有者校验（Owner Check）防误删**：
   底层通过 Lua 脚本执行释放：只有锁中存储的唯一 Value 与当前线程持有的锁标识完全一致时才执行 `DEL`，彻底防止因任务执行耗时超过租期时误释放其他节点的锁。
2. **强类型异常体系与优雅降级**：
   加锁失败抛出的 `DistributedLockException` 严格继承自 `BaseFrameworkException`，由全局异常处理器拦截转换为受控的 `400 Bad Request / 业务失败响应`，绝不向前端暴露未受控的 500 系统崩溃。
