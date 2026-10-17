# JPA 持久层组件：zhkj-framework-starter-jpa

> **模块定位**: Spring Data JPA 深度整合持久层 Starter，提供实体基类、Hibernate 租户 Filter 自动装配及国密加密对称性支持。  
> **适用基准**: OpenJDK 21 (LTS), Spring Boot 4.1.1, Hibernate 6.x / 7.x  
> **模块坐标**: `io.github.ybb88:zhkj-framework-starter-jpa`

---

## 📋 1. 架构定位与设计理念

在支持多样化微服务持久层体系中，`zhkj-framework-starter-jpa` 专为采用 Spring Data JPA 与 Hibernate 的业务微服务提供核心能力支持：

1. **契约互通**：基于 `zhkj-framework-data-api` 的统一契约，实现与 MyBatis-Plus 模块对齐的实体模型抽象；
2. **多租户 Hibernate 动态 Filter**：自动向 Hibernate Session 注册租户隔离 Filter（`tenantFilter`），在查询时自动追加租户条件；
3. **审计自动化**：利用 JPA Auditing 特性自动维护创建人、创建时间、修改人与修改时间。

---

## 📦 2. 依赖坐标

```xml
<dependency>
    <groupId>io.github.ybb88</groupId>
    <artifactId>zhkj-framework-starter-jpa</artifactId>
</dependency>
```

---

## 🧱 3. 核心包结构与类

```
io.github.ybb88.framework.jpa
├── base/
│   ├── JpaBaseEntity.java             # JPA 基础审计实体基类 (MappedSuperclass)
│   └── JpaTenantEntity.java           # JPA 多租户审计实体基类 (含 @FilterDef 与 tenantId)
├── config/
│   └── JpaTenantAutoConfiguration.java# JPA 租户 Filter 自动装配配置类
└── constant/
    └── JpaConstant.java               # JPA 模块常量 (Filter 命名等)
```

---

## ⚙️ 4. 核心配置清单 (`application.yml`)

```yaml
spring:
  jpa:
    open-in-view: false
    hibernate:
      ddl-auto: none
    properties:
      hibernate:
        format_sql: true
        show_sql: false

zhkj:
  tenant:
    enabled: true
    column: "tenant_id"
```

---

## 💡 5. 典型使用范式

### 5.1 实体声明
业务实体继承 `JpaTenantEntity`：

```java
package io.github.ybb88.demo.jpa.entity;

import io.github.ybb88.framework.jpa.base.JpaTenantEntity;
import jakarta.persistence.Entity;
import jakarta.persistence.GeneratedValue;
import jakarta.persistence.GenerationType;
import jakarta.persistence.Id;
import jakarta.persistence.Table;
import lombok.Getter;
import lombok.Setter;

import java.math.BigDecimal;

@Getter
@Setter
@Entity
@Table(name = "biz_account")
public class AccountEntity extends JpaTenantEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String accountNo;
    private BigDecimal balance;
}
```

### 5.2 仓储接口
```java
package io.github.ybb88.demo.jpa.repository;

import io.github.ybb88.demo.jpa.entity.AccountEntity;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

import java.util.Optional;

@Repository
public interface AccountRepository extends JpaRepository<AccountEntity, Long> {

    Optional<AccountEntity> findByAccountNo(String accountNo);
}
```
查询时，Hibernate 会根据当前线程 `TenantContext` 自动追加租户条件过滤。
