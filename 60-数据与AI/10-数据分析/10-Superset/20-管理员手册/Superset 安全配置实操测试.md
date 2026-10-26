---
title: Superset 安全配置实操测试
date: 2026-10-26
tags:
  - superset
  - 安全
  - 测试
  - 运维
---

# Superset 安全配置实操测试

> 按官方 6.1.0 安全文档逐点设计的**可执行测试手册**：44 个测试点，每条给出前置条件、操作步骤、预期结果与常见失败原因。
> 对应笔记：[[60-数据与AI/10-数据分析/10-Superset/20-管理员手册/Superset 安全配置|Superset 安全配置]]
> 原文：<https://superset.apache.org/admin-docs/6.1.0/security/>

**怎么用：**

1. 先做 **一、环境准备**（数据、账号、角色一次性建好）；
2. 按 **二、建议执行顺序** 分批跑，减少重启次数；
3. 每跑完一条，在 **三、测试矩阵** 的“结果”列记 ✅ / ❌ / N/A；
4. 失败时看该条的“排查”，绝大多数是**没重启**、**没刷新权限缓存**或**漏了 `superset init`**。

---

## 一、环境准备

### 1. 版本核对（T00）

```bash
# 容器内
docker compose exec superset superset version
# 或裸机
superset version
```

预期：`Superset 6.1.0`。**版本不同也能跑**，但请按实际版本的界面文案微调；差异较大时先看该版本的 Release Notes。

### 2. 配置文件与重启

所有涉及 `superset_config.py` 的测试都遵循同一套动作：

```bash
# 1) 改配置
vim /path/to/superset_config.py

# 2) 重启
docker compose restart superset          # 容器
# systemctl restart superset             # 裸机

# 3) 确认起来了
curl -s -o /dev/null -w '%{http_code}\n' http://localhost:8088/health
```

> [!warning] 两个最常见的“测试失败”
> 1. **改完没重启** —— 配置只在启动时读取；
> 2. **权限没刷新** —— 角色/权限改动后，用**无痕窗口**重新登录，避免旧会话里的权限缓存干扰。

### 3. 测试数据

在测试库里执行（PostgreSQL 语法；MySQL 请把 `serial` 换成 `auto_increment`、`numeric` 换成 `decimal`）：

```sql
DROP TABLE IF EXISTS sec_test_orders;

CREATE TABLE sec_test_orders (
    id          serial PRIMARY KEY,
    department  varchar(32)   NOT NULL,
    region      varchar(16)   NOT NULL,
    client_id   int           NOT NULL,
    amount      numeric(10,2) NOT NULL,
    created_at  date          NOT NULL,
    owner       varchar(32)   NOT NULL
);

INSERT INTO sec_test_orders
    (department, region, client_id, amount, created_at, owner)
VALUES
    ('Finance',   'US',     4, 100.00, '2026-01-10', 'u_finance'),
    ('Finance',   'Europe', 4, 200.00, '2026-02-11', 'u_finance'),
    ('Marketing', 'Europe', 5, 300.00, '2026-03-12', 'u_marketing'),
    ('Marketing', 'US',     5, 400.00, '2026-04-13', 'u_marketing'),
    ('Ops',       'US',     6, 500.00, '2026-05-14', 'u_ops'),
    ('Ops',       'Europe', 6, 600.00, '2026-06-15', 'u_ops');
```

**这 6 行数据是后面所有断言的基准：**

| 过滤条件 | 期望行数 |
| --- | --- |
| 无条件 | 6 |
| `department = 'Finance'` | 2 |
| `region = 'US'` | 3 |
| `department <> 'Ops'` | 4 |
| 部门是 Finance 或 Marketing，**且** region 为 Europe | 2 |
| `client_id = 4` | 2 |
| `client_id = 5` | 2 |
| `client_id = 4 AND client_id = 5` | 0 |
| `client_id = 4 OR client_id = 5` | 4 |
| `owner = 'u_finance'` | 2 |

在 Superset 里把它注册为数据集：

1. **Settings → Database Connections** 确认已有一个可用连接（没有就 **+ Database** 添加）；
2. **Data → Datasets → + Dataset**，选该连接与 `public.sec_test_orders`；
3. 保存后进 **Explore**，建一个最简单的 **Table** 图表，命名为 `probe_orders`，保存；
4. 之后所有 RLS 测试都复用这个 `probe_orders`，用 **View query** 看生成 SQL、用结果行数做断言。

### 4. 测试账号

统一密码：`Test@12345`（仅测试环境）。全部通过 **Settings → List Users → +** 创建，或按下面 API 批量建。

| 账号 | 角色 | 用途 |
| --- | --- | --- |
| `u_alpha` | Alpha | T04 |
| `u_gamma` | Gamma | T05、T06 |
| `u_finance` | Gamma + `r_ds_orders` | T06、T26–T31、T36 |
| `u_marketing` | Gamma + `r_ds_orders` | T30（Jinja）、T29 |
| `u_sql` | Gamma + sql_lab（+ 后期加库权限） | T07、T08、T17 |
| `u_api` | Gamma | T25 |
| `u_nods` | Gamma + `r_nods` | T12 |

### 5. 测试角色

| 角色 | 权限 | 用途 |
| --- | --- | --- |
| `r_ds_orders` | 仅 `datasource access on ...sec_test_orders` | T06、T10–T13、RLS |
| `r_nods` | **空**（无任何数据集权限） | T12 |
| `r_db` | `database access on [测试库]` | T17 |

创建方式：**Settings → List Roles → +**，填名称，在 **Permissions** 下拉里搜索 `datasource` / `database` / `menu access` 来勾选。

### 6. 调用 REST API

先拿 token（Superset 自身的 JWT，不是 API Key）：

```bash
BASE=http://localhost:8088

TOKEN=$(curl -s -X POST "$BASE/api/v1/security/login" \
  -H 'Content-Type: application/json' \
  -d '{"username":"admin","password":"admin","provider":"db","refresh":true}' \
  | jq -r .access_token)

echo "$TOKEN" | head -c 40; echo
```

之后统一带上：

```bash
curl -s -H "Authorization: Bearer $TOKEN" "$BASE/api/v1/security/roles/" | jq '.count'
```

> 用 Bearer token 时**不需要 CSRF token**；用 cookie 会话调写接口才需要。这是排查 401/403 时首先要分清的事。

### 7. 先知道：两处“文档与源码不一致”

跑测试前先记住这两条，否则会把**正常现象**记成 bug：

| 配置 | 文档说法 | 6.1.0 `superset/config.py` 实值 |
| --- | --- | --- |
| `TALISMAN_ENABLED` | security 篇：默认 `True`；securing 篇：4.0+ 默认 `False` | `os.environ.get("TALISMAN_ENABLED", True)` → **默认 True** |
| `FAB_ADD_SECURITY_API` | “默认不启用” | `FAB_ADD_SECURITY_API = True` → **默认启用** |
| `SESSION_COOKIE_HTTPONLY` | 默认 `False` | `True` |

**处理原则：以实测为准，把实测值记进结果表。** 这三条本身就是好的测试点（T22、T35、T39）。

---

## 二、建议执行顺序

| 阶段 | 测试 | 需要改配置？ |
| --- | --- | --- |
| **阶段 1：默认配置** | T01–T10、T19–T22、T25、T35、T37、T39–T41 | 否（除 T09/T10 需开 Public） |
| **阶段 2：开 `DASHBOARD_RBAC`** | T11–T14 | 是，重启一次 |
| **阶段 3：自定义权限** | T15–T18 | 否 |
| **阶段 4：RLS** | T26–T34 | T30/T32 需开 flag，重启一次 |
| **阶段 5：会话与 CSP** | T36、T38、T42–T43 | 是，重启多次（会踢掉所有会话） |

> 阶段 5 会让**所有已登录会话失效**，放在最后做。

---

## 三、测试矩阵

> “结果”列自己填：✅ 通过 / ❌ 失败 / ⚠️ 与预期不同但可解释 / N/A 不适用。

| 编号 | 测试点 | 对应章节 | 结果 |
| --- | --- | --- | --- |
| T01 | 5 个内置角色都存在 | 二 | |
| T02 | `superset init` 会重置内置角色权限 | 二 | |
| T03 | Admin 拥有全部权限 | 二.1 | |
| T04 | Alpha 全数据源、但不能授权他人 | 二.2 | |
| T05 | Gamma 不能新增/修改数据源 | 二.3 | |
| T06 | Gamma 只看到被授予的数据源与对象 | 二.3 | |
| T07 | `sql_lab` 与按库授权 | 二.4 | |
| T08 | SQLLab 两个附加权限 | 二.4 | |
| T09 | Public 未认证只读的边界 | 二.5 | |
| T10 | Public 仍需要数据集权限 | 二.5 | |
| T11 | `DASHBOARD_RBAC` 基本生效 | 四.2 | |
| T12 | RBAC 绕过数据集检查 | 四.2 | |
| T13 | 未分配角色的仪表盘回退 | 四.2 | |
| T14 | 未发布仪表盘不可见 | 四 | |
| T15 | Views 权限控制菜单与页面 | 八 | |
| T16 | Model & Action 权限 | 八 | |
| T17 | Database 权限 | 八 | |
| T18 | 多角色并集与访问画像 | 九 | |
| T19 | `DISALLOWED_SQL_FUNCTIONS` 默认清单生效 | 五 | |
| T20 | 自定义禁用清单 | 五 | |
| T21 | `get_samples` 的数据源权限检查 | 五.4 | |
| T22 | `FAB_ADD_SECURITY_API` 的实际默认值 | 六 | |
| T23 | 用户/角色 REST API 的 CRUD | 六 | |
| T24 | API Key 启用与“只显示一次” | 七 | |
| T25 | API Key 权限跟随创建者、可撤销 | 七 | |
| T26 | RLS Regular 生效 | 十.1 | |
| T27 | RLS Base 生效（Admin 豁免） | 十.1 | |
| T28 | 同 group key 用 OR | 十.2 | |
| T29 | 无 group key 用 AND（冲突→空） | 十.2 | |
| T30 | RLS 支持 Jinja 动态过滤 | 十 | |
| T31 | 虚拟数据集的双层注入 | 十.3 | |
| T32 | `RLS_IN_SQLLAB` 开关对比 | 十.3 | |
| T33 | RLS API 审计 | 十.4 | |
| T34 | 无权限用户访问 RLS API | 十.4 | |
| T35 | 会话 cookie 的标志位 | 十一.1 | |
| T36 | 服务端会话：登出即失效 | 十一.2 | |
| T37 | 换 `SECRET_KEY` 使旧会话失效 | 十一 | |
| T38 | `PERMANENT_SESSION_LIFETIME` 生效 | 十一.1 | |
| T39 | CSP 响应头存在 | 十二 | |
| T40 | nonce 存在且每次不同 | 十二.1 | |
| T41 | 无 nonce 的内联脚本被阻止 | 十二.1 | |
| T42 | `style-src 'unsafe-inline'` 是必需的 | 十二.1 | |
| T43 | MapBox / Cartodiagram 的 CSP 指令 | 十二.1 | |
| T44 | 漏洞上报流程（流程类，不实操） | 十三 | |

---

## 四、角色与权限

### T01 五个内置角色都存在

**前置**：无。
**步骤**：

1. **Settings → List Roles**；
2. 或调 API：`curl -s -H "Authorization: Bearer $TOKEN" "$BASE/api/v1/security/roles/?q=(page_size:100)" | jq -r '.result[].name' | sort`。

**预期**：至少包含 `Admin`、`Alpha`、`Gamma`、`sql_lab`、`Public`。

**排查**：角色列表因权限不足为空 → 你用的 token 不是 Admin。

---

### T02 `superset init` 会重置内置角色权限

**目的**：验证“不要改内置角色”这条结论，而不是只相信文档。

**步骤**：

1. **Settings → List Roles → Gamma**，在 Permissions 里搜索并添加 `can_delete on Dashboard`；
2. 保存，确认 Gamma 角色下能看到这条权限；
3. 执行同步：

```bash
docker compose exec superset superset init
docker compose restart superset
```

4. 重新登录，再打开 **Gamma** 的权限列表，搜索 `can_delete on Dashboard`。

**预期**：该权限**已消失**（回到原始权限集）。

> [!tip] 反向验证
> 改的是一个**自建角色**（如 `r_ds_orders`）→ 跑完 `superset init` 后**权限仍在**。这一正一反才证明“只有内置角色被同步”。

---

### T03 Admin 拥有全部权限

**步骤**：

1. 用 `u_alpha` 登录，随便改一个图表并保存；
2. 换 `admin` 登录，打开同一个图表 → 修改 → 保存；
3. 用 admin 调 `GET /api/v1/security/roles/` 并尝试 `PUT` 某角色。

**预期**：三步全部成功。Admin 能改**别人**的对象、能给他人授权撤销。

**排查**：改别人对象失败 → 检查该对象的所有者设置，或你的 admin 账号其实只有 Alpha。

---

### T04 Alpha 全数据源、但不能授权

**步骤**：

1. `u_alpha` 登录 → **Data → Datasets**：应列出**全部**数据集；
2. 点 **+ Dataset** 新增一个（可删）→ 应成功；
3. 打开 **Settings** 菜单：**List Roles / List Users** 应**不可见**；
4. 用 API 验证越权：

```bash
ALPHA_TOKEN=$(curl -s -X POST "$BASE/api/v1/security/login" \
  -H 'Content-Type: application/json' \
  -d '{"username":"u_alpha","password":"Test@12345","provider":"db"}' | jq -r .access_token)

curl -s -o /dev/null -w '%{http_code}\n' \
  -H "Authorization: Bearer $ALPHA_TOKEN" "$BASE/api/v1/security/roles/"
```

**预期**：1、2 成功；3 菜单隐藏；4 返回 **403**。

---

### T05 Gamma 不能新增/修改数据源

**步骤**：

1. `u_gamma` 登录；
2. 直接访问 `http://localhost:8088/tablemodelview/list/`；
3. 直接访问 `http://localhost:8088/dataset/add/`；
4. 检查左侧菜单是否有 **Datasets**。

**预期**：页面提示无权限 / 被重定向登录或无内容；菜单里**没有** Datasets 入口。

**排查**：能打开 → 该账号被额外授了 Dataset 的 `can_read/can_write`，或它不是纯 Gamma。

---

### T06 Gamma 只看到被授予的数据源与对象

**步骤**：

1. 用 `u_gamma`（纯 Gamma）登录 → **Dashboards** 列表：应为空；
2. 用 `u_finance`（Gamma + `r_ds_orders`）登录 → **Dashboards** 列表：只出现用 `sec_test_orders` 做的**已发布**仪表盘；
3. `u_finance` 进 **Explore** → 数据集下拉里只有 `sec_test_orders`；
4. `u_finance` 基于它建图表并保存 → 应成功。

**预期**：1 空；2、3 只有授权对象；4 成功。

> 这一步同时验证了 **三、给 Gamma 用户开放指定数据源** 的完整链路。

---

### T07 `sql_lab` 与按库授权

**步骤**：

1. `u_finance`（无 `sql_lab`）直接访问 `http://localhost:8088/sqllab/` → 应被拒；
2. `u_sql`（Gamma + `sql_lab`，**未给库权限**）登录 → SQL Lab 能打开，但左侧 **Database** 下拉为空；
3. 给 `u_sql` 加上 `r_db`（`database access on [测试库]`），重新登录；
4. 在 SQL Lab 执行 `SELECT 1 AS ok;`。

**预期**：1 拒绝；2 下拉为空；4 能查到结果。

**排查**：2 里下拉不为空 → 该用户有 `all_datasource_access` 或别的库权限。

---

### T08 SQLLab 的两个附加权限

**步骤**：

1. `u_sql` 在 SQL Lab 打开编辑器，截图记录工具栏：应有 **Run**，**没有** Estimate cost、**没有** Format SQL；
2. 给 `u_sql` 的角色加上 `can_estimate_query_cost on SQLLab` 与 `can_format_sql on SQLLab`：

> 权限名在 Permissions 下拉里搜 `SQLLab` 即可看到。

3. 退出重登，刷新 SQL Lab。

**预期**：两个按钮出现；点 Estimate cost 返回成本/行数估计；点 Format SQL 把乱糟糟的 SQL 格式化。

**排查**：加了权限仍不显示 → 没重新登录（前端菜单按登录时的权限渲染），或加到了别的角色上。

---

### T09 Public 的未认证只读边界

**前置**：`superset_config.py` 加一行，重启：

```python
AUTH_ROLE_PUBLIC = "Public"
```

**步骤**：

1. 准备一个**已发布**的仪表盘（用 `sec_test_orders`），假设 URL 为 `/superset/dashboard/<slug>/`；
2. 开**无痕窗口**（确保未登录）访问该 URL；
3. 在同一无痕窗口访问 `/sqllab/`；
4. 访问 `/userinfo/`；
5. 找页面上的 **Share** 按钮；
6. 尝试无 token 调写接口：

```bash
curl -s -o /dev/null -w '%{http_code}\n' -X POST "$BASE/api/v1/chart/" \
  -H 'Content-Type: application/json' -d '{"slice_name":"hack"}'
```

**预期**：

| 步骤 | 预期 |
| --- | --- |
| 2 | 能看图（前提是 T10 的数据权限已给） |
| 3 | 跳到登录页 |
| 4 | 跳到登录页或无权限 |
| 5 | **没有** Share 按钮 |
| 6 | **401** |

> 若第 2 步是空白/报错，先去做 T10——Public 只有 UI 权限，**没有数据权限时图表是空的**。

---

### T10 Public 仍然需要数据集权限

**步骤**：

1. 确认 Public 角色**没有**任何 datasource 权限（Settings → List Roles → Public → Permissions 搜 `datasource`）；
2. 无痕访问已发布仪表盘的**直链**；
3. 给 Public 加上 `datasource access on ...sec_test_orders`；
4. 再访问同一 URL（必要时强刷）。

**预期**：2 → 看不到数据 / 提示无权限；4 → 正常显示。

**排查**：第 4 步仍空 → 仪表盘没发布，或图表用了别的基础数据集。

---

## 五、仪表盘访问控制

### T11 `DASHBOARD_RBAC` 基本生效

**前置**：

```python
FEATURE_FLAGS = {
    "DASHBOARD_RBAC": True,
}
```

重启。

**步骤**：

1. 准备两个已发布仪表盘：`D1`（用 `sec_test_orders`）、`D2`（用另一个数据集）；
2. 打开 `D1` → **Edit → Properties**，在 **Roles** 里加上 `r_ds_orders`（6.x 在仪表盘属性面板里的 “Access” / “Roles” 项）；
3. 用 `u_finance`（有 `r_ds_orders`）登录 → Dashboards 列表；
4. 用 `u_gamma`（无该角色）登录 → Dashboards 列表。

**预期**：`u_finance` 能看到 `D1`；`u_gamma` 看不到。

**排查**：改了 Roles 但没生效 → 确认 flag 拼写是 `DASHBOARD_RBAC`，并**重启**；仪表盘还没发布也不行。

---

### T12 RBAC 会绕过数据集检查

**目的**：验证“给仪表盘 = 给里面所有图表与数据集的读权限”。

**步骤**：

1. 用 `u_nods`（Gamma + `r_nods`，**没有任何数据集权限**）登录 → 该用户看不到任何数据集；
2. 在仪表盘 `D1` 的 Roles 里**加上** `r_nods`；
3. `u_nods` 重新登录，打开 `D1`。

**预期**：图表**正常渲染**，尽管该用户对 `sec_test_orders` 毫无权限。

> 这是 RBAC 的“特性也是风险点”：`D1` 上新增的图表会自动被该角色看到。

---

### T13 未分配角色的仪表盘回退

**步骤**：

1. 新建仪表盘 `D3`（用 `sec_test_orders`），**不在 Properties 里配任何角色**，发布；
2. `u_finance` 登录看列表。

**预期**：能看到 `D3`（回退到“基于数据集”的规则）。

---

### T14 未发布仪表盘不可见

**步骤**：

1. 把 `D3` 取消发布（Edit → 关闭 Published）；
2. `u_finance` 重新登录 → 列表与直链；
3. 用 admin 打开同一 URL。

**预期**：2 → 看不到、直链也不可访问；3 → admin 正常打开。

---

## 六、自定义权限

### T15 Views 权限控制菜单与页面

**步骤**：

1. **Copy Role** 把 `Gamma` 复制成 `r_gamma_nosql`；
2. 在 `r_gamma_nosql` 里**删除** `menu access on SQL Lab`（Permissions 里搜 `SQL Lab`，菜单类权限形如 `menu access on ...`）；
3. 建用户 `u_nosql`，给 `r_gamma_nosql` + `sql_lab` 角色；
4. 登录：检查左侧菜单是否有 **SQL Lab**；
5. 直链访问 `/sqllab/`。

**预期**：4 → 菜单里没有 SQL Lab；5 → 无权限提示或被拦。

**排查**：5 仍能打开 → 视图权限只隐藏菜单，页面权限由别的权限项控制；把实际现象记进结果表（这本身就是有价值的发现）。

---

### T16 Model & Action 权限

**步骤**：

1. 建角色 `r_gamma_del`（复制 Gamma），加 `can_delete on Dashboard`；
2. 建用户 `u_del`，给 `r_gamma_del` + `r_ds_orders`；
3. `u_del` 登录 → Dashboards 列表 → 勾选一个自己建的仪表盘 → **Delete**；
4. 从角色里**移除** `can_delete on Dashboard`，重登，重复删除。

**预期**：3 成功；4 失败（按钮不可用或接口返回 403）。

---

### T17 Database 权限

**步骤**：

1. 给 `u_sql` 只加 `r_ds_orders`（单表权限），**不加** `r_db`；
2. SQL Lab 里 `SELECT * FROM sec_test_orders LIMIT 5;` → 成功；`SELECT * FROM 别的表 LIMIT 5;` → 失败；
3. 改为加 `r_db`（`database access on [测试库]`），重登；
4. 再查“别的表” → 成功，且 Database 下拉里能看到该库。

**预期**：2 只能查授权表；4 全库可查。

---

### T18 多角色并集

**步骤**：

1. 建 `r_ds_other`（另一个数据集/库的权限），加给 `u_finance`；
2. 重登，检查 Datasets、Dashboard、SQL Lab 的可见集合。

**预期**：可见范围是**各角色权限的并集**，任何单个角色的权限都不够解释全部可见项。

---

## 七、SQL 执行安全

### T19 默认禁用清单生效

**前置**：6.1.0 的 `DISALLOWED_SQL_FUNCTIONS` **自带一份非空默认清单**（postgresql 含 `version`、`pg_sleep`、`pg_read_file` 等），无需配置。

**步骤**：在 SQL Lab 依次执行：

```sql
SELECT 1;            -- A
SELECT version();    -- B
SELECT pg_sleep(1);  -- C
```

**预期**：A 成功；B、C 被拒绝，报错里出现“disallowed”（例如 `Disallowed function: version`）。

**排查**：B 居然成功 → 你的连接引擎不是 postgresql（清单按**引擎名**分组，MySQL 的清单里是 `version`、`sleep` 等），或你覆盖了该配置却没合并默认值。

---

### T20 自定义禁用清单

**步骤**：

1. 在 `superset_config.py` 里**复制完整字典**后修改（配置项是整体覆盖，不是追加）：

```python
DISALLOWED_SQL_FUNCTIONS = {
    "postgresql": {
        # 去掉 version → 让它可用
        "pg_sleep",
        "pg_read_file",
    },
}
```

2. 重启，执行 `SELECT version();` → **应成功**；
3. 再改成禁用 `lower`：

```python
DISALLOWED_SQL_FUNCTIONS = {"postgresql": {"lower"}}
```

4. 重启，执行 `SELECT lower('ABC');` → **应被拒**。

**预期**：2 成功、4 被拒。这证明清单是**由配置驱动、整体覆盖**的。

> 注意：这里的 `{"postgresql": {...}}` 结构以 6.1.0 的 `config.py` 为准；如果你的版本是集合而非字典，按注释改。

---

### T21 `get_samples` 的数据源权限检查

**步骤**：

1. 用 admin 打开 Swagger（`/swagger/v1`），搜 **Dataset** 下的 samples 端点，记下实际路径（6.x 形如 `/api/v1/dataset/{pk}/samples`）；
2. 取数据集 id：

```bash
curl -s -H "Authorization: Bearer $TOKEN" \
  "$BASE/api/v1/dataset/?q=(page_size:100)" \
  | jq -r '.result[] | select(.table_name=="sec_test_orders") | .id'
```

3. 用 `u_gamma`（无数据集权限）的 token 调该端点；
4. 用 `u_finance` 的 token 调同一端点。

**预期**：3 → **403**；4 → 200 并返回样例行。

**排查**：3 返回 200 → 该用户有 `all_datasource_access`，或你测的是别的（旧的、不带权限检查的）端点。

---

## 八、REST API 与 API Key

### T22 `FAB_ADD_SECURITY_API` 的实际默认值

**步骤**：

1. **不改配置**，打开 `/swagger/v1`，展开 **Security** 分类，把端点清单记下来（截图或复制路径）；
2. 在配置里显式写 `FAB_ADD_SECURITY_API = False`，重启，再记录一次；
3. 改回 `True`，重启，再记录一次；
4. 对比三份清单。

**预期**：`False` 时 Security 分类下的端点**减少**，`True` 时**恢复**。按 6.1.0 源码默认是 `True`，所以第 1 步应等同于第 3 步。

> [!warning] 别用错探测端点
> `/api/v1/security/roles/`、`/api/v1/security/users/` 是 **Superset 自己的** Security API，**不受** `FAB_ADD_SECURITY_API` 控制——拿它探测会得出错误结论。只能对比 Swagger 中的端点清单。

> 官方 security 篇写“默认不启用”，与 6.1.0 的 `config.py`（`FAB_ADD_SECURITY_API = True`）不一致——这正是要实测的理由。

---

### T23 用户/角色 REST API 的 CRUD

**步骤**（全部用 admin token）：

```bash
# 查
curl -s -H "Authorization: Bearer $TOKEN" "$BASE/api/v1/security/users/?q=(page_size:5)" | jq '.count'

# 建
curl -s -X POST "$BASE/api/v1/security/users/" \
  -H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json' \
  -d '{"first_name":"Api","last_name":"User","username":"u_apitest",
       "email":"u_apitest@example.com","password":"Test@12345",
       "active":true,"roles":[4]}' | jq '.result.id'
```

**预期**：查得到、建成功、能在 UI 的 List Users 里看到新用户。把返回的 id 记下来，后续可 `PUT` 改角色、`DELETE` 删除。

**排查**：404 → `FAB_ADD_SECURITY_API` 被关了（见 T22）。

---

### T24 API Key 启用与“只显示一次”

**前置**：

```python
FAB_API_KEY_ENABLED = True

FEATURE_FLAGS = {
    "FAB_API_KEY_ENABLED": True,
}
```

重启。

**步骤**：

1. 右上用户菜单 → **Info** → 展开 **API Keys**；
2. **+ API Key** → 填名称 `k1`，过期时间留空 → 生成；
3. 复制 token，记下前 8 位；
4. **刷新页面**，再看 API Keys 区域。

**预期**：1 区域存在（配置项控制后端蓝图，feature flag 控制 UI）；3 拿到 token；4 **看不到明文**，只能看到名称与过期时间。

**排查**：区域不出现 → 两个开关缺一个，或没重启。

---

### T25 API Key 的权限与撤销

**步骤**：

1. 用 `u_api`（Gamma）创建 key `k_gamma`；
2. 用该 key 调接口：

```bash
curl -s -o /dev/null -w '%{http_code}\n' \
  -H "Authorization: Bearer <k_gamma>" "$BASE/api/v1/security/users/"
```

3. 在同一请求里调用它**有权**的接口（如 `/api/v1/dashboard/`）；
4. 用 admin 创建 `k_admin`，重复第 2 步；
5. 回到 User Info → API Keys → 删除 `k_gamma`，再重复第 3 步。

**预期**：2 → **403**（以创建者权限执行）；3 → 200；4 → 200；5 → **401**。

**排查**：2 返回 200 → 该 Gamma 用户被额外授权；5 仍 200 → 前端缓存了 token，换一个新的 curl 进程再试。

---

## 九、行级安全（RLS）

> **统一做法**：RLS 过滤器分配给**数据集 + 角色**。所有断言都用 `probe_orders`（T00 建的 Table 图表），看**行数**与 **View query 里的 WHERE**。
>
> ⚠️ 若开启了结果缓存，先关掉或强制刷新，否则看到的是别的用户/上一次的缓存结果。

### T26 Regular 过滤器

**步骤**：

1. **Settings → Row Level Security → +**；
2. 填：Name=`rls_finance`、Dataset=`sec_test_orders`、**Filter Type=`Regular`**、Clause=`department = 'Finance'`、Roles=`r_ds_orders`；
3. 保存；
4. `u_finance` 重登，打开 `probe_orders`；
5. admin 打开同一个图表。

**预期**：`u_finance` → **2 行**；admin → **6 行**（admin 不在 `r_ds_orders` 里，Regular 不生效）。

**排查**：两边都是 6 → 角色没选对，或用户实际没拿到 `r_ds_orders`；两边都是 2 → 过滤器选成了 Base。

---

### T27 Base 过滤器（默认拒绝）

**步骤**：

1. 新建 RLS：Name=`rls_deny_all`、Dataset=`sec_test_orders`、**Filter Type=`Base`**、Clause=`1 = 0`、Roles=`Admin`；
2. `u_finance` 打开图表；
3. admin 打开图表；
4. 删除该过滤器。

**预期**：`u_finance` → **0 行**（Base 对“不在 Admin 角色里的人”生效）；admin → **6 行**（在角色里 → 豁免）。

> 这就是“默认拒绝 + 特权豁免”的基线模式。

---

### T28 同 group key 用 OR

**步骤**：

1. 删掉 T26 的过滤器，改建成三条，全部 **Regular**、角色 `r_ds_orders`、数据集 `sec_test_orders`：

| Name | Clause | Filter Type | Group Key |
| --- | --- | --- | --- |
| f1 | `department = 'Finance'` | Regular | `department` |
| f2 | `department = 'Marketing'` | Regular | `department` |
| f3 | `region = 'Europe'` | Regular | `region` |

> Group Key 字段在 RLS 表单里（部分版本折叠在高级项中）。

2. `u_finance` 打开图表 → 点 **View query**。

**预期**：**2 行**（Finance/Europe、Marketing/Europe），SQL 形如：

```sql
WHERE (department = 'Finance' OR department = 'Marketing')
  AND (region = 'Europe')
```

**排查**：行数 0 或 4 → Group Key 没填/填错，退化成 AND 或全 OR。

---

### T29 无 group key 用 AND（冲突 → 空）

**步骤**：

1. 清空上面的过滤器，新建两条：`client_id = 4`、`client_id = 5`，都 **Regular**、`r_ds_orders`、Group Key **留空**；
2. `u_finance` 看图表；
3. 把两条的 Group Key 都设为 `client`，再看。

**预期**：2 → **0 行**（AND 成恒假）；3 → **4 行**（同 key → OR）。

---

### T30 RLS + Jinja 动态过滤

**前置**：

```python
FEATURE_FLAGS = {
    "ENABLE_TEMPLATE_PROCESSING": True,
}
```

重启。

**步骤**：

1. 清空前面的过滤器，新建一条：Clause=`owner = '{{ current_username() }}'`、Regular、`r_ds_orders`、数据集 `sec_test_orders`；
2. `u_finance` 看图表；`u_marketing` 看图表（两账号都要有 `r_ds_orders`）。

**预期**：各自 **2 行**，且 `View query` 里的 clause 已渲染成各自用户名（不再是 `{{ }}`）。

**排查**：报 Jinja 错误或行数为 0 → `ENABLE_TEMPLATE_PROCESSING` 没开；见字面量 `{{ }}` → 同上。

---

### T31 虚拟数据集的双层注入

**步骤**：

1. 在物理数据集 `sec_test_orders` 上加 RLS：`region = 'US'`、Regular、角色 `r_ds_orders`；
2. 建**虚拟数据集** `v_orders`：SQL 为 `SELECT * FROM sec_test_orders WHERE department <> 'Ops'`；
3. 用 `v_orders` 建图表 `probe_virtual`，`u_finance` 打开 → **View query**；
4. 再在 `v_orders` 上**直接**加一条 RLS：`department = 'Finance'`、Regular、`r_ds_orders`；
5. 重开图表看结果。

**预期**：

- 步骤 3：外层是虚拟 SQL，**内层**子查询里出现 `region = 'US'`；结果 **2 行**（Finance/US、Marketing/US）。
- 步骤 5：**外层**再套 `department = 'Finance'`；结果 **1 行**（Finance/US）。

**排查**：只有外层或只有内层 → 检查两条 RLS 分别挂在哪个数据集上（物理 vs 虚拟）。

---

### T32 `RLS_IN_SQLLAB` 开关对比

**步骤**：

1. 保持 T30 的 Jinja 过滤器生效；
2. 确认 `FEATURE_FLAGS` 里 **没有** `"RLS_IN_SQLLAB": True`（默认 False），重启；
3. `u_finance` 在 SQL Lab 执行 `SELECT count(*) FROM sec_test_orders;`；
4. 加上 `"RLS_IN_SQLLAB": True`，重启；
5. 同样执行。

**预期**：3 → **6**（SQL Lab 不套 RLS）；5 → **2**（套上了）。

**排查**：5 仍 6 → 该表没有对应数据集，或 RLS 挂在了别的数据集上。

---

### T33 RLS API 审计

**步骤**（admin token）：

```bash
# 全部
curl -s -H "Authorization: Bearer $TOKEN" "$BASE/api/v1/rowlevelsecurity/?q=(page_size:100)" | jq '.count'

# 按数据集
curl -s -H "Authorization: Bearer $TOKEN" \
  "$BASE/api/v1/rowlevelsecurity/?q=(filters:!((col:tables,opr:rel_m_m,value:<dataset_id>)))" | jq '.count'

# 按角色
curl -s -H "Authorization: Bearer $TOKEN" \
  "$BASE/api/v1/rowlevelsecurity/?q=(filters:!((col:roles,opr:rel_m_m,value:<role_id>)))" | jq '.count'

# 详情
curl -s -H "Authorization: Bearer $TOKEN" "$BASE/api/v1/rowlevelsecurity/<id>" \
  | jq '.result | {name, filter_type, clause, group_key, tables, roles}'
```

**预期**：列表数量与 UI 一致；详情里 `filter_type`、`clause`、`group_key`、`tables`、`roles` 都有值。

---

### T34 无权限用户访问 RLS API

**步骤**：用 `u_finance` 的 token 调 `/api/v1/rowlevelsecurity/`。

**预期**：**403**（需要 `Row Level Security` 的 `can_read`）。

---

## 十、用户会话

### T35 会话 cookie 的标志位

**步骤**：

1. 登录后打开 DevTools → **Application → Cookies**，找 `session`（或 `session_<hash>`）；
2. 记录 `HttpOnly`、`Secure`、`SameSite`、`Expires`；
3. 对照 `config.py`：`SESSION_COOKIE_HTTPONLY` 默认 `True`、`SESSION_COOKIE_SECURE` 默认 `False`、`SESSION_COOKIE_SAMESITE` 默认 `Lax`。

**预期**：HttpOnly ✓；SameSite=Lax；`Secure` 在 http 下为 ✗（这是正常的——设 `True` 后**必须**走 HTTPS，否则浏览器不发 cookie，你会登录不上）。

> [!warning] 本地 http 环境的坑
> 想测 `SESSION_COOKIE_SECURE = True`，先给 Superset 套 HTTPS（反代或自签证书），否则只会得到“登录后又被踢回登录页”。

---

### T36 服务端会话：登出即失效

**前置**（需要 Redis）：

```python
SESSION_SERVER_SIDE = True
SESSION_TYPE = "redis"
SESSION_REDIS = {"host": "redis", "port": 6379}
```

**步骤**：

1. **浏览器**登录 Superset，DevTools → Application → Cookies，复制 `session` 的值（记作 `SID`）；
2. 在 Redis 里看会话键：

```bash
docker compose exec redis redis-cli --scan --pattern 'session:*'
```

3. 用复制出来的 cookie 直接调接口，确认可用：

```bash
curl -s -o /dev/null -w '%{http_code}\n' \
  -H "Cookie: session=$SID" "$BASE/api/v1/me/"
```

4. 在浏览器里点**登出**；
5. 再用**同一个** `$SID` 调一次第 3 步的接口。

**预期**：3 → **200**；登出后 Redis 里对应键消失；5 → **401**。

**排查**：

- 5 仍是 200 → 没真正启用服务端会话（`SESSION_SERVER_SIDE` 未生效或没重启），此时走的是无状态 cookie；
- 3 就 401 → 复制 cookie 时漏了前缀，或站点在 HTTPS 下而你先前的 cookie 标了 `Secure`。

> 不要用 `curl -X POST "$BASE/login/"` 直接登录：表单登录需要 CSRF token，用浏览器拿 cookie 更省事。

---

### T37 换 `SECRET_KEY` 使旧会话失效

**步骤**：

1. 登录，保存 cookie 文件；
2. 改 `SECRET_KEY` 为一个新值，重启；
3. 用旧 cookie 调 `/api/v1/me/`。

**预期**：**401 / 跳登录**——旧 cookie 的签名无法验证。这从侧面验证了 `SECRET_KEY` 确实是会话的信任根。

> `SECRET_KEY` 同时用于加密元数据库里的连接凭证，所以这一改动还会影响已保存的数据库密码（生产轮换请严格按官方流程）。

---

### T38 会话超时

**步骤**：

1. 设 `PERMANENT_SESSION_LIFETIME = timedelta(minutes=1)`（记得 `from datetime import timedelta`），重启；
2. 登录，记下时间；
3. 等 70 秒，刷新页面 / 调 `/api/v1/me/`。

**预期**：需要重新登录（401 或跳转）。

**排查**：仍有效 → 该会话不是 permanent session；或用了服务端会话但后端没按 lifetime 清理。

---

## 十一、CSP 与安全响应头

### T39 CSP 响应头存在

**步骤**：

```bash
curl -sI "$BASE/" | grep -i 'content-security-policy'
```

**预期**：有输出，且包含 `script-src`、`style-src`、`object-src` 等指令。

**排查**：没有该头 → 你的 `TALISMAN_ENABLED` 为 `False`（6.1.0 默认 `True`，但如果有人显式关了就会这样）。

---

### T40 nonce 存在且每次不同

**步骤**：

1. 浏览器打开任意页面 → 右键**查看页面源代码**；
2. 搜索 `nonce=`，找到一个 `<script nonce="...">`；
3. 刷新页面，再搜一次。

**预期**：两次 nonce **不同**（Talisman 每次页面加载生成）。

---

### T41 无 nonce 的内联脚本被阻止

**步骤**：在页面 DevTools Console 里执行：

```javascript
document.body.insertAdjacentHTML('beforeend', '<script>alert("xss")<\/script>');
```

**预期**：**不弹窗**，Console 出现 CSP 拦截错误（`Refused to execute inline script ...`）。

> 注意：直接在 Console 里输入的代码**不受 CSP 限制**，只有“插入到 DOM 的内联脚本”才受控——所以必须用上面的插入方式测。

---

### T42 `style-src 'unsafe-inline'` 是必需的

**步骤**：

1. 在 `TALISMAN_CONFIG` 里把 `style-src` 的 `'unsafe-inline'` 去掉（记得**复制完整字典**再改）；
2. 重启，打开任意页面；
3. 恢复配置。

**预期**：页面样式明显崩坏（布局错乱、组件无样式）——这证明该指令不可省。

> 这条会让界面很难看，测完立刻恢复；不要在生产做。

---

### T43 MapBox / Cartodiagram 的 CSP 指令

**步骤**：

1. 配置 `MAPBOX_API_KEY`，建一个 MapBox 地图图表；
2. DevTools → Console / Network 观察；
3. 若 CSP 缺 `worker-src 'self' blob:` 或 `connect-src ... mapbox`，会看到 CSP 报错、地图不渲染；
4. 按安全配置篇的示例补齐指令后重试。

**预期**：补齐后地图正常、无 CSP 报错。

> Cartodiagram 需要 `img-src` / `connect-src` 允许外部地图资源域名（或 `'*'`），因为它会请求用户可编辑的外部资源。

---

## 十二、流程类（不可实操）

### T44 漏洞上报

**这不是测试，是流程确认**：

- 邮箱：`security@apache.org`，邮件里写明项目名 **Superset** 与问题描述、复现方式；
- **先私下报告，再公开**；
- 认知项：**严格依赖 Admin 权限的攻击向量不算漏洞**（Admin 是受信任边界，不分配 CVE）。所以内部发现“Admin 能改别人仪表盘”时，不要当成漏洞上报。

---

## 十三、清理

测完恢复环境，避免影响后续使用：

1. 删除测试用户、角色、数据集、图表、仪表盘、RLS 过滤器；
2. 恢复被改过的配置项（`FEATURE_FLAGS`、`AUTH_ROLE_PUBLIC`、`SESSION_*`、`TALISMAN_CONFIG`、`DISALLOWED_SQL_FUNCTIONS`）；
3. 重启并确认 `/health` 正常；
4. 若是临时容器环境：

```bash
docker compose down -v   # 会删除元数据库，谨慎
```

---

## 相关

- [[60-数据与AI/10-数据分析/10-Superset/20-管理员手册/Superset 安全配置|Superset 安全配置]] —— 本文测试点的原始说明
- [[60-数据与AI/10-数据分析/10-Superset/20-管理员手册/Superset 生产环境安全加固|Superset 生产环境安全加固]] —— 上线检查清单
- [[60-数据与AI/10-数据分析/10-Superset/20-管理员手册/Superset 配置管理|Superset 配置管理]] —— 配置项与重启方式
- 官方原文：<https://superset.apache.org/admin-docs/6.1.0/security/>
