# 持久层核心组件：zhkj-framework-starter-mybatis

> **模块定位**: MyBatis-Plus 深度整合持久层 Starter，提供泛型审计基类、审计字段自动填充、持久层国密加密类型处理器（`EncryptTypeHandler`）、多租户行级拦截、AST 数据权限与分页拦截器严格拓扑编排。  
> **适用基准**: OpenJDK 21 (LTS), Spring Boot 4.1.1, MyBatis-Plus 3.5.x+  
> **模块坐标**: `io.github.ybb88:zhkj-framework-starter-mybatis`

---

## 📋 1. 架构定位与核心设计

`zhkj-framework-starter-mybatis` 深度扩展 MyBatis-Plus 能力，为企业级分布式应用构建稳固、安全且高性能的持久层基础设施：

1. **强类型泛型实体继承体系**：
   - `MybatisBaseEntity`：默认 Long 雪花算法主键，自动装配主键生成策略、审计字段与逻辑删除；
   - `MybatisAuditEntity<ID>`：自定义主键类型（如 String 租户/编码、UUID），强类型泛型隔离；
   - `MybatisTenantEntity`：内置多租户隔离字段 `tenant_id` 与租户正交契约；
2. **审计与加密全自动填充 (`ZhkjMetaObjectHandler`)**：
   - 新增数据时自动回填 `create_user`、`create_time`、`update_user`、`update_time`；
   - 联动持久层加解密引擎，当检测到 `@EncryptField` 标注字段更新时，自动同步更新密文字段与计算盲索引哈希辅助列；
3. **国密 SM4/AES 存储加密类型处理器 (`EncryptTypeHandler`)**：
   - 数据落盘自动调用底层算法加密为密文字符串；查询映射自动解密为明文；
   - 内置 `isSm4Available()` 反射环境自愈探测，兼顾信创国密与轻量开发环境；
4. **拦截器严格物理拓扑编排（杜绝初始化竞态）**：
   $$\mathbf{TenantLineInnerInterceptor} \longrightarrow \mathbf{MybatisDataScopeInterceptor} \longrightarrow \mathbf{PaginationInnerInterceptor}$$
   租户物理隔离绝对处于第 0 位执行，优先于租户内部的数据权限过滤与分页插件；
5. **高频查询 GC 防护 (`SqlKeyword`)**：
   - 针对动态条件防注入清洗，采用 `private static final Pattern` 静态预编译正则，彻底消除批量查询下的堆内存抖动与频繁 Minor GC。

---

## 📦 2. 依赖坐标

```xml
<dependency>
    <groupId>io.github.ybb88</groupId>
    <artifactId>zhkj-framework-starter-mybatis</artifactId>
</dependency>
```

---

## 🧱 3. 核心包结构与组件

```
io.github.ybb88.framework.mybatis
├── base/                              # 实体分层基类
│   ├── MybatisBaseEntity.java         # 默认 Long 主键基础审计实体 (实现 MybatisAuditEntity<Long>)
│   ├── MybatisAuditEntity.java        # 强类型泛型审计基类 (IEntity<ID>, IAuditEntity<ID>)
│   └── MybatisTenantEntity.java       # 多租户审计实体基类 (ITenantEntity)
├── config/                            # 自动装配与拦截器编排
│   ├── MybatisAutoConfiguration.java  # 核心拦截器注入与分页插件
│   └── MybatisProperties.java         # 属性配置
├── crypto/handler/                    # 国密持久层加解密 TypeHandler
│   └── EncryptTypeHandler.java        # MyBatis 国密 SM4/AES 存储加解密类型处理器
├── handler/                           # 元对象自动填充
│   └── ZhkjMetaObjectHandler.java     # 审计时间、操作人及盲索引联动填充
├── injector/                          # 扩展 SQL 注入器
│   └── ZhkjSqlInjector.java           # 批量插入与增强方法
└── util/
    └── SqlKeyword.java                # 预编译正则防注入过滤工具
```

---

## ⚙️ 4. 核心配置清单 (`application.yml`)

```yaml
mybatis-plus:
  mapper-locations: "classpath*:/mapper/**/*.xml"
  type-aliases-package: "io.github.ybb88.**.entity"
  configuration:
    map-underscore-to-camel-case: true
    cache-enabled: false
    call-setters-on-nulls: true
  global-config:
    db-config:
      id-type: ASSIGN_ID               # 默认采用雪花算法生成 Long 主键
      logic-delete-field: isDeleted     # 逻辑删除字段名
      logic-delete-value: 1             # 逻辑已删除值
      logic-not-delete-value: 0         # 逻辑未删除值
```

---

## 💡 5. 典型使用范式

### 5.1 实体模型声明 (多租户 + 审计 + 国密存储加密 + 盲索引)
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

import java.math.BigDecimal;

@Data
@EqualsAndHashCode(callSuper = true)
@TableName(value = "biz_order", autoResultMap = true)
public class OrderEntity extends MybatisTenantEntity {

    private String orderNo;
    private BigDecimal amount;

    /**
     * 敏感字段：手机号
     * 1. 展示态: Jackson 自动脱敏打码 (138****8000)
     * 2. 存储态: 国密 SM4 透明加密落盘 (EncryptTypeHandler)
     * 3. 盲索引: 框架自动计算 HMAC-SM3 哈希并注入辅助列 mobileHash
     */
    @Desensitize(rule = DesensitizeType.MOBILE_PHONE)
    @EncryptField(blindIndexField = "mobileHash")
    @TableField(typeHandler = EncryptTypeHandler.class)
    private String mobile;

    /**
     * 盲索引辅助列，用于密文等值精确检索
     */
    @BlindIndex(source = "mobile")
    private String mobileHash;
}
```

### 5.2 Mapper 接口声明
```java
package io.github.ybb88.demo.mapper;

import com.baomidou.mybatisplus.core.mapper.BaseMapper;
import io.github.ybb88.demo.entity.OrderEntity;
import org.apache.ibatis.annotations.Mapper;

@Mapper
public interface OrderMapper extends BaseMapper<OrderEntity> {
}
```

### 5.3 业务精准密文等值检索范式
在 Service 或 Controller 中检索加密字段时，调用 `BlindIndexUtil.generateIndex` 计算哈希值后匹配辅助列：

```java
package io.github.ybb88.demo.service;

import com.baomidou.mybatisplus.core.toolkit.Wrappers;
import io.github.ybb88.framework.util.crypto.BlindIndexUtil;
import io.github.ybb88.demo.entity.OrderEntity;
import io.github.ybb88.demo.mapper.OrderMapper;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;

import java.util.List;

@Service
@RequiredArgsConstructor
public class OrderService {

    private final OrderMapper orderMapper;

    public List<OrderEntity> searchByMobile(String plainMobile) {
        // 1. 计算明文的强一致加盐哈希盲索引
        String hash = BlindIndexUtil.generateIndex(plainMobile);

        // 2. 针对盲索引辅助列进行等值 (EQ) 精确查询
        return orderMapper.selectList(
            Wrappers.<OrderEntity>lambdaQuery().eq(OrderEntity::getMobileHash, hash)
        );
    }
}
```

> [!CAUTION]
> **盲索引查询严禁使用模糊匹配**：加盐哈希属于不可逆确定性散列，**只允许执行精确等值匹配（`EQ` 或 `IN`），严禁使用 `LIKE '%...%'`**。
