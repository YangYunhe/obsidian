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
| 动作 | 本地提交（commit）→ 拉取合并（pull）→ 推送到 `origin`（push） |
| 提交信息 | `vault backup: YYYY-MM-DD HH:mm:ss` |
| 触发条件 | 仅当 **Obsidian 处于运行状态** 时生效 |

流程：

```
修改笔记 ──▶ [每 10 分钟] ──▶ git commit
                              └▶ git pull（合并远端，pullBeforePush=true）
                                  └▶ git push origin main
```

> 顺序由 `syncMethod` 决定：本仓库为 `merge`，所以是 **commit → pull → push**；
> 只有 `syncMethod = reset` 时才是先 pull 再 commit。

## 二、前置条件

| 项 | 值 | 说明 |
| --- | --- | --- |
| 远程仓库 | `git@github.com:YangYunhe/obsidian.git` | `origin`，SSH 协议 |
| SSH 免密 | 已配置 | 无需输入口令；验证：`git ls-remote --heads origin` |
| git 身份 | `yangyunhe <yyh_5786@sina.com>` | 设置于**全局**配置（`git config --global user.*`），仓库本地未单独覆盖 |
| 插件 | obsidian-git `2.41.1` | 已在 `.obsidian/community-plugins.json` 中启用 |

> 验证 SSH 是否可用：
> ```bash
> cd ~/文档/obsidian
> git ls-remote --heads origin      # 能列出 refs/heads/main 即为正常
> ```

## 三、配置过程

### 3.1 配置文件（推荐方式）

文件位置：`.obsidian/plugins/obsidian-git/data.json`

> 该文件**会入库**（见第四节），所以换机器克隆后插件设置可直接复用。
> 代价是要自己保证其中不含 Token / 密码——入库前请按第四节的检查方法过一遍。

下面是**关键字段节选**。插件首次保存会把全部默认项一并写回，实际文件有 40 多个键，不必手工补齐：

```json
{
  "commitMessage": "vault backup: {{date}}",
  "autoCommitMessage": "vault backup: {{date}}",
  "commitDateFormat": "YYYY-MM-DD HH:mm:ss",
  "autoSaveInterval": 10,
  "autoBackupAfterFileChange": false,
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
| `autoBackupAfterFileChange`           | `false`                  | **必须为 `false`**；`true` 会改用「停止编辑 N 分钟后」的防抖触发，见下方提示 |
| `disablePush`                         | `false`                  | `false`＝提交后推送；`true`＝只做本地提交            |
| `pullBeforePush`                      | `true`                   | 推送前先拉取并合并远端改动                          |
| `autoPullOnBoot`                      | `true`                   | Obsidian 启动时先拉取一次                      |
| `autoPushInterval`                    | `0`                      | 独立的「仅推送」定时器，已由 commit-and-sync 覆盖，故关闭  |
| `commitMessage` / `autoCommitMessage` | `vault backup: {{date}}` | 提交信息模板                                 |
| `commitDateFormat`                    | `YYYY-MM-DD HH:mm:ss`    | `{{date}}` 的渲染格式                       |
| `disablePopups`                       | `false`                  | `false` 时每次同步会弹出提示                     |
| `showStatusBar`                       | `true`                   | 在状态栏显示当前分支与同步状态                        |

> [!warning] 两个开关只能留一个：`autoBackupAfterFileChange` 必须为 `false`
> obsidian-git 里有**两条互斥**的「自动提交」路径，由 `autoBackupAfterFileChange` 决定走哪条：
>
> | `autoBackupAfterFileChange` | 实际机制 | 触发时机 |
> | --- | --- | --- |
> | `false`（推荐） | `window.setTimeout` 定时器 | **每 N 分钟**一次，与有没有编辑无关；提交完成后自动重排下一次 |
> | `true` | `debounce` 防抖 | 只在**文件发生变动**时启动计时，**停止编辑 N 分钟后**才提交 |
>
> 踩坑点：`true` 时防抖函数只挂在 `vault` 的 `modify` / `create` / `delete` / `rename` 事件上。
> 如果这段时间**没有任何文件改动**，函数连被调用的机会都没有 —— 自然一分钟都不会提交。
> 本仓库要的是「每 10 分钟一次」的**定时**提交，所以必须是 `false`。

### 3.2 图形界面（等价操作）

不想改文件时，可在 Obsidian 内操作：

**设置 → Git → Automatic**

- `Auto commit-and-sync interval` 填 **10**
- **关闭** `Auto commit-and-sync after stopping file edits`（对应 `autoBackupAfterFileChange=false`）
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
| `.obsidian/plugins/*/main.js`<br>`.obsidian/plugins/*/styles.css` | 插件本体 | 插件打包后的代码与样式，**单文件最大 4.9 MB**（excalidraw 4.9 M + realclaudian 2.3 M + obsidian-git 0.76 M ≈ 8.7 MB），可随时从社区重装；入库只会让仓库与克隆变慢 |
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
- `.obsidian/plugins/*/` 的**配置**：`data.json`、`manifest.json`（不含 `main.js` / `styles.css`）

### 附：历史里的插件本体（已清理）

`.gitignore` 只约束**之后**的提交，`git rm --cached` 也只把它们从当前索引里移除。已经写进历史的 blob 会一直留在 `.git` 里——旧提交仍然引用它，`git gc` 回收不了。本项目已执行过一次清理：

| | 清理前 | 清理后（2026-10-15 实测） |
| --- | --- | --- |
| `.git` 体积 | 5.7 M | **264 K** |
| 历史中的插件本体 | 6 个对象 / 8.3 MB（解压） | 无 |
| 工作区的插件文件 | — | **一个没动**（6 个文件逐字节校验一致） |
| 全新克隆下载量 | 4.1 M | **188 K** |

> 上表是**清理当时**的实测值。`.git` 会随笔记与附件正常增长，与清理无关：2026-10-16 新增带截图的《使用阿里云服务器安装部署 Cloudera 6.3 服务》后，`.git` 已涨到约 13 M。
> 判断清理是否仍然有效，看下面这条（应无输出），而不是看 `.git` 的绝对值：
>
> ```bash
> git rev-list --objects --all | grep -E 'plugins/.*(main\.js|styles\.css)$'
> ```

本机没装 `git-filter-repo`，用 git 自带的 `filter-branch` 就够（提交少时完全没问题）：

```bash
cd ~/文档/obsidian

# 0) 改写前工作区必须干净（data.json 已入库，改过插件设置就会变脏）
git status --short

# 1) 改写：--index-filter 只操作索引与对象库，从不触碰工作区文件
FILTER_BRANCH_SQUELCH_WARNING=1 git filter-branch --force --index-filter \
  'git rm --cached --ignore-unmatch \
     .obsidian/plugins/obsidian-excalidraw-plugin/main.js \
     .obsidian/plugins/obsidian-excalidraw-plugin/styles.css \
     .obsidian/plugins/obsidian-git/main.js \
     .obsidian/plugins/obsidian-git/styles.css \
     .obsidian/plugins/realclaudian/main.js \
     .obsidian/plugins/realclaudian/styles.css' \
  --prune-empty -- --all

# 2) 校验插件文件没被动过（应无输出）
git status --short

# 3) 拆掉 filter-branch 留下的安全网 ref，否则第 5 步的回收白做
git update-ref -d refs/original/refs/heads/main
git reflog expire --expire=now --all
git gc --prune=now

# 4) 确认历史里已无插件本体（应无输出）
git rev-list --objects --all | grep -E 'plugins/.*(main\.js|styles\.css)$'

# 5) 强推。--force-with-lease 钉住推送前的旧 HEAD，避免覆盖别人的新提交
git push --force-with-lease=refs/heads/main:<推送前的HEAD全SHA> origin main
```

**坑点**

- 过滤命令里的路径要**逐个写全**。`--index-filter` 的字符串是交给 `sh` 执行的，写 `.obsidian/plugins/*/main.js` 会被 shell 当通配符展开，而不是 git 路径。
- 改写前工作区必须干净，否则直接报「无法重写分支：您有未暂存的变更」。
- `filter-branch` 会把旧历史存到 `refs/original/` 当安全网；不删它，第 5 步的回收就是白做。
- 改写后**所有提交的 SHA 都会变**（内容不变），任何按旧 SHA 记录的引用／书签都会失效。
- GitHub 端**不会立刻**释放空间：远端仓库里的旧对象要等 GitHub 自己 GC（想立刻清得联系 support）。本地 `.git` 是立即变小的。

**回滚**：改写前把整个仓库（含 `.git`）`cp -a` 备份一份，出问题覆盖回来即可。

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
- 确认 `autoBackupAfterFileChange` 为 **`false`**。若为 `true`，定时器会被换成「停止编辑 N 分钟后」的防抖触发，**期间不动任何文件就永远不会提交**（这是最容易踩的坑，见 3.1 提示）；
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
笔记、附件和插件配置（`data.json` / `manifest.json`）都在库里，克隆后即可拿到。
但**插件本体不入库**，需要在 `设置 → 第三方插件` 里从社区重新安装并启用：`obsidian-git`（必需）、`obsidian-excalidraw-plugin`、`realclaudian`。
启用后插件会读取库里的 `data.json`，自动提交配置随之生效，不必逐项重设。

**Q4：想让插件的代码本体也同步？**
删掉 `.gitignore` 里 `.obsidian/plugins/*/main.js` 与 `.obsidian/plugins/*/styles.css` 两行即可。
但这三份文件合计约 **8.7 MB**，会让仓库与克隆明显变慢，通常没必要——换机时从社区重装更省事。

**Q4b：入库 `data.json` 会不会泄露密钥？**
会，所以入库前必须检查。本仓库的检查方法：

```bash
cd ~/文档/obsidian
# 挑出可能存放密钥的字段名（只看字段，不看值）
grep -oiE '"[^"]*(token|secret|password|api_?key|credential)[^"]*"' .obsidian/plugins/*/data.json | sort -u
```

本仓库当前结果：`obsidian-git` 无此类字段；`excalidraw` 有 `taskboneAPIkey`，以及 AI 供应商配置里的 `apiKey` —— **均为空**，可以安全入库。
今后若给这些插件填过任何 Key，**填入前请先把它从版本库里拿掉**（或改用环境变量）。

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

# 3. （可选）删除插件配置。data.json 已被跟踪，用 git rm，否则删除动作会被自动提交
git rm .obsidian/plugins/obsidian-git/data.json
```

## 相关

- [[80-工具与环境/40-IDE与编辑器/10-Obsidian/00-索引|10-Obsidian]]
- [[80-工具与环境/40-IDE与编辑器/00-索引|40-IDE与编辑器]]
- [[80-工具与环境/10-版本控制/00-索引|10-版本控制]]
- Obsidian Git 插件文档：<https://github.com/Vinzent03/obsidian-git>
