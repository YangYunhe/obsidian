---
title: Obsidian 设置记录
date: 2026-10-16
tags:
  - obsidian
  - 环境配置
  - 记录
---

# Obsidian 设置记录

> 记录本仓库 Obsidian 的相关设置与「带附件笔记」的目录约定，用于在**另一台机器**上复现同一套环境。

## 一、随仓库同步的设置（已入库）

`.obsidian/` 下以下文件已纳入版本库，克隆后自动带入：

| 文件 | 作用 | 关键值 |
| --- | --- | --- |
| `app.json` | 编辑器 / 链接 / 附件 | 附件默认位置 `./assets`；新建链接格式 `relative`；`alwaysUpdateLinks: true` |
| `appearance.json` | 外观 | `nativeMenus: true`（未装主题，字体等均为默认） |
| `core-plugins.json` | 核心插件开关 | 见下方全文 |
| `community-plugins.json` | 社区插件清单 | `obsidian-excalidraw-plugin`、`realclaudian`、`obsidian-git` |
| `graph.json` | 关系图谱 | 默认视图 |
| `templates.json` | 模板文件夹 | `91-模板` |
| `plugins/*/data.json`、`plugins/*/manifest.json` | 插件设置 | 各插件自身设置（**不含插件本体**） |

**`app.json` 全文：**

```json
{
  "readableLineLength": false,
  "promptDelete": false,
  "attachmentFolderPath": "./assets",
  "alwaysUpdateLinks": true,
  "newLinkFormat": "relative"
}
```

**`core-plugins.json` 全文：**

```json
{
  "file-explorer": true,
  "global-search": true,
  "switcher": true,
  "graph": true,
  "backlink": true,
  "canvas": true,
  "outgoing-link": true,
  "tag-pane": true,
  "footnotes": false,
  "properties": true,
  "page-preview": true,
  "daily-notes": true,
  "templates": true,
  "note-composer": true,
  "command-palette": true,
  "slash-command": false,
  "editor-status": true,
  "bookmarks": true,
  "zk-prefixer": false,
  "random-note": false,
  "outline": true,
  "word-count": true,
  "slides": false,
  "audio-recorder": false,
  "workspaces": false,
  "file-recovery": true,
  "publish": false,
  "sync": true,
  "bases": true,
  "webviewer": false
}
```

**obsidian-git 设置**（`.obsidian/plugins/obsidian-git/data.json`，详见 [[80-工具与环境/40-IDE与编辑器/10-Obsidian/Obsidian 仓库自动提交配置手册|自动提交配置手册]]）：

- `autoSaveInterval: 10` —— 每 10 分钟 commit-and-sync
- `autoBackupAfterFileChange: false` —— **必须为 `false`**，否则定时器会变成「停止编辑后」触发
- `disablePush: false`、`pullBeforePush: true`、`autoPullOnBoot: true`

## 二、不同步的设置（每台机器各自维护）

已在 `.gitignore` 中排除，克隆后取各机默认值：

| 文件 | 原因 |
| --- | --- |
| `.obsidian/workspace.json`、`workspace-mobile.json` | 窗口布局与打开的文件，因机而异 |
| `.obsidian/hotkeys.json` | 个人快捷键 |
| `.obsidian/plugins/*/main.js`、`styles.css` | 插件打包本体（约 8.7 MB），从社区重装 |
| `.obsidian/cache`、`.trash/` | 缓存与回收站 |

## 三、核心设置要点

### 1. 附件与链接

- **附件默认位置**：当前笔记文件夹下的 `assets/`（`attachmentFolderPath: "./assets"`）。
- **新建链接格式**：相对路径（`newLinkFormat: "relative"`），写出的引用形如 `![[assets/xxx.png]]`。
- **自动更新内部链接**：开（移动 / 重命名时由 Obsidian 维护链接）。

### 2. 带附件笔记的目录约定

```
<笔记名>/
├── <笔记名>.md
└── assets/
    └── <附件>
```

- `assets/` 内文件不重名即可；但**全库范围内可能重名**（比如两个笔记各有一张 `封面.png`）。
- 因此引用**必须带路径前缀**（相对路径 `![[assets/xxx.png]]`），不要用裸文件名 `![[xxx.png]]`，否则全库同名时会产生歧义。
- 顶层 `附件/` 仅作跨笔记共用或历史遗留资源。

## 四、在新机器上复现

1. `git clone git@github.com:YangYunhe/obsidian.git`（或本机已有仓库执行 `git pull`）。
2. Obsidian →「打开文件夹作为库」→ 选择仓库根目录。
3. 首次打开时社区插件**不会**自动启用：到 设置 → 第三方插件 → 浏览，安装并启用 **Obsidian Git**、**Excalidraw**、**RealClaudian**（名称以社区市场为准；插件本体不入库，需各机自行安装）。
4. 安装后各插件配置（`data.json`）自动带入。
5. 核对 设置 → 文件与链接：
   - 默认附件位置：**当前文件夹下的 `assets`**
   - 新建链接格式：**相对路径**
   - 自动更新内部链接：**开**
6. 核对 设置 → 核心插件勾选，以及 模板文件夹 = `91-模板`。
7. 该机差异项（快捷键、窗口布局）按个人习惯另设；如需与主机完全一致，可单独拷贝 `hotkeys.json`。
8. 配置 Git 身份与 SSH 免密，见 [[80-工具与环境/40-IDE与编辑器/10-Obsidian/Obsidian 仓库自动提交配置手册|Obsidian 仓库自动提交配置手册]]。

## 五、核对清单

- [ ] 附件默认位置 = 当前文件夹下的 `assets`
- [ ] 新建链接格式 = 相对路径
- [ ] 自动更新内部链接 = 开
- [ ] 社区插件 = Obsidian Git / Excalidraw / RealClaudian
- [ ] 模板文件夹 = `91-模板`

## 相关

- [[80-工具与环境/40-IDE与编辑器/10-Obsidian/Obsidian 仓库自动提交配置手册|Obsidian 仓库自动提交配置手册]]
- [[80-工具与环境/40-IDE与编辑器/10-Obsidian/00-索引|10-Obsidian]]
- [[00-总览/04-写作与命名规范|写作与命名规范]]
