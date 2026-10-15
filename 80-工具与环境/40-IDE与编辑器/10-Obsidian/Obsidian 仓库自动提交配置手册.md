---
title: Obsidian 仓库自动提交配置手册
date: 2026-10-15
tags:
  - obsidian
  - 版本控制
  - git
  - 自动化
---

# Obsidian 仓库自动提交配置手册

> 让本仓库（Obsidian 技术文档库）定时自动 `commit` 并 `push` 到 GitHub，
> 同时在版本控制中剔除无需入库的文件。本文记录配置过程与剔除清单及原因。

## 一、方案总览

| 项目 | 配置 |
| --- | --- |
| 机制 | Obsidian Git 插件（当前 `2.41.1`）的「自动 commit-and-sync」 |
| 间隔 | 每 **10 分钟** |
| 动作 | 拉取合并（pull）→ 本地提交（commit）→ 推送到 `origin`（push） |
| 提交信息 | `vault backup: YYYY-MM-DD HH:mm:ss` |
| 触发条件 | 仅当 **Obsidian 处于运行状态** 时生效 |

流程：

```
修改笔记 ──▶ [每 10 分钟] ──▶ git pull（合并远端）
                              └▶ git commit
                                  └▶ git push origin main
```

## 二、前置条件

| 项 | 值 | 说明 |
| --- | --- | --- |
| 远程仓库 | `git@github.com:YangYunhe/obsidian.git` | `origin`，SSH 协议 |
| SSH 免密 | 已配置 | 无需输入口令；验证：`git ls-remote --heads origin` |
| git 身份 | `yangyunhe <yyh_5786@sina.com>` | 已设置于仓库本地配置 |
| 插件 | obsidian-git `2.41.1` | 已在 `.obsidian/community-plugins.json` 中启用 |

> 验证 SSH 是否可用：
> ```bash
> cd ~/文档/obsidian
> git ls-remote --heads origin      # 能列出 refs/heads/main 即为正常
> ```

## 三、配置过程

### 3.1 配置文件（推荐方式）

文件位置：`.obsidian/plugins/obsidian-git/data.json`

> 该文件已被 `.gitignore` 排除（见第四节），仅保存在本地，不会同步到远端。

```json
{
  "commitMessage": "vault backup: {{date}}",
  "autoCommitMessage": "vault backup: {{date}}",
  "commitDateFormat": "YYYY-MM-DD HH:mm:ss",
  "autoSaveInterval": 10,
  "autoPushInterval": 0,
  "autoPullInterval": 0,
  "autoPullOnBoot": true,
  "disablePush": false,
  "pullBeforePush": true,
  "differentIntervalCommitAndPush": false,
  "disablePopups": false,
  "showStatusBar": true
}
```

**字段含义：**

| 字段                                    | 取值                       | 含义                                     |
| ------------------------------------- | ------------------------ | -------------------------------------- |
| `autoSaveInterval`                    | `10`                     | 自动 commit-and-sync 间隔（**分钟**），`0` 表示关闭 |
| `disablePush`                         | `false`                  | `false`＝提交后推送；`true`＝只做本地提交            |
| `pullBeforePush`                      | `true`                   | 推送前先拉取并合并远端改动                          |
| `autoPullOnBoot`                      | `true`                   | Obsidian 启动时先拉取一次                      |
| `autoPushInterval`                    | `0`                      | 独立的「仅推送」定时器，已由 commit-and-sync 覆盖，故关闭  |
| `commitMessage` / `autoCommitMessage` | `vault backup: {{date}}` | 提交信息模板                                 |
| `commitDateFormat`                    | `YYYY-MM-DD HH:mm:ss`    | `{{date}}` 的渲染格式                       |
| `disablePopups`                       | `false`                  | `false` 时每次同步会弹出提示                     |
| `showStatusBar`                       | `true`                   | 在状态栏显示当前分支与同步状态                        |

### 3.2 图形界面（等价操作）

不想改文件时，可在 Obsidian 内操作：

**设置 → Git → Automatic**

- `Auto commit-and-sync interval` 填 **10**
- 打开 `Push on commit-and-sync`（对应 `disablePush=false`）
- 打开 `Pull on commit-and-sync`（对应 `pullBeforePush=true`）
- 打开 `Auto pull on boot`

> 界面改完会写回同一个 `data.json`，两种方式等价。

### 3.3 让配置生效

修改 `data.json` 后，需要让插件重新加载：

- 最简单：在 **设置 → Git** 里随便改动一项再改回来（触发保存）；或
- 禁用再启用 obsidian-git 插件；或
- 重启 Obsidian。

生效后状态栏会出现 Git 图标并显示分支名。

### 3.4 验证

1. 修改任意一篇笔记并等待 ≥10 分钟；
2. 终端执行：

   ```bash
   cd ~/文档/obsidian
   git log --oneline -3          # 应看到 "vault backup: ..." 提交
   git status -sb                # 应显示 "## main...origin/main"（无 ahead/behind）
   ```

## 四、剔除的文件（.gitignore）

`.gitignore` 已重写为**针对本仓库定制**的规则，只提交「笔记 / 附件 / Obsidian 配置」。
各条目与原因如下：

| 规则 | 类别 | 剔除原因 |
| --- | --- | --- |
| `.obsidian/workspace.json`<br>`.obsidian/workspace-mobile.json` | 本地状态 | 记录窗口布局、打开的文件，**每台设备各不相同**且频繁变动，入库只会产生噪声冲突 |
| `.obsidian/hotkeys.json` | 本地状态 | 个人快捷键，属本机偏好 |
| `.obsidian/cache`、`.cache`、`.temp_cache` | 缓存 | 可再生的缓存数据，无版本价值 |
| `.obsidian/plugins/*/data.json` | 密钥风险 | 插件本地配置，**可能含 Token / 密码 / 本机绝对路径**（如 obsidian-git 的凭据） |
| `.trash/` | 回收站 | Obsidian 删除的临时文件 |
| `raw` | 原料 | 未整理的原始素材，不入版本库 |
| `.claudian/` | 个人数据 | Claudian 的会话记录，含私人对话，且体积随时间增长 |
| `.claude/*`、`CLAUDE.local.md` | 个人配置 | AI 工具的本地会话与个人设置（保留 `.claude/skills/`） |
| `.DS_Store`、`Thumbs.db`、`desktop.ini`、`ehthumbs.db` | 系统垃圾 | 各操作系统生成的索引文件 |
| `*.tmp`、`*.temp`、`*.swp`、`*.swo`、`*.swn`、`*~`、`*.bak`、`*.orig`、`*.rej` | 编辑器临时文件 | 编辑过程产生的中间/备份文件 |
| `*.log`、`logs/`、`log/` | 日志 | 运行时日志，无需备份 |

### 附：由插件自行排除的文件

obsidian-git 插件会把凭据辅助脚本写入 **`.git/info/exclude`**（仅本地生效、不影响他人）：

```
.obsidian/plugins/obsidian-git/obsidian_askpass.sh
```

该文件是插件生成的临时凭据通道，不需要也不应入库。

### 会被提交的内容（确认清单）

```bash
git status --short        # 下列内容会被提交
```

- 所有笔记（`README.md`、`00-总览/` … `99-归档/`、`附件/`）
- `.gitignore`
- `.obsidian/` 的核心配置：`app.json`、`appearance.json`、`core-plugins.json`、`community-plugins.json`、`graph.json`、`templates.json`
- `.obsidian/plugins/*/` 的插件本体与 `manifest.json`（不含 `data.json`）

## 五、相对旧版 .gitignore 的改动

旧 `.gitignore` 是一份通用的 **Java / Python / ML / Superset** 工程模板（约 418 行），与本仓库（纯 Obsidian 文档库）无关，已整体替换。删除的规则举例及原因：

| 删除的规则 | 原因 |
| --- | --- |
| `*.class`、`*.jar`、`*.war`、`target/`、`dependency-reduced-pom.xml` | Java 构建产物，本仓库不含代码 |
| `.gradle`、`build/`、`.mvn/…` | 构建工具目录，本仓库无构建过程 |
| `node_modules`、`*.tsbuildinfo`、`*.map`、`npm-debug.log*` | 前端工程产物，本仓库不需要 |
| `__pycache__`、`.venv/`、`.pytest_cache`、`*.egg-info/` | Python 运行/打包产物，本仓库不需要 |
| `datasets/`、`models/`、`chroma/`、`*.db`、`dump.rdb` | 数据集/模型/数据库文件，本仓库不涉及 |
| `superset/*`、`docker-compose.override.yml` 等 | Superset 与编排工具专属规则 |
| `*.zip`、`*.tar.gz`、`*.7z` 等 | 归档文件；文档库里的附件应当被备份，故**不再忽略** |
| `.idea`、`*.iml`、`.vscode/` | IDE 工程配置；如日后需要可再加回 |

保留并可继续生效的：`raw`、`.DS_Store`、`Thumbs.db`、`.trash/`、`.claude/*`、临时/备份文件、日志。

> 若将来在本库内放置某个真正的小型工程，请**在该工程自己的子目录里**再加一份 `.gitignore`，不要污染仓库根规则。

## 六、常见问题

**Q1：等了 10 分钟没有自动提交？**
- 确认 Obsidian 正在运行（插件的定时器只在运行时工作）；
- 确认 obsidian-git 已启用（`设置 → 第三方插件`）；
- 确认 `data.json` 的 `autoSaveInterval` 为 `10` 且已重载插件（见 3.3）；
- 看状态栏是否有 Git 图标。

**Q2：提示「有冲突」怎么办？**
自动同步在 `pull` 阶段遇到冲突会停止并报错。手动处理：

```bash
cd ~/文档/obsidian
git status                 # 查看冲突文件
# 编辑冲突文件，保留需要的内容，删除 <<<<<<< ======= >>>>>>> 标记
git add <冲突文件>
git commit -m "fix: 解决同步冲突"
git push origin main
```

**Q3：换一台电脑怎么恢复？**
插件配置（`data.json`）不入库，需在新机器上重新：安装并启用 obsidian-git → 按 3.1/3.2 配置。笔记与附件会自动同步过来。

**Q4：想让插件配置也同步？**
删除 `.gitignore` 中 `.obsidian/plugins/*/data.json` 一行，并**先确认其中不含任何 Token / 密码**。

**Q5：想改提交间隔？**
把 `autoSaveInterval` 改成想要的分钟数（如 `5`、`30`），或直接在设置界面改。

**Q6：仓库里出现了大文件（如图片、附件）？**
附件会正常入库。若单个文件很大（>50MB），建议压缩或改用图床，避免仓库膨胀。

## 七、回滚

```bash
# 1. 关闭自动同步：把 data.json 的 autoSaveInterval 改为 0
#    或在 设置 → Git → Automatic 中把间隔清空

# 2. 恢复原 .gitignore
cd ~/文档/obsidian
git checkout HEAD -- .gitignore

# 3. （可选）删除插件配置
rm .obsidian/plugins/obsidian-git/data.json
```

## 相关

- [[80-工具与环境/40-IDE与编辑器/10-Obsidian/00-索引|10-Obsidian]]
- [[80-工具与环境/40-IDE与编辑器/00-索引|40-IDE与编辑器]]
- [[80-工具与环境/10-版本控制/00-索引|10-版本控制]]
- Obsidian Git 插件文档：<https://github.com/Vinzent03/obsidian-git>
