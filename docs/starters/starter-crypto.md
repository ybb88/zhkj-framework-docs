# 数据安全与国密加解密：zhkj-framework-starter-crypto

> **模块定位**: 全生命周期数据安全中心 Starter，提供“配置态透明解密 (`ENC(...)`) + 展示态动态脱敏 (`@Desensitize`) + 存储态国密透明加密与 HMAC-SM3 盲索引强一致检索”三层一体化合规防护。  
> **适用基准**: OpenJDK 21 (LTS), Spring Boot 4.1.1, BouncyCastle 国密算法库  
> **模块坐标**: `io.github.ybb88:zhkj-framework-starter-crypto`

---

## 📋 1. 架构定位与三层防护模型

在现代金融与企业级分布式应用中，隐私保护与数据合规是生命线。`zhkj-framework-starter-crypto` 统一纳管所有加解密核心组件：

```
┌─────────────────────────────────────────────────────────────────────────┐
│ 1. 配置态 (Configuration Layer)                                         │
│    - 本地 YAML 或 Nacos 敏感参数使用 ENC(...) 加密；                      │
│    - 基于 Spring Boot 4 原生 EnvironmentPostProcessor 在启动期透明解密。 │
└────────────────────────────────────┬────────────────────────────────────┘
                                     │
                                     ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ 2. 传输与展示态 (Transmission & Presentation Layer)                     │
│    - 展示态: Jackson 序列化输出时通过 @Desensitize 动态打码 (手机/身份证等) │
│    - 内部微服务 RPC: 携带 X-Internal-RPC: true 自动旁路放行明文；        │
│    - 传输态: 通过 @ApiEncrypt / @ApiDecrypt 实现 Controller 报文对称加密。│
└────────────────────────────────────┬────────────────────────────────────┘
                                     │
                                     ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ 3. 存储与检索态 (Persistence & Storage Layer)                           │
│    - 密文落盘: 实体字段通过 EncryptTypeHandler (国密 SM4/AES) 透明加密； │
│      加密引擎具备事务安全阻断机制，发生异常坚决抛出 CryptoException，    │
│      绝不向数据库降级写入明文，防范明文脱库合规事故；                   │
│    - 盲索引检索: 辅助列通过 HMAC-SM3 加盐哈希生成强一致性盲索引 (*_hash)│
│      由 ZhkjMetaObjectHandler 自动注入，仅允许确定性精确等值 (EQ) 查询。 │
└─────────────────────────────────────────────────────────────────────────┘
```

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
├── property/                                  # 1. 配置文件启动期解密
│   ├── EncryptablePropertyEnvironmentPostProcessor.java # 现代 EnvironmentPostProcessor 实现
│   ├── EncryptablePropertySource.java         # 属性源透明代理
│   ├── ZhkjPropertyEncryptor.java             # 属性解密执行器
│   └── ZhkjCryptoCli.java                     # 密文生成 CLI 工具类
├── sm4/                                       # 2. 国密 SM4 核心与环境嗅探
│   └── SM4Util.java                           # SM4 ECB/CBC 算法实现与 isSm4Available() 探测
├── storage/                                   # 3. 数据库持久层密文存储与盲索引
│   ├── annotation/
│   │   ├── EncryptField.java                  # 实体敏感存储加密字段标记
│   │   └── BlindIndex.java                    # 盲索引辅助列映射
│   ├── engine/
│   │   └── StorageCryptoEngine.java           # 存储加密核心引擎 (硬性事务安全阻断)
│   ├── spi/
│   │   ├── CryptoAlgorithmProvider.java       # 加密算法扩展 SPI
│   │   ├── Sm4CryptoAlgorithmProvider.java    # 国密 SM4 实现
│   │   └── AesCryptoAlgorithmProvider.java    # AES 实现
│   └── util/
│       ├── BlindIndexUtil.java                # HMAC-SM3 加盐哈希盲索引工具
│       └── StorageBlindIndexUtil.java         # 存储盲索引辅助
├── advice/                                    # 4. HTTP 报文对称加解密切面
│   ├── ApiDecryptRequestBodyAdvice.java       # 请求入参透明解密
│   └── ApiEncryptResponseBodyAdvice.java      # 响应出参透明加密
└── exception/
    └── CryptoException.java                   # 加解密核心异常 (事务阻断)
```

---

## 🛡️ 4. 架构级合规与安全防线

### 4.1 存储加密引擎事务安全阻断
- 当出现主密钥配置错误、算法提供者异常或字符集不匹配时，`StorageCryptoEngine` 坚决抛出 `CryptoException`，直接触发数据库事务回滚；
- **绝不降级明文存储**：绝不在加密失败时将用户的手机号或身份证直接以明文写入数据库，坚决守牢数据防脱库底线。

### 4.2 盲索引强一致性散列与防哈希漂移
- 盲索引哈希通过 `BlindIndexUtil.generateIndex(plainText)` 计算，底层动态嗅探运行环境，优先采用国密 `HmacSM3`（无 BC 依赖环境平滑降级为 `HmacSHA256`）；
- 散列计算遭遇异常坚决阻断写入，绝不使用随机或缺省值回退，杜绝不同节点生成不一致散列导致的历史数据失效；
- **查询语法约束**：盲索引列仅支持确定性精确等值匹配（`EQ` 或 `IN`），**严禁使用 `LIKE '%...%'` 进行模糊检索**。

---

## ⚙️ 5. 核心配置清单 (`application.yml`)

```yaml
zhkj:
  crypto:
    enabled: true
    # 1. 配置文件 ENC(...) 敏感属性解密配置
    property:
      enabled: true
      password: "MasterKey@2026"               # 生产环境推荐通过环境变量注入
    # 2. HTTP 报文传输层加密配置 (可选)
    crypto-type: AES
    aes-key: "1234567890123456"                # 16 字节 AES 密钥
```

### 命令行密文生成 CLI 工具
在打包或部署前，可使用内置的 CLI 工具生成 `ENC(...)` 密文：

```bash
java -cp zhkj-framework-starter-crypto.jar io.github.ybb88.framework.crypto.property.ZhkjCryptoCli -e "my_db_password" -p "MasterKey@2026"
# 输出: ENC(XyKx5jN5684AxytPGmOTAV7+XSDh/S83BwCb8aCdyGU=)
```

---

## 💡 6. 典型使用范式

### 6.1 配置文件中写入密文
```yaml
spring:
  datasource:
    url: jdbc:mysql://127.0.0.1:3306/db_demo?useUnicode=true&characterEncoding=utf-8
    username: root
    password: "ENC(XyKx5jN5684AxytPGmOTAV7+XSDh/S83BwCb8aCdyGU=)"
```
应用启动时，框架通过 `EnvironmentPostProcessor` 自动将明文注入 Spring 环境变量中，外部日志与监控不会暴露真实密码。

### 6.2 实体类持久层国密加密与盲索引声明
```java
package io.github.ybb88.demo.entity;

import com.baomidou.mybatisplus.annotation.TableField;
import com.baomidou.mybatisplus.annotation.TableName;
import io.github.ybb88.framework.crypto.storage.annotation.BlindIndex;
import io.github.ybb88.framework.crypto.storage.annotation.EncryptField;
import io.github.ybb88.framework.desensitize.annotation.Desensitize;
import io.github.ybb88.framework.desensitize.DesensitizeType;
import io.github.ybb88.framework.mybatis.base.MybatisTenantEntity;
import io.github.ybb88.framework.mybatis.crypto.handler.EncryptTypeHandler;
import lombok.Data;
import lombok.EqualsAndHashCode;

@Data
@EqualsAndHashCode(callSuper = true)
@TableName(value = "biz_customer", autoResultMap = true)
public class CustomerEntity extends MybatisTenantEntity {

    private String customerName;

    /**
     * 身份证号码：
     * 1. 存储态: 国密 SM4 加密落盘
     * 2. 展示态: Jackson 自动脱敏打码 (110101********2345)
     * 3. 盲索引: 自动映射至 idCardHash
     */
    @Desensitize(rule = DesensitizeType.ID_CARD)
    @EncryptField(blindIndexField = "idCardHash")
    @TableField(typeHandler = EncryptTypeHandler.class)
    private String idCard;

    /**
     * 身份证盲索引辅助列 (HMAC-SM3 确定性哈希)
     */
    @BlindIndex(source = "idCard")
    private String idCardHash;
}
```

### 6.3 密文等值精确检索范式
```java
package io.github.ybb88.demo.service;

import com.baomidou.mybatisplus.core.toolkit.Wrappers;
import io.github.ybb88.framework.util.crypto.BlindIndexUtil;
import io.github.ybb88.demo.entity.CustomerEntity;
import io.github.ybb88.demo.mapper.CustomerMapper;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;

import java.util.List;

@Service
@RequiredArgsConstructor
public class CustomerService {

    private final CustomerMapper customerMapper;

    public List<CustomerEntity> findByIdCard(String plainIdCard) {
        // 计算输入的明文身份证哈希
        String hash = BlindIndexUtil.generateIndex(plainIdCard);
        
        // 基于盲索引辅助列进行等值检索
        return customerMapper.selectList(
            Wrappers.<CustomerEntity>lambdaQuery().eq(CustomerEntity::getIdCardHash, hash)
        );
    }
}
```
