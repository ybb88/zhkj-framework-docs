# 细粒度数据权限组件：zhkj-framework-starter-datascope

> **模块定位**: 细粒度数据权限管控核心 Starter，基于 JSqlParser 抽象语法树（AST）实现动态 SQL 重写，提供 `@DataAuth` 注解、权限范围模型 `ScopeModel` 与组织架构解耦 SPI（`DeptAncestorProvider`）。  
> **适用基准**: OpenJDK 21 (LTS), Spring Boot 4.1.1, JSqlParser  
> **模块坐标**: `io.github.ybb88:zhkj-framework-starter-datascope`

---

## 📋 1. 架构定位与核心设计

企业管理系统中，不同角色的用户在同一个租户内部查看数据的范围不同（如：仅本人、本部门、本部门及下级部门、全部数据或自定义范围）。

传统方案通常在 XML 中散落手写 `AND dept_id IN (...)` 或在业务 Service 中硬拼接 SQL，极易引发 SQL 注入、跨机构越权或分页计数不准。

`zhkj-framework-starter-datascope` 构建了安全的数据权限引擎：
1. **JSqlParser 抽象语法树（AST）动态重写**：
   - 拦截带有 `@DataAuth` 的 Mapper 方法，解析原始 SQL 为语法树对象；
   - 构造标准的 `BinaryExpression` 节点递归安全合并进 `WHERE` 表达式，100% 语法严密、零 SQL 注入风险；
2. **UPDATE / DELETE 行级改写闭环与恒假防穿透 (`1=0`)**：
   - 全面支持 SELECT、UPDATE、DELETE 语句的权限改写；
   - 当用户无任何匹配数据权限时，自动注入 `AND 1 = 0` 恒假表达式，彻底阻断全表无条件查询或批量误删；
3. **组织拓扑解耦 SPI (`DeptAncestorProvider`)**：
   - 框架底层不硬编码任何具体业务部门表（如 `sys_dept`）或组织字段，业务端通过实现 SPI 注入具体的部门祖先树逻辑。

---

## 📦 2. 依赖坐标

```xml
<dependency>
    <groupId>io.github.ybb88</groupId>
    <artifactId>zhkj-framework-starter-datascope</artifactId>
</dependency>
```

---

## 🧱 3. 核心包结构与 SPI

```
io.github.ybb88.framework.datascope
├── annotation/
│   └── DataAuth.java                  # Mapper/方法级数据权限注解
├── handler/
│   ├── ScopeModelHandler.java         # 数据权限解析处理器 SPI
│   └── MybatisDataScopeInterceptor.java# 基于 AST 的 MyBatis 拦截器
├── model/
│   ├── ScopeModel.java                # 数据权限作用域模型 (本人/部门/全部)
│   └── DataScopeEnum.java             # 权限范围枚举 (ALL, DEPT, DEPT_AND_CHILD, SELF, CUSTOM)
└── provider/
    └── DeptAncestorProvider.java      # 部门树拓扑提供者 SPI
```

---

## ⚙️ 4. 核心配置清单 (`application.yml`)

```yaml
zhkj:
  datascope:
    enabled: true                      # 是否全局开启数据权限拦截 (默认: true)
```

---

## 💡 5. 典型使用范式

### 5.1 业务端实现组织树 SPI (`DeptAncestorProvider`)
业务服务通过注入自身数据库的部门表查询方法，提供子部门 ID 列表：

```java
package io.github.ybb88.demo.datascope;

import io.github.ybb88.framework.datascope.provider.DeptAncestorProvider;
import io.github.ybb88.demo.mapper.SysDeptMapper;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Component;

import java.util.List;

@Component
@RequiredArgsConstructor
public class CustomDeptAncestorProvider implements DeptAncestorProvider {

    private final SysDeptMapper deptMapper;

    @Override
    public List<Long> getSubDeptIds(Long deptId) {
        // 根据业务自身的部门表或闭包表递归查询所有下级子部门 ID
        return deptMapper.selectSubDeptIds(deptId);
    }
}
```

### 5.2 Mapper 查询接口声明 `@DataAuth`
在 Mapper 查询方法上直接添加注解，指定部门列与人员列：

```java
package io.github.ybb88.demo.mapper;

import com.baomidou.mybatisplus.core.mapper.BaseMapper;
import com.baomidou.mybatisplus.core.metadata.IPage;
import io.github.ybb88.framework.datascope.annotation.DataAuth;
import io.github.ybb88.demo.dto.OrderQueryDto;
import io.github.ybb88.demo.entity.OrderEntity;
import org.apache.ibatis.annotations.Mapper;
import org.apache.ibatis.annotations.Param;

@Mapper
public interface OrderMapper extends BaseMapper<OrderEntity> {

    /**
     * 自动追加当前登录人所属角色的数据权限过滤条件
     */
    @DataAuth(deptField = "dept_id", userField = "create_user")
    IPage<OrderEntity> selectOrderPage(IPage<OrderEntity> page, @Param("param") OrderQueryDto param);
}
```

执行时，框架根据当前登录人角色配置自动追加过滤：
- 若为「本部门及下属部门」：`WHERE ... AND dept_id IN (10, 11, 12)`
- 若为「仅本人」：`WHERE ... AND create_user = 10001`
- 若无权限：`WHERE ... AND 1 = 0`
