---
title: Superset 创建第一个仪表盘
date: 2026-10-21
tags:
  - 数据可视化
  - superset
  - BI
---

# Superset 创建第一个仪表盘

> 基于 Superset **6.1.0** 官方用户文档，走通「连接数据库 → 注册数据集 → Explore 建图 → 保存到仪表盘」的端到端流程，并整理仪表盘、Table / AG Grid 图表与 URL 参数等进阶能力。
> 原文：<https://superset.apache.org/user-docs/6.1.0/using-superset/creating-your-first-dashboard>

## 一、总览

Superset 本身**没有存储层**，而是对接你已有的 SQL 数据库 / 数据仓库，在其上做查询与可视化。

两个主要界面：

| 界面 | 用途 |
| --- | --- |
| **Explore** | 无代码可视化构建器：选数据集 → 选图表 → 调样式 → 发布 |
| **SQL Lab** | SQL IDE，用于清洗、关联、准备数据 |

> [!tip] 本地 Docker Compose 环境
> 自带一个名为 `examples` 的 Postgres 库且已预配置，可跳过「连接数据库」这一步。

## 二、连接数据库

1. 右上角 `+` 菜单 → `Data` → `Connect Database`。
2. 在弹出的窗口中选择数据库类型。
3. 若不配置高级项，点击表单下方链接，只需填两项：**数据库名称**与 **SQLAlchemy URI**（URI 写法参考 SQLAlchemy 文档）。
4. 点 `Test Connection` 验证，通过后点 `Connect` 保存。

> [!warning] 连接本地/其它容器的数据库报 `The port is closed.`
> 把 HOST 改为 `host.docker.internal`。

**共享连接**：在连接窗口的 `Advanced` 标签中打开 `Share connection with other users`，其他用户即可在数据库列表看到该连接；之后也可编辑修改。

## 三、注册数据集（表）

Superset 中的「数据集（Dataset）」就是被暴露出来供查询的表：

1. `Data` → `Datasets` → 右上角 `+ Dataset`。
2. 在弹窗中选择 `Database`、`Schema`、`Table`。
3. 点 `Add`，数据集即出现在列表中。

**用文件夹整理**（个人用途）：

- Datasets 列表左侧 `Folders` 面板 → `+ New Folder` 建顶层文件夹，或把已有文件夹拖入嵌套。
- 把数据集行拖到文件夹上，或右键数据集 → `Move to folder`。
- 文件夹只是**个人组织手段**，不影响数据集权限，也不改变其他用户的视图。

## 四、自定义列属性

注册数据集后，可配置各列在 Explore 中的行为：

- 是否为**时间列**（temporal，用于时间序列的切片/切块）；
- 是否**可过滤**（filterable）；
- 是否为**维度**（dimensional）；
- 若是 datetime 列，按 ISO-8601 模式配置日期解析格式。

## 五、Superset 语义层

轻量语义层可存放两类计算数据：

| 类型 | 说明 | 是否允许聚合 |
| --- | --- | --- |
| **虚拟指标**（Virtual metrics） | 用 SQL 聚合多列，如 `SUM(recovered) / SUM(confirmed)`，作为指标用于可视化 | ✅ 允许且鼓励 |
| **虚拟计算列**（Virtual calculated columns） | 定制某列的外观/行为，如 `CAST(recovery_rate as float)` | ❌ 不允许 |

指标还可在该视图中「认证（certify）」，供团队识别可信指标。

## 六、在 Explore 创建图表

1. 从 `Datasets` 点击数据集名称进入 Explore。
2. 左侧 `Dataset` 面板列出当前数据集的列与指标；图表下方有数据预览。
3. 用 `Data` / `Customize` 两个标签切换图表类型、时间列、分组指标与视觉样式。
4. **每次改动后点 `Run`** 获取可视化反馈。

示例：全部通过下拉菜单操作，做出按产品线分组的季度销售 **Time-series Bar Chart**。

## 七、保存为 slice 并加入仪表盘

1. 点 `Save`，选择：
   - 保存并加入**已有**仪表盘，或
   - 保存并加入**新建**仪表盘。
2. 点 `Save and goto Dashboard` 完成发布。
3. 后台会创建一个 **slice**，把查询、图表类型、所选选项、名称等存入 Superset 的轻量数据层。

**调整图表大小**：点 `Edit Dashboard` → 拖动图表右下角，吸附到网格位置 → `Save`。

## 八、仪表盘管理与发布

| 功能 | 操作 |
| --- | --- |
| 访问控制 | 通过 owner 与权限管理；可用 `DASHBOARD_RBAC` 特性开关做仪表盘级角色控制 |
| 发布 | 点标题旁的 `Draft` 按钮；**草稿**仅 owner/admin 可见，**已发布**对所有有数据集权限（或 RBAC 授权角色）的用户可见 |
| 收藏 | 点标题旁星标，便于在仪表盘列表/首页查找 |
| 分享特定 Tab | 每个 Tab 有独立分享 URL，复制地址栏链接即可定位到该 Tab |
| 自动刷新 | 仪表盘右上 `⋮` → `Set auto-refresh interval`（如 10 秒 / 1 分钟 / 10 分钟）；**会话级**，关闭标签页即失效 |
| 最后查询时间 | `Dashboard Properties` → `Styling` → `Show last queried time` |
| 保存图表到指定 Tab | 从 Explore 保存时，在 `Add to dashboard` 弹窗中用 Tab 树选择目标 Tab |

> [!warning] 自动刷新的代价
> 自动刷新会对仪表盘上**所有图表触发全量数据重载**。查询较重的仪表盘请选较长间隔，避免压垮数据库。

## 九、仪表盘 URL 参数

| 参数 | 取值 | 效果 |
| --- | --- | --- |
| `standalone` | `0`（默认） | 正常显示 |
| | `1` | 隐藏顶部导航 |
| | `2` | 隐藏顶部导航 + 标题 |
| | `3` | 隐藏顶部导航 + 标题 + 顶层 Tab |
| `show_filters` | `0` | 不渲染筛选栏 |
| | `1`（默认） | 若启用原生筛选器则渲染筛选栏 |
| `expand_filters` | 默认 | 存在原生筛选器时展开 |
| | `0` | 收起筛选栏 |
| | `1` | 展开筛选栏 |

示例（本地开发环境，隐藏顶部导航并去掉筛选栏）：

```plain
http://localhost:8088/superset/dashboard/my-dashboard/?standalone=1&show_filters=0
```

## 十、Table 图表进阶

- **条件格式**：按值高亮单元格。支持数值列（阈值/渐变）、字符串列（文本/模式匹配）、布尔列（`true`/`false`/`null`）。每条规则有 `Use gradient` 开关：开启为渐变透明度，关闭为满不透明纯色。
- **单元格 HTML 渲染**：在列配置面板按列打开 `Render HTML`，可渲染链接、彩色徽标、图标等。
  > [!danger] XSS 风险
  > 只对**你可控的数据**开启 HTML 渲染；渲染不可信 HTML 会带来跨站脚本风险。
- **列头 Tooltip**：悬停列头显示数据集编辑器中的该列 `Description`，保持列描述更新可提升图表可发现性。
- **显示控制**：在仪表盘查看模式（非编辑）下，可从图表上下文菜单打开 `Display Controls`，调整 Time Grain、Time Column、图层可见性等，无需回到 Explore。
- **导出过滤后的数据**：表图显示 Search Box 时，`Download` **只导出筛选后当前可见的行**（有意为之）。要导出全量，请用图表三点菜单中的 `Download as CSV`（在应用搜索筛选之前）。

## 十一、AG Grid 交互表

适合**大数据量分页**场景，功能比标准 Table 更全。

- **服务端列过滤**：过滤在 SQL 中作为 `WHERE` 下推（不是只过滤当前页），因此百万行也正确；每列可用 AND/OR 组合多条件，不同列之间恒为 AND；分页始终基于已过滤结果集。
- **Time Shift（时间对比）**：`Advanced Analytics` → `Time Comparison`，输入偏移表达式（如 `1 year ago`、`minus 7 days`），生成对比列；仪表盘级时间范围会同时作用于基准期与对比期。

## 十二、动态货币格式

1. 数据集编辑器（`Datasets` → `Edit`）→ `Advanced` → `Currency Code Column`，填入存放 ISO 4217 货币码（`USD`/`EUR`/`GBP`…）的列名。
2. 在 Explore 中打开指标的 `Number format`，货币选择 `Auto-detect`。

开启后逐行按该列货币码格式化，单张图即可正确显示多币种。

## 十三、ECharts Option Editor

对 ECharts 系图表（line/bar/area/scatter/pie 等），`Customize` → `ECharts Options` 可输入原始 JSON 覆盖底层 ECharts 配置；JSON 会**深合并**到 Superset 生成的配置上，只需覆盖需要的部分。

```json
{
  "legend": { "orient": "vertical", "right": "5%", "top": "middle" },
  "title": { "text": "My Custom Title", "left": "center" }
}
```

> [!warning] 绕过校验
> 该覆盖会跳过 Superset 的校验层：无效键被 ECharts 静默忽略；与 Superset 生成项（如 `series`）冲突时可能产生意外结果。

## 相关

- [[60-数据与AI/10-数据分析/00-索引|10-数据分析]]
- [[60-数据与AI/00-索引|60-数据与AI]]
- Superset 官方文档：<https://superset.apache.org/>
