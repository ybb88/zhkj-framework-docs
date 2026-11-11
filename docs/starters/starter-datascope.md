# 细粒度数据权限组件：zhkj-framework-starter-datascope

> **模块定位**: 数据权限抽象与模型模块。定义 `@DataAuth`、`DataScopeEnum`、`DataScopeModel`、`DataScopeContext`、`DeptAncestorProvider` SPI，并提供框架无关的 JSqlParser 行级改写引擎 `SqlDataScopeAstEngine`。MyBatis 拦截与规则查询由 `zhkj-framework-starter-mybatis` 完成。  
> **适用基准**: OpenJDK 21, Spring Boot 4.x, JSqlParser（`mybatis-plus-jsqlparser`）  
> **模块坐标**: `io.github.ybb88:zhkj-framework-starter-datascope`

---

## 📋 1. 架构定位与核心设计

1. **注解声明**：`@DataAuth` 标注在 Mapper 方法或接口上，声明资源编码、过滤列与默认范围。
2. **规则来源**：运行期优先读取数据库中的数据权限配置（按 Mapper 方法与角色、或按资源编码）；未命中时使用注解中的值。
3. **SQL 改写**：`SqlDataScopeAstEngine.rewriteSql` 对 `SELECT` / `UPDATE` / `DELETE` 解析 AST，将条件以 `AND (condition)` 合并进 `WHERE`。
4. **组织拓扑 SPI**：`DeptAncestorProvider` 提供子孙部门 ID；未提供时通过 `sys_org` 表的 `ancestors` 列查询。
5. **管理员放行**：角色为 `administrator` 或 `super_admin` 时不做改写（见第 6 节）。

---

## 📦 2. 依赖坐标

```xml
<dependency>
    <groupId>io.github.ybb88</groupId>
    <artifactId>zhkj-framework-starter-datascope</artifactId>
</dependency>
```

`zhkj-framework-starter-mybatis` 以 `<optional>true</optional>` 声明本模块，业务工程需显式引入本模块；MyBatis 侧的拦截器与规则查询 Bean 由 `starter-mybatis` 的 `MybatisDataScopeAutoConfiguration` 在 `zhkj.datascope.enabled=true` 时装配。

---

## 🧱 3. 核心包结构与 SPI

```
io.github.ybb88.framework.datascope
├── annotation/DataAuth.java                     # 方法/类级注解
├── config/DataScopeAutoConfiguration.java       # 注册 DataScopeSnapshotProvider
├── constant/DataScopeConstant.java              # DEFAULT_COLUMN = "create_dept"
├── context/
│   ├── DataScopeContext.java                    # ThreadLocal 当前数据权限模型
│   └── DataScopeSnapshotProvider.java           # 快照属性 DATA_SCOPE_MODEL
├── engine/SqlDataScopeAstEngine.java            # JSqlParser 改写
├── enums/DataScopeEnum.java                     # ALL / OWN / OWN_DEPT / OWN_DEPT_CHILD / CUSTOM
├── exception/DataScopeException.java
├── handler/
│   ├── DataScopeHandler.java                    # 改写接口（仅 MybatisDataScopeHandler 实现，见第 8 节）
│   └── ScopeModelHandler.java                   # 规则查询接口
├── model/DataScopeModel.java
├── props/DataScopeProperties.java               # zhkj.datascope.*
├── provider/DeptAncestorProvider.java           # 子孙部门 SPI
└── util/DataScopeSqlUtil.java                   # 规则与部门查询 SQL 构造
```

自动装配：`DataScopeAutoConfiguration`（`META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`）仅注册快照提供者 `DataScopeSnapshotProvider`（另由 `META-INF/services/io.github.ybb88.framework.context.ContextSnapshotProvider` 声明）。规则查询、拦截器等 Bean 由 `starter-mybatis` 的 `MybatisDataScopeAutoConfiguration` 在 `zhkj.datascope.enabled=true` 时注册。

---

## ⚙️ 4. 核心配置清单 (`application.yml`)

```yaml
zhkj:
  datascope:
    enabled: false                       # 默认 false，需显式开启
    mapper-key: [page, Page, list, List, update, Update, delete, Delete]
    mapper-exclude: [FlowMapper]
    dept-table: sys_org
    dept-id-column: id
    dept-ancestors-column: ancestors
    dept-logic-delete-column: is_deleted
    dept-logic-delete-value: "0"
    scope-data-table: zhkj_scope_data
    role-scope-table: zhkj_role_scope
```

| 键 | 默认值 | 说明 |
|----|--------|------|
| `enabled` | `false` | 为 `true` 时装配 MyBatis 数据权限拦截器 |
| `mapper-key` | `[page, Page, list, List, update, Update, delete, Delete]` | 无 `@DataAuth` 时，方法名包含其中任一字符串才参与改写 |
| `mapper-exclude` | `FlowMapper` | Mapper 简单类名包含其中任一字符串时跳过（无注解时） |
| `dept-table` | `sys_org` | 部门表名 |
| `dept-id-column` | `id` | 部门主键列 |
| `dept-ancestors-column` | `ancestors` | 部门祖先路径列 |
| `dept-logic-delete-column` | `is_deleted` | 部门逻辑删除列 |
| `dept-logic-delete-value` | `"0"` | 部门未删除值；所有表名与列名须匹配 `^[a-zA-Z0-9_]+$`（`DataScopeSqlUtil`） |
| `scope-data-table` | `zhkj_scope_data` | 数据权限配置表 |
| `role-scope-table` | `zhkj_role_scope` | 角色与数据权限关联表 |

---

## 🧩 5. 注解与模型

### 5.1 `@DataAuth`

| 属性 | 默认值 | 说明 |
|------|--------|------|
| `enabled` | `true` | `false` 时该方法跳过改写 |
| `code` | `""` | 资源编码，用于按资源查询规则 |
| `column` | `create_dept` | 过滤列 |
| `type` | `DataScopeEnum.ALL` | 范围类型 |
| `field` | `"*"` | 复制到 `DataScopeModel.scopeField`，改写逻辑不读取 |
| `value` | `""` | `CUSTOM` 类型的条件模板 |

注解可标注在方法或 Mapper 接口上；方法上的注解优先。

### 5.2 `DataScopeEnum`

| 枚举 | `type` 值 | 生成条件 |
|------|-----------|----------|
| `ALL` | 1 | 不改写 |
| `OWN` | 2 | `<column> = <userId>`；列为空或为 `create_dept` 时列名改为 `create_user` |
| `OWN_DEPT` | 3 | `<column> = <deptId>`（单值；列为空时默认 `dept_id`） |
| `OWN_DEPT_CHILD` | 4 | `<column> IN (<deptId>, <子孙部门 ID>...)`（列为空时默认 `dept_id`；仅一个 ID 时为 `=`） |
| `CUSTOM` | 5 | `scopeValue` 经占位符替换后作为条件 |

无可用 ID 时生成 `1 = 0`。

### 5.3 `DataScopeModel`

字段：`searched`、`resourceCode`、`scopeColumn`（默认 `create_dept`）、`scopeType`（默认 `ALL`）、`scopeField`、`scopeValue`。

---

## 🔄 6. 改写规则（`MybatisDataScopeHandler`）

1. 取当前用户（`AuditUserProvider`）。无用户身份时，`MybatisDataScopeInterceptor` 抛出 `SecurityException`，拒绝执行。
2. `AuditUserProvider.isAdmin()` 为 `true`（角色为 `administrator` 或 `super_admin`）时不改写。
3. 规则查询顺序：按 `mapperId` + 角色 ID 查询 `scope-data-table` 与 `role-scope-table`；未命中且注解提供 `code` 时按资源编码查询；均未命中则使用注解值（无注解时 `scopeType` 为 `ALL`，不改写）。
4. 按范围类型生成条件并通过 `SqlDataScopeAstEngine` 合并到原 SQL。
5. `UPDATE` 与 `DELETE` 同样改写。`SELECT` 在执行前改写一次，同一 `BoundSql` 不重复追加。

部门子孙查询（`ZhkjScopeModelHandler.getDeptAncestors`）：

- 若存在 `DeptAncestorProvider` Bean（`ObjectProvider.getIfAvailable()`）且其返回值非 `null`（空列表同样直接采用），直接使用其结果；调用异常时记录 WARN 并降级为动态 SQL。
- 否则执行 `select <id> from <dept-table> where <ancestors> like concat(concat('%', ?),'%') and <logic-col> = <value>`（参数为 `deptId`，由 `DataScopeSqlUtil.dataByDept` 生成），结果写入 `SYS_CACHE`；结果为空时下次仍会重新查询。

---

## 💡 7. 典型使用范式

### 7.1 Mapper 注解

```java
@Mapper
public interface OrderMapper extends BaseMapper<Order> {

    /** 仅本部门可见 */
    @DataAuth(code = "order", column = "dept_id", type = DataScopeEnum.OWN_DEPT)
    List<Order> selectOrderList(@Param("query") OrderQuery query);

    /** 仅本人可见 */
    @DataAuth(code = "task", column = "create_user", type = DataScopeEnum.OWN)
    List<Task> selectTaskList(@Param("query") TaskQuery query);

    /** 自定义条件（value 为 SQL 片段，占位符由用户信息替换） */
    @DataAuth(code = "project", column = "project_id", type = DataScopeEnum.CUSTOM, value = "project_id IN (1, 2, 3)")
    List<Project> selectProjectList(@Param("query") ProjectQuery query);
}
```

生成条件示例：`WHERE status = 1 AND (dept_id IN (100, 101))`。

### 7.2 子孙部门 SPI

```java
@Component
@RequiredArgsConstructor
public class CustomDeptAncestorProvider implements DeptAncestorProvider {

    private final SysDeptMapper deptMapper;

    @Override
    public List<Long> getDeptAncestors(Long deptId) {
        return deptMapper.selectSubDeptIds(deptId);   // 返回 deptId 及其所有子孙部门 ID（包含 deptId 自身，与 DeptAncestorProvider Javadoc 一致）
    }
}
```

`OWN_DEPT_CHILD` 会把当前用户部门 ID 与该 SPI 结果合并（框架不去重，SPI 结果含自身时会出现重复 ID）。方法名 `getDeptAncestors` 沿用框架命名，语义为部门及其子孙部门。

### 7.3 上下文

```java
DataScopeModel model = DataScopeContext.getDataScope();   // 改写期间有效
```

---

## 🛡️ 8. 行为约束与注意事项

- 解析失败时，`SqlDataScopeAstEngine.rewriteSql` 记录 WARN 日志并返回原 SQL（不附加条件）。依赖数据权限的语句须保证可被 JSqlParser 解析。
- `DataScopeSqlUtil` 对表名与列名做字符白名单校验；`DataAuth.column` 与 `scope_column` 不做校验，应由开发者或数据库配置保证。
- `CUSTOM` 类型的 `scopeValue` 按原文拼接进 SQL，仅可由可信配置写入。
- `MybatisDataScopeInterceptor` 的字段与构造参数均为具体类 `MybatisDataScopeHandler`（`MybatisDataScopeInterceptor.java` 第 59、63 行），`MybatisDataScopeAutoConfiguration` 亦只注册该类型 Bean；因此 `DataScopeHandler` 接口的其它实现不会被调用。当前可扩展点为 `DeptAncestorProvider`。
- 部门子孙默认查询使用 `LIKE '%<id>%'` 匹配 `ancestors` 列，未对分隔符做边界处理（见第 9 节）。

---

## 🚫 9. 已知问题（代码现状）

- `ZhkjScopeModelHandler.getDeptAncestors` 默认 SQL（`DataScopeSqlUtil.dataByDept`，`DataScopeSqlUtil.java` 第 50 行）使用子串匹配。部门 ID `1` 会命中 `ancestors` 中包含 `10`、`11` 的记录。
- 数据权限对角色为 `administrator` 或 `super_admin` 的用户整体放行（`RoleUtil.isPrivileged`）。

---

## 📎 10. 相关文档

- [starter-datascope 模块 README](../../zhkj-framework-starter-datascope/README.md)
- [starter-mybatis 模块 README](../../zhkj-framework-starter-mybatis/README.md)：MyBatis 拦截器与规则查询实现
