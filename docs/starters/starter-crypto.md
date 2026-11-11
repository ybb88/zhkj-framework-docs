# 数据安全与国密加解密：zhkj-framework-starter-crypto

> **模块定位**: 数据安全 Starter，提供配置文件 `ENC(...)` 启动期解密、存储态字段加密（AES / SM4）与 HMAC 盲索引、HTTP 报文加解密所需的算法与配置。展示态脱敏 `@Desensitize` / `DesensitizeType` 位于 `zhkj-framework-core`（`io.github.ybb88.framework.desensitize`），不在本模块内；报文加解密的 Controller 切面位于 `zhkj-framework-starter-web`。  
> **适用基准**: OpenJDK 21, Spring Boot 4.1.1, BouncyCastle 1.78.1（`bcprov-jdk18on` / `bcpkix-jdk18on`）  
> **模块坐标**: `io.github.ybb88:zhkj-framework-starter-crypto`

---

## 📋 1. 模块职责与三类能力

| 能力 | 入口 | 说明 |
|------|------|------|
| 配置态解密 | `EncryptablePropertyEnvironmentPostProcessor` | 启动期将配置中的 `ENC(...)` 值替换为明文 |
| 存储态加密 | `StorageCryptoEngine`、`@EncryptField`、`@BlindIndex` | 字段落库加密；盲索引供等值检索 |
| 报文加解密 | `@ApiEncrypt` / `@ApiDecrypt`、`ApiCryptoProcessor` | 算法执行与密钥解析；Controller 切面位于 `starter-web` |

盲索引通用工具 `BlindIndexUtil` 位于 `zhkj-framework-core`（`io.github.ybb88.framework.util.crypto.BlindIndexUtil`），本模块仅注册其 SM3 实现。

---

## 📦 2. 依赖坐标

```xml
<dependency>
    <groupId>io.github.ybb88</groupId>
    <artifactId>zhkj-framework-starter-crypto</artifactId>
</dependency>
```

---

## 🧱 3. 核心包结构与组件

```
io.github.ybb88.framework.crypto
├── property/                                   # 配置态解密
│   ├── EncryptablePropertyEnvironmentPostProcessor.java  # EnvironmentPostProcessor
│   ├── EncryptablePropertySource.java          # 属性源包装（非 Map 类型）
│   ├── EncryptableMapPropertySource.java       # 属性源包装（MapPropertySource）
│   ├── ZhkjPropertyEncryptor.java              # AES 解密执行器（密钥由 SHA-256 派生）
│   ├── ZhkjPropertyCryptoProperties.java       # zhkj.crypto.property.*
│   └── ZhkjCryptoCli.java                      # 命令行加解密工具
├── storage/                                    # 存储态加密与盲索引
│   ├── annotation/
│   │   ├── EncryptField.java                   # 标记加密字段
│   │   └── BlindIndex.java                     # 标记盲索引目标/来源字段
│   ├── config/ZhkjStorageCryptoAutoConfiguration.java
│   ├── props/ZhkjStorageCryptoProperties.java  # zhkj.storage.crypto.*
│   ├── engine/StorageCryptoEngine.java         # 加解密主引擎（静态）
│   ├── spi/
│   │   ├── CryptoAlgorithmProvider.java        # 存储算法 SPI
│   │   ├── AesCryptoAlgorithmProvider.java     # AES 实现
│   │   ├── Sm4CryptoAlgorithmProvider.java     # SM4 实现
│   │   └── Sm3BlindIndexProvider.java          # HMAC-SM3 盲索引（BlindIndexProvider SPI）
│   └── util/
│       ├── BlindIndexBindingUtil.java          # 实体字段绑定关系解析与缓存
│       └── StorageBlindIndexUtil.java          # 盲索引门面（委托 BlindIndexUtil）
├── transport/                                  # 报文加解密（算法与密钥）
│   ├── annotation/ApiEncrypt.java, ApiDecrypt.java
│   ├── config/ZhkjTransportCryptoAutoConfiguration.java
│   ├── processor/ApiCryptoProcessor.java
│   └── properties/ApiCryptoProperties.java     # zhkj.api.crypto.*
├── shredding/                                  # 密码学粉碎（逻辑销毁）
│   ├── DataShreddingService.java
│   ├── DefaultDataShreddingService.java        # 进程内存实现
│   └── DataShreddedException.java
├── sm2/SM2Util.java, sm3/SM3Util.java, sm4/SM4Util.java
├── bean/CryptoInfoBean.java, enums/CryptoType.java   # CryptoType: DES, AES, SM4
├── exception/                                  # CryptoException 及报文相关异常
└── aot/CryptoRuntimeHints.java
```

自动装配（`META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`）：

- `ZhkjStorageCryptoAutoConfiguration`：注册 `StorageCryptoEngine`、`DataShreddingService`、`Sm3BlindIndexProvider`（均 `@ConditionalOnMissingBean`），并将 `CryptoAlgorithmProvider` Bean 注册到引擎。
- `ZhkjTransportCryptoAutoConfiguration`：注册 `ApiCryptoProcessor`（`@ConditionalOnMissingBean`）。

`EnvironmentPostProcessor` 通过 `META-INF/spring/org.springframework.boot.EnvironmentPostProcessor.imports` 与 `META-INF/spring.factories` 注册。

---

## 🛡️ 4. 安全行为与约束

### 4.1 配置态
- 主密钥不内置。解析顺序：系统属性 `zhkj.crypto.property.password` → 环境变量 `ZHKJ_CRYPTO_PASSWORD` → 配置项 `zhkj.crypto.property.password`。
- 三个来源读取到的主密钥若可被 Base64 解码，则使用解码结果（`EncryptablePropertyEnvironmentPostProcessor.tryDecodeBase64`）。
- 未配置主密钥时，处理器输出 ERROR 日志并**不包装任何属性源**，`ENC(...)` 原样保留，启动不失败。
- 解密密钥为主密钥 SHA-256 摘要的前 32 个字符（AES-256）。解密失败抛出 `IllegalStateException`。

### 4.2 存储态
- `StorageCryptoEngine` 默认 `zhkj.storage.crypto.enabled=false`，此时写入明文、读取原样返回。启用后加密失败抛出 `CryptoException`，由调用方事务回滚，引擎不回退为明文写入。
- 写入格式为 `<version>:<算法输出>`（默认 `v1:`）；算法为 SM4 时前缀为 `sm4:<version>:`。AES 算法输出为 `cbc$` 开头的动态 IV 密文（`AesCryptoAlgorithmProvider`）。
- 读取时同时识别 `sm4:` 前缀、当前版本前缀及历史 `v1:` 前缀；无前缀数据原样返回。
- 密钥来源：`zhkj.storage.crypto.key` → 回退 `zhkj.crypto.master-key` → 均未配置则抛出 `CryptoException`。长度为 32 直接使用，否则取 SHA-256 前 32 字符。
- 密码学粉碎：读取时若 `FrameworkContext` 中的 `dimensionId` 或 `tenantId` 已被 `DataShreddingService` 标记，抛出 `DataShreddedException`（仅 `decrypt` 路径检查）。默认实现为进程内存集合，多实例部署需自行提供 Bean。

**已知限制（代码现状）**: 当 `algorithm=SM4` 且 SM4 不可用时，`StorageCryptoEngine.encrypt` 以 AES 算法加密，但仍使用 `sm4:<version>:` 前缀写入；读取时 `decryptPayload` 因 SM4 不可用直接返回去除前缀后的密文载荷（不解密）。详见 `StorageCryptoEngine.java` 第 266–281 行与 314–320 行。

### 4.3 盲索引
- 计算入口：`io.github.ybb88.framework.util.crypto.BlindIndexUtil.generateIndex(String)`（读取 `zhkj.storage.crypto.blind-index-salt`）与 `generateIndex(String, String salt)`。
- 实现链（`BlindIndexUtil.generateIndex(String, String)`）：优先使用 `sm3Provider`（本模块的 `Sm3BlindIndexProvider`，HMAC-SM3 结果取前 32 个十六进制字符）；其计算失败时回退到 `PROVIDERS` 列表首个提供者；仍失败则使用 `HmacSM3`（JCE 或 BouncyCastle 可用时）否则 `HmacSHA256`，结果取前 32 个十六进制字符。
- 盐值未配置时抛出 `IllegalStateException`，不使用内置默认盐。
- 仅支持确定性等值匹配（`EQ` / `IN`），不得用于 `LIKE`。

### 4.4 盲索引字段的自动推导（`BlindIndexBindingUtil`）
按以下顺序建立 来源字段 → 盲索引字段 映射：

1. `@BlindIndex(source = "x")` 标注在目标字段上：来源为 `x`（须存在）。`source` 为空且目标字段名以 `Hash` 结尾时，依次尝试 `基名` 与 `基名Cipher`。
2. `@EncryptField(blindIndexField = "h")` 标注在来源字段上：目标为 `h`（须存在）。
3. 未映射的加密字段（判定规则见下段）：若存在 `<字段名>Hash` 则映射；字段名以 `Cipher` 结尾时尝试 `<基名>Hash`。

加密字段判定（`BlindIndexBindingUtil`）：标注 `@EncryptField`；或 `@TableField(typeHandler = …)` 的处理器为 `EncryptTypeHandler`（或其子类，位于 `starter-mybatis`）；或 JPA `@Convert(converter = …)` 的转换器为 `CryptoAttributeConverter`（`starter-jpa`）。普通 `@TableField` 字段不被视为加密字段；`exist` 属性不参与判定。

---

## ⚙️ 5. 配置项

### 5.1 配置态解密 `zhkj.crypto.property.*`

| 键 | 类型 | 默认值 | 说明 |
|----|------|--------|------|
| `enabled` | boolean | `true` | `false` 时不包装属性源 |
| `password` | String | 无 | 主密钥；推荐通过环境变量 `ZHKJ_CRYPTO_PASSWORD` 注入 |
| `prefix` | String | `ENC(` | 密文前缀 |
| `suffix` | String | `)` | 密文后缀 |
| `cache-enabled` | boolean | `true` | 解密结果内存缓存 |
| `algorithm` | String | `AES` | 已绑定，当前实现不读取 |

### 5.2 存储态加密 `zhkj.storage.crypto.*`

| 键 | 类型 | 默认值 | 说明 |
|----|------|--------|------|
| `enabled` | boolean | `false` | 全局存储加密开关 |
| `write-enabled` | boolean | `true` | `false` 时新写入为明文，历史密文仍可读取 |
| `algorithm` | String | `AES` | `AES` 或 `SM4`（大小写不敏感） |
| `key` | String | 无 | 存储密钥；未配置回退 `zhkj.crypto.master-key` |
| `version` | String | `v1` | 密文版本前缀 |
| `blind-index-salt` | String | 无 | 盲索引盐值，必须配置 |

### 5.3 报文加解密 `zhkj.api.crypto.*`

| 键 | 类型 | 默认值 | 说明 |
|----|------|--------|------|
| `enabled` | Boolean | `true` | 是否开启 |
| `param-name` | String | `data` | URL 参数密文名 |
| `aes-key` | String | 无 | AES 全局密钥，须 32 字节（AES-256） |
| `des-key` | String | 无 | DES 全局密钥 |
| `sm4-key` | String | 无 | SM4 全局密钥；字节长度不为 16 时经 SM3 派生前 16 字节（`SM4Util.deriveKey16`） |

注解 `@ApiEncrypt` / `@ApiDecrypt`：`value` 为 `CryptoType`（默认 `AES`），`secretKey` 为空时使用全局配置；全局配置也缺失时抛出 `KeyNotConfiguredException`。注解可标注在类或方法上。

### 5.4 示例（仅占位，密钥须由环境注入）

```yaml
spring:
  datasource:
    password: "ENC(<密文>)"

zhkj:
  crypto:
    property:
      password: "${ZHKJ_CRYPTO_PASSWORD}"
  storage:
    crypto:
      enabled: true
      algorithm: AES
      key: "${ZHKJ_STORAGE_CRYPTO_KEY}"
      blind-index-salt: "${ZHKJ_STORAGE_CRYPTO_BLIND_INDEX_SALT}"
  api:
    crypto:
      aes-key: "${ZHKJ_API_AES_KEY}"
```

`zhkj.crypto.master-key` 不在 `ZhkjStorageCryptoProperties` 中绑定，仅由 `StorageCryptoEngine.resolveKey` 通过 `SpringUtil.getProperty` 读取，作为 `zhkj.storage.crypto.key` 的回退。

### 5.5 命令行生成密文

```bash
java -cp zhkj-framework-starter-crypto.jar io.github.ybb88.framework.crypto.property.ZhkjCryptoCli -e "<明文>" -p "<主密钥>"
java -cp zhkj-framework-starter-crypto.jar io.github.ybb88.framework.crypto.property.ZhkjCryptoCli -d "ENC(<密文>)" -p "<主密钥>"
```

`ZhkjCryptoCli` 直接使用 `-p` 原值派生密钥，不执行运行期的 Base64 解码。若主密钥本身是合法 Base64 字符串，运行期与 CLI 的派生结果不一致（见第 4.1 节）。

---

## 💡 6. 典型使用范式

### 6.1 实体加密字段与盲索引

```java
package io.github.ybb88.demo.entity;

import com.baomidou.mybatisplus.annotation.TableField;
import com.baomidou.mybatisplus.annotation.TableName;
import io.github.ybb88.framework.crypto.storage.annotation.BlindIndex;
import io.github.ybb88.framework.crypto.storage.annotation.EncryptField;
import io.github.ybb88.framework.desensitize.DesensitizeType;
import io.github.ybb88.framework.desensitize.annotation.Desensitize;
import io.github.ybb88.framework.mybatis.base.MybatisTenantEntity;
import io.github.ybb88.framework.mybatis.crypto.handler.EncryptTypeHandler;
import lombok.Data;
import lombok.EqualsAndHashCode;

@Data
@EqualsAndHashCode(callSuper = true)
@TableName(value = "biz_customer", autoResultMap = true)
public class CustomerEntity extends MybatisTenantEntity {

    private String customerName;

    /** 存储加密（EncryptTypeHandler）；盲索引自动写入 idCardHash */
    @Desensitize(rule = DesensitizeType.ID_CARD)
    @EncryptField(blindIndexField = "idCardHash")
    @TableField(typeHandler = EncryptTypeHandler.class)
    private String idCard;

    /** 盲索引辅助列（HMAC-SM3） */
    @BlindIndex(source = "idCard")
    private String idCardHash;
}
```

`EncryptTypeHandler` 位于 `starter-mybatis`（`io.github.ybb88.framework.mybatis.crypto.handler`）。`@Desensitize` 与 `DesensitizeType` 位于 `zhkj-framework-core`（`io.github.ybb88.framework.desensitize`），本模块不提供。

### 6.2 等值检索

```java
package io.github.ybb88.demo.service;

import com.baomidou.mybatisplus.core.toolkit.Wrappers;
import io.github.ybb88.demo.entity.CustomerEntity;
import io.github.ybb88.demo.mapper.CustomerMapper;
import io.github.ybb88.framework.util.crypto.BlindIndexUtil;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;

import java.util.List;

@Service
@RequiredArgsConstructor
public class CustomerService {

    private final CustomerMapper customerMapper;

    public List<CustomerEntity> findByIdCard(String plainIdCard) {
        String hash = BlindIndexUtil.generateIndex(plainIdCard);
        return customerMapper.selectList(
            Wrappers.<CustomerEntity>lambdaQuery().eq(CustomerEntity::getIdCardHash, hash)
        );
    }
}
```

调用前必须已配置 `zhkj.storage.crypto.blind-index-salt`。

### 6.3 报文加解密注解

```java
@RestController
@RequestMapping("/api/v1/trade")
public class TradeController {

    @PostMapping("/pay")
    @ApiDecrypt(CryptoType.AES)
    @ApiEncrypt(CryptoType.AES)
    public R<OrderVo> createOrder(@RequestBody OrderCreateDto dto) {
        return R.data(tradeService.processPayment(dto));
    }
}
```

请求解密与响应加密由 `starter-web`（`io.github.ybb88.framework.web.crypto.core`）中的 `ApiDecryptRequestBodyAdvice` 与 `ApiEncryptResponseBodyAdvice` 完成。

---

## 📎 7. 相关文档

- `zhkj-framework/zhkj-framework-starter-mybatis`：`EncryptTypeHandler`、`ZhkjMetaObjectHandler` 盲索引填充
- `zhkj-framework/zhkj-framework-starter-web`：报文加解密切面
