# ZHKJ Framework 安全架构与全生命周期数据保护

> **文档定位**: 系统安全防护、全生命周期数据加解密体系、JJWT 0.12+ 认证与 SSO 单点登录全景架构指南

---

## 🛡️ 1. 数据安全全生命周期三层防护模型

框架在业界首创并落地了**全生命周期三层数据安全防护模型**，将隐私数据从“配置文件 $\rightarrow$ 网络传输/前端展示 $\rightarrow$ 数据库持久存储与检索”实施闭环管控：

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
│    - 盲索引检索: 敏感字段配套生成 HMAC-SM3 加盐哈希盲索引辅助列 (*_hash)│
│      由 ZhkjMetaObjectHandler 在新增/更新时自动注入，支持数据库密文      │
│      精准检索与前后缀分词模糊检索，彻底杜绝明文脱库。                    │
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
     * 敏感字段：入库透明使用 SM4-CBC/AES 算法加密，读取透明还原为明文
     */
    @TableField(typeHandler = EncryptTypeHandler.class)
    @EncryptField(blindIndexField = "mobileHash")
    private String mobile;

    /**
     * 盲索引列：HMAC-SM3 加盐不可逆哈希，由 ZhkjMetaObjectHandler 自动注入，专用于 SQL 等值查询
     */
    @TableField("mobile_hash")
    private String mobileHash;
}
```

业务查询时仅需针对哈希辅助列进行等值检索：
```java
public Customer getByMobile(String mobile) {
    // 根据明文生成国密 HMAC-SM3 哈希
    String mobileHash = BlindIndexUtil.generateIndex(mobile);
    return customerMapper.selectOne(
        new LambdaQueryWrapper<Customer>()
            .eq(Customer::getMobileHash, mobileHash)
    );
}
```

---

## 🔑 3. 认证授权与统一单点登录 (SSO)

### 3.1 轻量无状态认证 (`starter-secure`)
- **现代化 JJWT 0.12+ 验签**:
  彻底废弃旧版 `setSigningKey`，统一采用流式语法验签：
  ```java
  SecretKey key = Keys.hmacShaKeyFor(signKey.getBytes(StandardCharsets.UTF_8));
  Claims claims = Jwts.parser()
          .verifyWith(key)
          .build()
          .parseSignedClaims(token)
          .getPayload();
  ```
- **方法级细粒度权限控制 (`@PreAuth`)**:
  ```java
  @RestController
  @RequestMapping("/api/v1/cases")
  public class CaseController {

      @PostMapping("/assign")
      @PreAuth("hasRole('admin') or hasPermission('case:assign')")
      public R<Void> assignCase(@RequestBody CaseAssignDto dto) {
          return R.success("案件分配成功");
      }
  }
  ```
- **运维端点自动化保护 (`ActuatorSecurityConfiguration`)**:
  当类路径中引入 Spring Security 时，自动对 `/actuator/prometheus`、`/actuator/health` 放行，其余敏感监控端点强制开启 Basic 认证保护。

---

### 3.2 统一单点登录客户端与多上下文自愈 (`starter-sso-client`)
1. **登录重定向与无感换票**:
   未登录请求自动拦截重定向至统一认证收银台；持有授权码（`code`）的请求无感调用认证服务端换取 Access Token。
2. **多上下文双向自愈规约**:
   `SsoClientFilter` 在构造 `SsoUserContext` 时，**必须同步组装并存入 `SecurityContextHolder.setUser(...)`**，并自动提取当前用户的租户 ID 桥接至 `TenantContext`，彻底杜绝多系统间上下文缺失。
3. **线程资源彻底释放**:
   在请求结束的 `finally` 阶段，通过最高优先级的 `FrameworkContextClearFilter` 兜底清理 `SsoContextHolder`、`SecurityContextHolder` 与 `TenantContext`，杜绝 Tomcat 线程复用串号污染。
