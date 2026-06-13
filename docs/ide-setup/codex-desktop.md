# Codex 桌面版接入

> **能力**：读 ✅ + 写 ✅ + system prompt 注入 ✅。能力等同于 Claude Code。
> **难度**：中。`npx claude-mem install` + GUI 启用 plugin + Approve 7 hooks，需要 3 个步骤。

## 重要：Codex 桌面版 ≠ Codex CLI

- **桌面版**（`/Applications/Codex.app/` 或 Windows 桌面应用）：本文档
- **CLI**（`codex` 命令）：见 [codex-cli.md](codex-cli.md)

二者**共享相同的插件机制**和 hook 注册，但 plugin 启用入口不同。

## 前置

- claude-mem worker 在跑 ([INSTALL.md](../INSTALL.md))
- Codex 桌面版已装

## 安装步骤

### Step 1: 跑 claude-mem 安装器

```bash
npx -y claude-mem@latest install --ide codex-cli
```

> 即使你用桌面版，参数是 `--ide codex-cli`——桌面版和 CLI 用同一套插件元数据。

输出会包含：

```
Enabled Codex plugin_hooks so claude-mem hooks can run.
Codex CLI: hooks marketplace registered OK
Installation Complete
Version:     13.x.x
```

也会包含：

```
Skipped (non-TTY)
Next steps:
  1. Open Codex CLI in your project
  2. Run /plugins
  3. Install claude-mem from the claude-mem (local) marketplace
  4. Restart Codex CLI after install so MCP tools and plugin hooks reload
```

**这条提示是给 CLI 的，桌面版没 `/plugins` slash command**。继续 Step 2。

### Step 2: 在 Codex 桌面版 GUI 启用 plugin

> ⚠️ Codex 桌面版**不响应** `/plugins` 作为 slash command（输到对话框会被发给 agent）。要走 GUI 入口。

1. 打开 Codex 桌面版
2. 顶部菜单栏 → **Codex** → **Settings...**（macOS）/ 设置（Windows）
3. 找到 **Plugins** / **Extensions** / **Marketplace** 标签
4. 在 Marketplace 列表下拉里选 **claude-mem (local)**
5. 在 Productivity 分类下找到 **claude-mem** plugin
6. 点右侧 **`+`** 按钮（或 "Install" / "启用"）

完成后 Codex 会显示提示：

```
7 hooks need review before they can run    [Review hooks]
```

### Step 3: 审查并批准 7 个 hooks

点 `[Review hooks]`，会列出 7 个 hook：

| 事件 | 数量 | 做什么 |
|---|---|---|
| SessionStart | 3 | (1) version-check (2) start worker if down (3) load context |
| UserPromptSubmit | 1 | 建立 session id |
| PreToolUse | 1 | file-context（Bash/Read 前提取文件上下文） |
| PostToolUse | 1 | 把工具调用转成观察 |
| Stop | 1 | session summary |

**安全评估**：每个 hook 都是 `node $PLUGIN_ROOT/scripts/bun-runner.js worker-service.cjs hook codex <subcommand>` 形式：
- 只调本地 node 脚本
- 不访问外网（worker 在 127.0.0.1）
- 不 sudo
- 不读 Keychain / .ssh / .aws
- 不删/改任何文件

**判定**：**批准全部 7 个 hooks**。

### Step 4: 完全退出 Codex 并重新打开

- macOS: `Cmd+Q`
- Windows: 关闭所有窗口 + 右键托盘 Quit

重新打开 Codex。**hook 只在新会话起 SessionStart 时生效**，不会自动重放到旧会话。

## 验收

### 决定性验证（绕过 Codex 自我报告偏差）

Codex 自己的 deep_thinking 经常误判 hook 是否触发（比如把 marker grep 包在它后续的 bash 命令文本里产生假命中）。**真正的 ground truth 是数据库字段**。

在 Codex 里新建对话，**复制粘贴下面这条命令**（marker 字面值不要改成变量）：

```bash
echo "CODEX_HOOK_VERIFICATION_LITERAL_$(uname -n)"
```

等 30 秒，让 worker LLM 处理。然后**在 Codex 之外**（终端 / Claude Code / Cursor 都行）跑：

```bash
# 1. sdk_sessions 应该有 platform_source='codex' 的 session
sqlite3 -header -column ~/.claude-mem/claude-mem.db \
  "SELECT id, platform_source, project,
          datetime(started_at_epoch/1000,'unixepoch','localtime') as t,
          substr(user_prompt,1,40) as prompt
   FROM sdk_sessions WHERE platform_source='codex'
   ORDER BY id DESC LIMIT 3;"
# 看到 platform_source='codex' = ✅

# 2. observations 应该有 project='workspace_<你 Codex cwd 的 basename>' 的新观察
sqlite3 -header -column ~/.claude-mem/claude-mem.db \
  "SELECT id, project, generated_by_model, substr(title,1,50) as title
   FROM observations
   WHERE created_at_epoch/1000 > strftime('%s','now')-300
     AND project LIKE 'workspace_%'
   ORDER BY id DESC LIMIT 5;"

# 3. worker 日志应该有 platformSource=codex 的 SessionRoutes
grep "platformSource=codex" ~/.claude-mem/logs/claude-mem-*.log | tail -3
```

### 判定标准（必须 3 个全 ✅）

| 指标 | ✅ 条件 | ❌ 含义 |
|---|---|---|
| `sdk_sessions.platform_source='codex'` 有最近条目 | 有 | hook 没触发 SessionStart |
| `observations.project='workspace_<cwd>'` 5 分钟内新增 ≥1 | 有 | hook 没触发 PostToolUse |
| `platformSource=codex` 日志行有今天时间戳 | 有 | hook 没向 worker 发请求 |

3 个全 ✅ = Codex 真正接入了。

## 工作机制

跟 Claude Code 一模一样。Codex 桌面版**内部**实现了 plugin_hooks，**不需要** spawn `codex` CLI 子进程（这点容易被诊断误导——pgrep 看不到 codex CLI 子进程不代表 hook 没工作）。

## 能力矩阵

| 能力 | 状态 |
|---|---|
| SessionStart 注入历史 | ✅ |
| PostToolUse 写记忆 | ✅ |
| MCP 查询 `search` 等 | ✅ |
| MCP 写工具 | ⚠️ 受 server-beta runtime 限制 |
| project 名 | `workspace_<cwd basename>` |
| 跨设备 | 配置自动 sync（plugin 元数据走 `~/.codex/config.toml`） |

## 已知坑

- [TS #8 Codex `/plugins` 不是 slash command](../TROUBLESHOOTING.md#codex-plugin-not-enabled)
- [TS #9 Codex hook 抢占 LaunchAgent](../TROUBLESHOOTING.md#codex-hook-race) ⚠️ **多 IDE 共存时最常踩**
- [TS #12 server-beta runtime 限制](../TROUBLESHOOTING.md#server-beta-runtime)

## 排错速查

| 症状 | 第一步检查 |
|---|---|
| 装完没 GUI 启用入口 | 重启 Codex 桌面版 |
| 启用了但 Approve hooks 后还是 0 obs | 完全 Quit + 重新打开，新建会话 |
| 数据库 0 增长 | 看 §验收 三个指标，定位是 hook / SDK / DB 哪层断 |
| 重启 Codex 后 LaunchAgent 显示 `not running` | [TS #9 孤儿 worker](../TROUBLESHOOTING.md#codex-hook-race) |

## Windows

Codex 桌面版在 Windows 也有原生应用。GUI 启用流程一致（菜单 → Settings → Plugins）。

hook 脚本会 spawn `bun` 和 `node` 子进程——确保 PATH 含这俩（Task Scheduler 的环境变量同 [INSTALL.md](../INSTALL.md) Windows 章节）。

**未在 Windows 实战验证 Codex 桌面版**，欢迎 PR 修正。
