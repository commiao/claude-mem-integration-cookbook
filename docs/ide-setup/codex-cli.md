# Codex CLI 接入

> **能力**：读 ✅ + 写 ✅ + system prompt 注入 ✅。与桌面版同一套 hook 机制。
> **难度**：低。`npx claude-mem install --ide codex-cli` + `/plugins` slash command。

## 桌面版 vs CLI

| 项 | 桌面版 ([codex-desktop.md](codex-desktop.md)) | CLI（本文档） |
|---|---|---|
| 启动 | `/Applications/Codex.app/` 或图标 | 终端 `codex` 命令 |
| Plugin 启用入口 | GUI Settings → Plugins | **`/plugins` slash command** |
| Approve hooks 入口 | GUI 弹窗 | CLI prompt |
| Hook 机制 | 一致 | 一致 |

## 前置

- claude-mem worker 在跑 ([INSTALL.md](../INSTALL.md))
- Codex CLI 在 PATH (`which codex` 应该有路径)

## 安装

### Step 1: 跑 claude-mem 安装器

```bash
npx -y claude-mem@latest install --ide codex-cli
```

### Step 2: 在 Codex CLI 启用 plugin

打开 Codex CLI（在某个 project 目录）：

```bash
cd /path/to/your/project
codex
```

在 Codex 提示符下输入：

```
/plugins
```

Codex 显示一个 plugin 管理界面，找到 `claude-mem (local)` marketplace → `claude-mem` plugin → 选择 install / enable。

### Step 3: Approve 7 hooks

CLI 会列出 7 个 hook，逐个或一键 approve all。详见 [codex-desktop.md Step 3](codex-desktop.md#step-3-审查并批准-7-个-hooks)。

### Step 4: 退出并重新打开 Codex CLI

```bash
exit  # 退出当前 codex 会话
codex # 重新开
```

## 验收

同 [codex-desktop.md 验收](codex-desktop.md#验收)。检查：
- `sdk_sessions.platform_source='codex'`
- `observations.project='workspace_<cwd>'`
- worker 日志含 `platformSource=codex`

## CLI 特有：跨 project 切换

每次 `cd` 到不同目录再起 `codex`，**project 名按 cwd basename 推断**。

如果你想让多个目录共享同一份记忆（如同一 repo 的多个 worktree），看 [claude-code.md 高级](claude-code.md#高级跨-worktree-同-project-共享记忆)。

## 能力矩阵

同 [codex-desktop.md 能力矩阵](codex-desktop.md#能力矩阵)。

## 已知坑

同桌面版。`/plugins` 在 CLI 是真的 slash command（不像桌面版要走 GUI）。

## Windows

Codex CLI 在 Windows PowerShell / WSL 都能跑。

- **WSL2 路径**（推荐）：按 Linux 章节配 systemd worker，CLI 与 worker 都在 WSL 内
- **原生 PowerShell**：CLI 在 PowerShell 里跑，但 hook 脚本是 `sh -c "..."` shell 命令，**必须有 sh** → 装 Git Bash 或 MSYS2 提供 `sh.exe`

**未在 Windows PowerShell 实战 Codex CLI**，欢迎 PR。
