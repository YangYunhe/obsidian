---
title: Superset 安全配置
date: 2026-10-26
tags:
  - superset
  - 安全
  - 权限
  - 运维
---

# Superset 安全配置

> 管理员视角的 Superset **6.1.0** 安全指南：内置角色（Admin / Alpha / Gamma / sql_lab / Public）、权限模型、仪表盘访问控制、行级安全（RLS）、API Key、会话安全与 CSP。
> 原文：<https://superset.apache.org/admin-docs/6.1.0/security/>

**按需求找章节：**

| 我要……                    | 看第    |
| ----------------------- | ----- |
| 搞清 5 个内置角色各能干什么         | 二     |
| 让某个团队只看到指定的几张表          | 三、九   |
| 控制某个仪表盘谁能看              | 四     |
| 限制某角色只能看自己部门的数据行        | 十     |
| 让 CI/CD 或 AI 助手免登录调 API | 七     |
| 加固会话与浏览器侧安全             | 十一、十二 |

## 一、认证授权的底座：FAB

Superset 的**认证与授权由 Flask AppBuilder（FAB）** 处理。FAB 是构建在 Flask 之上的应用开发框架，提供认证、用户管理、权限与角色。

> 所以 Superset 的角色、权限语义都要按 FAB 的模型理解，遇到细节问题应查 FAB 的 Security 文档。

## 二、内置角色

Superset 自带一组**由 Superset 自己维护**的角色。可以认为它们会随 Superset 演进（以及你升级版本）保持最新。

> [!warning] 不要改内置角色的权限
> 即使 Admin 用户有能力这么做，也**不建议**给内置角色增删权限。运行 `superset init`（升级版本之间常做）时，这些角色的权限会被**重新同步回原始值**。
>
> 各角色权限清单见 Superset 仓库的 `/RESOURCES/STANDARD_ROLES.md`。

### 1. Admin

拥有**所有**权限，包括给其他用户授予 / 撤销权限，以及修改他人的 slice 与 dashboard。

#### 威胁模型与权限边界

Superset 的权限模型是细粒度的，而 **Admin 角色被视为完全受信任**：Admin 对应用配置、UI 渲染和访问控制有完全控制权。

因此，Admin 所做的、会改变应用行为或呈现的操作——注入自定义 CSS、修改 Jinja 模板、改动安全开关等——**按设计就是管理能力**。

依据 MITRE CNA Rule 4.1，一个漏洞必须违反**明确的安全策略**。既然 Admin 角色被定义为受信任的运维边界，用 Admin 权限执行的操作就不越过安全边界。因此，**严格依赖 Admin 权限的攻击向量不算安全漏洞，也不会分配 CVE**。

> 实际含义：不要用 Admin 账号跑日常分析；给业务人员的账号不要给 Admin。

### 2. Alpha

- 可访问**所有**数据源；
- 不能给他人授予 / 撤销权限；
- 只能修改**自己拥有**的对象；
- 可以添加和修改数据源。

### 3. Gamma

权限受限，是典型的**内容消费者**：

- 只能消费**通过另一个补充角色**授予它的数据源中的数据；
- 只能看这些数据源做出来的 slice 与 dashboard；
- 目前**不能**修改或新增数据源；
- 但可以创建 slice 和 dashboard；
- 在仪表盘 / slice 列表视图里，只看到自己有权访问的对象。

### 4. sql_lab

`sql_lab` 角色授予 **SQL Lab** 的访问权。

注意数据库访问是另一层：Admin 默认能访问所有数据库，而 **Alpha 与 Gamma 需要按数据库逐个授权**。

除基础 `sql_lab` 角色外，还有两个 SQL Lab 权限需要**显式授予**：

| 权限 | 作用 |
| --- | --- |
| `can_estimate_query_cost` on `SQLLab` | 运行前估算查询成本 |
| `can_format_sql` on `SQLLab` | 用数据库方言格式化 SQL |

授予方式：**Security → List Roles**，把权限加到相应角色上。

### 5. Public

最严格的内置角色，专为**匿名 / 未认证**用户查看仪表盘而设计，只提供最小只读能力：

- 查看仪表盘与图表
- 使用交互式仪表盘筛选器
- 访问仪表盘与图表的永久链接（permalink）
- 阅读嵌入的仪表盘
- 查看图表上的注释

**明确不包括**：

- 对仪表盘、图表、数据集的任何写权限
- SQL Lab
- 分享功能
- 用户资料或管理功能
- 大多数 Superset 功能的菜单入口

匿名用户会在配置 `AUTH_ROLE_PUBLIC`（FAB 设置）时**自动获得** Public 角色。`PUBLIC_ROLE_LIKE` 是可选设置，控制执行 `superset init` 时把哪些权限同步给 Public 角色：

```python
# 可选：给 Public 角色同步一套合理的默认权限
PUBLIC_ROLE_LIKE = "Public"

# 另一种选择：从 Gamma 复制权限以获得更宽的访问
# PUBLIC_ROLE_LIKE = "Gamma"
```

如果你更愿意**手动**配置 Public 的权限（或改用 `DASHBOARD_RBAC` 在仪表盘级别授权），就不需要设置 `PUBLIC_ROLE_LIKE`。

**几个要点：**

1. **数据访问仍需单独授予**
   Public 只给 UI / API 权限，还必须授予查看仪表盘所必需的**特定数据集**访问权。两种做法：
   - **不启用 `DASHBOARD_RBAC`**：只要用户有权访问仪表盘用到的**至少一个**数据集，仪表盘就会出现在列表里并可访问。授予方式：**Menu → Security → List Roles → Public**，添加相关数据源。所有使用这些数据集的**已发布**仪表盘都会可见。
   - **启用 `DASHBOARD_RBAC`**：匿名用户只会看到属性里**显式添加了 Public 角色**的仪表盘；**不需要**数据集权限——`DASHBOARD_RBAC` 会处理级联权限检查。这样能更细地控制哪些仪表盘对外公开。

2. **角色同步会覆盖手动修改**
   内置角色（Admin、Alpha、Gamma、sql_lab，以及 `PUBLIC_ROLE_LIKE = "Public"` 时的 Public）的权限会在 `superset init` 时同步，**手动改的权限可能在升级时被覆盖**。要自定义 Public，两种稳妥做法：
   - 直接编辑 Public，且**不设置** `PUBLIC_ROLE_LIKE`（这样 `superset init` 不会覆盖）；
   - 用 UI 的 **Copy Role** 把 Public 复制成新角色（如 `Public_Custom`），改好权限后，两处配置一起改：

```python
PUBLIC_ROLE_LIKE = "Public_Custom"
AUTH_ROLE_PUBLIC = "Public_Custom"
```

### 6. 五个角色横向对比

| 角色 | 数据访问 | 能否授权他人 | 能否改数据源 | 典型用途 |
| --- | --- | --- | --- | --- |
| **Admin** | 全部 | 能 | 能 | 平台管理员（**别给业务账号**） |
| **Alpha** | 全部 | 不能 | 能（自己拥有的） | 数据团队、分析师 |
| **Gamma** | 仅被授予的数据源 | 不能 | 不能 | 业务查看者（需配补充角色） |
| **sql_lab** | 另需按库授权 | 不能 | 不能 | 需要写 SQL 的用户 |
| **Public** | 仅被授予的数据集 | 不能 | 不能 | 匿名访问已发布仪表盘 |

## 三、给 Gamma 用户开放指定数据源

目标是：用户**只能**访问特定数据集。步骤：

1. 确认该受限用户**只**被分配了 **Gamma** 角色；
2. 新建角色：**Menu → Security → List Roles**，点 **+**；
3. 在弹窗里给角色命名、关联用户，并在 **Permissions 下拉**里选择表——直接点下拉用 typeahead 搜表名；
4. 让被授予该角色的 Gamma 用户确认：能看到这些表对应的对象（仪表盘与 slice）。

> Gamma 提供“能看内容”的基础能力，补充角色提供“能看哪些数据”。两者叠加才生效。

## 四、仪表盘访问控制

仪表盘的访问由 **owner**（对该仪表盘有编辑权限的用户）管理。非 owner 用户的访问有两种方式。

> 前提：仪表盘必须**已发布（published）**，才会对其他人可见。

### 1. 基于数据集的访问（默认）

默认情况下，只要用户对仪表盘里用到的**至少一个数据集**有访问权，就能查看该已发布仪表盘。授予方式：把相应数据源权限加到角色上（**Menu → Security → List Roles**）。

这是最简单的做法，但它是 **all-or-nothing**：用户对某个数据集有权限，就能看到**所有**用到该数据集的已发布仪表盘。

### 2. 基于仪表盘的访问（`DASHBOARD_RBAC`）

要按角色精确控制能访问哪些仪表盘，启用这个 feature flag：

```python
FEATURE_FLAGS = {
    "DASHBOARD_RBAC": True,
}
```

启用后，可在每个仪表盘的**属性**里指定角色；用户只会看到**显式添加了自己角色**的仪表盘。

**重要注意事项：**

- 仪表盘访问会**绕过数据集级检查**——给某角色一个仪表盘，等于隐式授予该仪表盘内所有图表与数据集的读权限；
- **没有分配任何角色**的仪表盘会回退到基于数据集的访问；
- 仪表盘仍然必须**已发布**才可见。

**特别适合：**

- 让某些仪表盘公开、其他保持私有；
- 在不暴露底层数据集的情况下授权访问仪表盘；
- 建立与数据集归属不一致的、按仪表盘划分的访问模式。

## 五、SQL 执行安全

Superset 提供了一些防护机制，比如 `DISALLOWED_SQL_FUNCTIONS`，用来阻止从 SQL Lab 等界面直接执行**可能有危害的数据库函数或系统变量**。但必须理解以下几点。

### 1. Superset 不是数据库防火墙

`DISALLOWED_SQL_FUNCTIONS` 这类内置检查只是一层防护，**无法保证**抵御所有数据库级威胁或高级绕过手法（例如特定的注释注入）。它们应被视为数据库安全的**补充**，而非替代。

### 2. 配置是关键

防护是否有效，很大程度上取决于管理员的配置：维护 `DISALLOWED_SQL_FUNCTIONS` 列表、谨慎管理 feature flag（例如 `ENABLE_TEMPLATE_PROCESSING`）、以及其它安全设置。

### 3. 数据库安全才是根本

保护数据库访问、控制权限、阻止未授权函数执行的**最终责任在 DBA 与数据库安全团队**。推荐做法：

| 实践 | 说明 |
| --- | --- |
| **最小权限** | Superset 用**专用数据库账号**连接，只给运行所需的最小权限（通常是对必要 schema / 表的只读权限） |
| **数据库角色与权限** | 用数据库原生角色限制敏感函数、系统变量（如 `@@hostname`）、schema、表的访问 |
| **网络安全** | 用数据库防火墙或代理等网络层控制限制连接来源 |
| **审计** | 开启数据库级审计，监控已执行的查询与访问模式 |

把这些数据库侧实践与 Superset 的可配置防护叠加，才有纵深防御。

### 4. 数据集样本访问（6.1.0 的变化）

`get_samples()` 端点现在会执行**数据源级访问控制**：用户只能从**被显式授权**的数据集获取样例行——与运行图表查询时是同一套权限检查。

这补上了此前的一个缺口：未认证或低权限的访问可能拿到样本数据。

## 六、用户与角色管理的 REST API

FAB 提供了用户 CRUD 的 REST API，但该功能处于 **beta**，且 Superset 中**默认不启用**。

启用方式：

```python
FAB_ADD_SECURITY_API = True
```

配置后，Swagger 中会出现更多 **Security** 端点可供探索。

## 七、API Key 认证

Superset 支持**长期有效的 API Key**，面向服务账号、CI/CD 流水线与程序化集成（含 MCP 客户端）。

### 1. 启用

API Key 认证**默认关闭**。需要在 `superset_config.py` 里设置 FAB 配置项，**同时**打开对应的 feature flag（否则管理入口的 UI 不出现）：

```python
FAB_API_KEY_ENABLED = True
FEATURE_FLAGS = {
    "FAB_API_KEY_ENABLED": True,
}
```

- 配置项负责在后端注册 `ApiKeyApi` blueprint；
- feature flag 负责决定用户能否看到管理 key 的 UI。

### 2. 创建

启用后，每个用户在自己的资料页管理自己的 key：

1. 打开右上角用户菜单 → **Info**，进入 User Info 页；
2. 展开 **API Keys** 区域；
3. 点 **+ API Key**；
4. 输入名称，可选填过期日期；
5. **复制生成的 token——它只显示一次。**

只有对 `ApiKey` 拥有 `can_read` 与 `can_write` 权限的用户（默认授予 Admin）才能管理 API Key。

### 3. 使用

作为 Bearer token 放进 `Authorization` 头：

```http
Authorization: Bearer <your-api-key>
```

适用于所有 REST API 端点与 MCP server。请求会以**创建该 key 的用户的权限**执行。

### 4. 适用场景

- **CI/CD**：自动化导出 / 导入图表与仪表盘；
- **MCP 集成**：让 AI 助手无需交互式登录即可连接；
- **外部服务**：把仪表盘嵌入其它应用；
- **服务账号**：不随会话 cookie 过期的长期凭证。

> [!caution] 妥善保管
> 任何拿到有效 key 的人都能**代表创建者**发起请求。一旦泄露，立即到 User Info 页的 API Keys 区域删除该 key。

## 八、自定义权限

FAB 暴露的权限非常细粒度，支持高度定制。FAB 会为**每个模型**自动创建大量权限（`can_add`、`can_delete`、`can_show`、`can_edit`……），**每个视图**同样如此。此外 Superset 还会暴露更细的权限，例如 `all_datasource_access`。

> 不建议改动 3 个基础角色——Superset 的很多行为建立在对它们的假设之上。但你可以**创建自己的角色**，并与现有角色**取并集**。

权限的四大类别：

| 类别 | 含义 | 例子 |
| --- | --- | --- |
| **Model & Action** | 模型是 Dashboard、Slice、User 这类实体，每个模型有固定权限集 | 给角色加 `Dashboard` 的 `can_delete`，授予该用户后即可删仪表盘 |
| **Views** | 视图是单个网页，如 Explore 视图、SQL Lab 视图 | 授予后用户能在菜单里看到该入口并打开页面 |
| **Data source** | 每个数据源都会生成一个权限 | 没有 `all_datasource_access` 时，用户只能看被授予的 slice、浏览被授予的数据源 |
| **Database** | 授予某个数据库的访问权 | 可访问该库内**所有**数据源；若已授予 SQL Lab 相关权限，还能在 SQL Lab 里查询该库 |

## 九、把访问限制到数据源子集

推荐做法：给用户 **Gamma 角色**，再叠加其它“提供特定数据源访问”的角色。并且**为每一种访问画像单独建一个角色**。

例：Finance 团队需要一组数据库与数据源，就把这些权限**合并到一个角色**里。属于该画像的用户统一获得：

- **Gamma**：作为模型与视图的基础；
- **Finance 角色**：一堆数据对象权限的集合。

一个用户可以同时拥有多个角色。例如 Finance 团队的高管可以同时被授予 **Gamma + Finance + Executive**，其中 Executive 角色提供只对高管开放的数据源与仪表盘。

在 Dashboards 视图里，用户只能看到依据其角色与权限有权访问的对象。

## 十、行级安全（Row Level Security）

在 **Security** 菜单下可以创建 RLS 过滤器，把它分配给**特定数据集**和**一组角色**。

例：让 Finance 团队成员只能看到 `department = "finance"` 的行：

1. 创建一条 RLS 过滤器，clause 写 `department = "finance"`；
2. 把该 clause 分配给 **Finance 角色**和它适用的**数据集**。

`clause` 字段可以包含任意文本，会被拼进生成 SQL 的 `WHERE` 子句。所以还能做：

- 最近 30 天：`date_field > DATE_SUB(NOW(), INTERVAL 30 DAY)`；
- 多条件：`client_id = 6 AND advertiser="foo"`。

启用 `ENABLE_TEMPLATE_PROCESSING` 后，RLS clause 还支持 **Jinja 模板**，可写动态过滤：

```jinja
user_id = '{{ current_username() }}'
```

这样就能按登录用户限制可见行。

### 1. 两种过滤器类型

| 类型 | 生效范围 | 用途 |
| --- | --- | --- |
| **Regular** | 查询用户属于该过滤器**已分配的角色之一**时生效 | 限制特定角色能看到什么 |
| **Base** | 对**除已分配角色之外**的所有用户生效 | 定义默认限制，让特权角色（如 Admin）豁免 |

**Base 的典型用法**：clause 写 `1 = 0`、角色选 Admin，则除 Admin 外**所有人都看不到任何行**——一个“默认拒绝”的基线。

### 2. Group Key 与过滤器的组合

所有适用的 RLS 过滤器会在加入查询前**合并**，规则是：

- **同一 group key** 的过滤器用 **OR** 组合（组内任一匹配即可）；
- **不同 group**（不同 key，或无 key）之间用 **AND** 组合（所有组都要匹配）；
- **没有 group key** 的过滤器各自成组，且总是参与 AND。

例：某数据集有三条过滤器：

| 过滤器 | Clause | Group Key |
| --- | --- | --- |
| F1 | `department = 'Finance'` | `department` |
| F2 | `department = 'Marketing'` | `department` |
| F3 | `region = 'Europe'` | `region` |

最终 WHERE 子句是：

```sql
(
  department = 'Finance'
  OR department = 'Marketing'
)
AND
(
  region = 'Europe'
)
```

**冲突的过滤器**：`client_id = 4` 与 `client_id = 5` 若**没有共享 group key**，会被 AND 起来变成 `client_id = 4 AND client_id = 5`，永远为假、结果为空。如果本意是“二选一”，请给它们**相同的 group key**，这样才会用 OR。

### 3. RLS 与虚拟（SQL-based）数据集

RLS 过滤器分配给**数据集**，而不是直接分配给底层数据库表。这对虚拟数据集影响很大：

- **物理数据集**（直接由表或视图支撑）：分配给数据集的 RLS 会作为 `WHERE` 子句加进查询；
- **虚拟数据集**（由自定义 SQL 定义）：直接分配给虚拟数据集的 RLS 会应用到**包装该 SQL 的外层查询**；同时，虚拟数据集 SQL 引用到的**底层物理数据集**上的 RLS，会被注入到**每个被引用表的内部子查询**里。

例如：

- 物理数据集 `orders` 上有 RLS `region = 'US'`；
- 虚拟数据集定义为 `SELECT * FROM orders WHERE status = 'active'`。

受该 RLS 影响的用户实际看到的是：

```sql
SELECT *
FROM (
  SELECT * FROM orders
  WHERE (region = 'US') AND status = 'active'
) ...
```

**关键注意点：**

- 一般**不需要**在物理数据集和虚拟数据集上重复设置 RLS——物理数据集上的会在查询时自动应用；
- 如果直接把 RLS 分配给虚拟数据集，clause 必须引用**虚拟数据集输出中的列**，不一定是底层表的列；
- 在 SQL Lab 中，**只有启用 `RLS_IN_SQLLAB` feature flag 时**才会强制执行 RLS：此时对带有 RLS 过滤器的数据集所对应的表执行查询，会自动注入相应谓词。

### 4. 通过 API 检查 RLS

可以用 RLS REST API 审计“配置了哪些过滤器、影响了哪些数据集”。这需要 `Row Level Security` 资源的 `can_read` 权限。

列出所有 RLS 规则：

```http
GET /api/v1/rowlevelsecurity/
```

按数据集过滤（Rison 查询语法）：

```http
GET /api/v1/rowlevelsecurity/?q=(filters:!((col:tables,opr:rel_m_m,value:<dataset_id>)))
```

按角色过滤：

```http
GET /api/v1/rowlevelsecurity/?q=(filters:!((col:roles,opr:rel_m_m,value:<role_id>)))
```

查看某条规则详情：

```http
GET /api/v1/rowlevelsecurity/<id>
```

响应包含 `name`、`filter_type`（Regular 或 Base）、`clause`、`group_key`、分配的 `tables`（含 id、schema、table_name）与 `roles`（含 id、name）。

**审计虚拟数据集**：先按该虚拟数据集的 ID 查**直接分配**的规则；再查它 SQL 里引用的**物理数据集**——那些 RLS 在查询时同样会生效。

## 十一、用户会话

Superset 使用 Flask 与 Flask-Login 管理用户会话。会话 cookie 在请求之间维持会话信息与用户状态；它**不包含**用户个人信息，但用于在服务端标识一个会话。

会话 cookie 用应用的 `SECRET_KEY` 加密，**客户端无法读取**。因此保管好 `SECRET_KEY` 非常重要，应设置为安全、唯一、复杂的随机值。

### 1. 相关 Flask 设置

| 设置 | 默认值 | 说明 |
| --- | --- | --- |
| `SESSION_COOKIE_HTTPONLY` | `False` | cookie 是否带 `HttpOnly` 标志 |
| `SESSION_COOKIE_SECURE` | `False` | 标记为 secure 的 cookie 只在 HTTPS 请求中发送；应用本身必须走 HTTPS 才有意义 |
| `SESSION_COOKIE_SAMESITE` | `"Lax"` | 阻止浏览器在**跨站**请求中携带该 cookie |
| `PERMANENT_SESSION_LIFETIME` | `"31 days"` | 永久会话的生命周期（`datetime.timedelta`） |

### 2. 切换到服务端会话

服务端会话在**安全与性能**上都优于客户端会话：会话数据存在服务端，只把 **session ID** 发给客户端。

流程：

1. 用户登录 → 服务端创建会话，把 session ID 通过 cookie 发给客户端；
2. 客户端每次请求带上 session ID，服务端据此取出会话数据；
3. 登出 → 服务端销毁会话，客户端删除 cookie。

这降低了**重放攻击与会话劫持**的风险。Superset 用 **Flask-Session** 实现服务端会话：

```python
SESSION_SERVER_SIDE = True
```

Flask-Session 支持多种后端；完整的 Redis 示例见官方的 production hardening 指南。

## 十二、内容安全策略（CSP）

Superset 通过 **Talisman** 扩展实现 **Content Security Policy（CSP）**，这是额外一层防护，用于检测与缓解 **XSS**、数据注入等攻击。

CSP 让服务器管理员可以指定浏览器认可的**可执行脚本来源域名**，从而减少甚至消除 XSS 的发生途径。兼容 CSP 的浏览器只会执行来自允许域名的脚本，忽略其它一切脚本（包括内联脚本和 HTML 事件处理属性）。策略由一组 policy directive 描述，每条对应某类资源或策略区域。

> 生产部署时**正确配置 CSP 极其重要**。

`config.py` 里有两个相关变量：

| 变量 | 说明 |
| --- | --- |
| `TALISMAN_ENABLED` | 默认 `True`；设为 `False` 即关闭 CSP |
| `TALISMAN_CONFIG` | 存放实际的策略定义，以及要传给 Talisman 的其他参数 |

生产模式下 Superset 启动时会检查是否存在 CSP；没有就发出带安全风险说明的警告。如果 CSP 是在 Superset 之外用其它软件定义的，管理员可以用 `CONTENT_SECURITY_POLICY_WARNING` 关闭这个警告。

### 1. CSP 的硬性要求

- **必须有 `style-src 'unsafe-inline'`**，否则 Superset 无法运行：

```text
style-src 'self' 'unsafe-inline'
```

- **只有带 nonce 的脚本才能加载执行**。nonce 是 Talisman 每次页面加载自动生成的随机串，可用 Jinja 宏 `csp_nonce()` 取当前值：

```html
<script nonce="{{ csp_nonce() }}">
  /* my script */
</script>
```

- 有些仪表盘用 **data URI** 加载图片，需要 `img-src` 里包含 `data:`：

```text
img-src 'self' data:
```

- **MapBox 图表**使用 worker，并需要连接 MapBox 服务器：

```text
worker-src 'self' blob:
connect-src 'self' https://api.mapbox.com https://events.mapbox.com
```

- **Cartodiagram 图表**会从外部资源请求地图数据（图片与 json），而这些资源可被用户编辑，因此 `img-src` 与 `connect-src` 需要允许域名列表或通配符 `'*'`；
- 其它指令默认 `'self'`，把内容限制在同源。

### 2. Talisman 的其它安全选项

`TALISMAN_ENABLED = True` 会用 Talisman 的**默认参数**启用防护，`content_security_policy` 只是其中之一。这些默认项一般都能提升安全性，但管理员应当知道它们的存在。

尤其是 `force_https = True`（默认 `False`）：如果 worker 通过以 `http://` 开头的 `WEBDRIVER_BASEURL` 访问图表，它**可能破坏 Alerts & Reports**。只要部署在上游（如负载均衡或应用网关）强制了 HTTPS，保持该项禁用是可以接受的；否则可以这样开启：

```python
TALISMAN_CONFIG = {
    "force_https": True,
    "content_security_policy": {
        # ...
    },
}
```

### 3. 配置示例

在 `superset_config.py` 中覆盖默认配置即可。下面是官方给出的示例（从 S3 等外部来源加载图片）：

```python
TALISMAN_CONFIG = {
    "content_security_policy": {
        "base-uri": ["'self'"],
        "default-src": ["'self'"],
        "img-src": [
            "'self'",
            "blob:",
            "data:",
            "https://apachesuperset.gateway.scarf.sh",
            "https://static.scarf.sh/",
            # "https://cdn.brandfolder.io",  # SLACK_ENABLE_AVATARS 为 True 时取消注释
            "ows.terrestris.de",
            "aws.s3.com",
            # 在此添加你的 Bucket 或外部数据源
        ],
        "worker-src": ["'self'", "blob:"],
        "connect-src": [
            "'self'",
            "https://api.mapbox.com",
            "https://events.mapbox.com",
        ],
        "object-src": "'none'",
        "style-src": ["'self'", "'unsafe-inline'"],
        "script-src": ["'self'", "'strict-dynamic'"],
    },
    "content_security_policy_nonce_in": ["script-src"],
    "force_https": False,
    "session_cookie_secure": False,
}
```

> 想按需调整策略，参考官方的 Content Security Policy Reference，或参见 <https://superset.apache.org/admin-docs/configuration/networking-settings/#changing-flask-talisman-csp>。

## 十三、上报安全漏洞

Apache 软件基金会对消灭软件安全问题是**严格**态度，Superset 对自身功能相关的问题也高度敏感、乐于响应。

如果你对 Superset 的安全有疑虑，或发现了漏洞 / 潜在威胁：

- 发邮件到 **security@apache.org**；
- 邮件中写明项目名 **Superset**，并描述问题或潜在威胁；
- 建议一并给出**复现方式**。

安全团队与 Superset 社区评估分析后会回复。

> [!danger] 先私下报告，再公开
> 在公开披露之前，务必先发到安全邮箱。

ASF 安全团队维护了漏洞与潜在威胁**处理流程**的说明页面，可查阅细节。

## 十四、避坑清单

- **内置角色别改**：`superset init` 会把 Admin / Alpha / Gamma / sql_lab（以及 `PUBLIC_ROLE_LIKE = "Public"` 时的 Public）的权限同步回原值。
- **Admin 不是"普通管理员"**：它是受信任边界，用 Admin 权限能做到的事**不算漏洞**，所以别把 Admin 发给业务人员。
- **Public 只管界面**：它给的是 UI/API 权限，**数据权限必须另外授予**；否则仪表盘还是看不到。
- **Gamma 要配补充角色**：Gamma 本身不带你想要的任何数据源。
- **`DASHBOARD_RBAC` 会绕过数据集检查**：给了仪表盘，等于给了里面所有图表与数据集的读权限。
- **RLS 的 Base 是"除我之外"**：Regular 是"给这些角色"，Base 是"除这些角色以外的所有人"。
- **group_key 决定 AND 还是 OR**：同 key 用 OR，不同 key / 无 key 用 AND；想做"二选一"就给同一个 key，否则结果恒空。
- **虚拟数据集的 RLS 分两层**：直接挂在虚拟数据集上的加在外层查询，挂在底层物理数据集上的注入内层子查询。
- **SQL Lab 默认不套 RLS**：要开 `RLS_IN_SQLLAB`。
- **API Key 只显示一次**：复制不出来就得重新生成；泄露立刻删除。
- **`SECRET_KEY` 泄露等于会话可被伪造**：它是会话 cookie 的加密密钥。

## 相关

- [[60-数据与AI/10-数据分析/10-Superset/20-管理员手册/Superset 配置管理|Superset 配置管理]] —— `superset_config.py`、`SECRET_KEY`、OAuth2/LDAP、Feature Flags
- [[60-数据与AI/10-数据分析/10-Superset/10-用户使用手册/00-索引|10-用户使用手册]]
- [[60-数据与AI/10-数据分析/10-Superset/00-索引|10-Superset]]
- 官方原文：<https://superset.apache.org/admin-docs/6.1.0/security/>
