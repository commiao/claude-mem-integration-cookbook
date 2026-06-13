# 跨平台从零安装

> 把 claude-mem 装到一台新机器，并选好开机自启策略。各 IDE 的接入步骤**不在本文档**——见 [ide-setup/](ide-setup/)。

## 前置依赖（所有平台共通）

| 依赖 | 用途 | 检查命令 |
|---|---|---|
| **Node ≥ 18** | npx 安装器 / MCP server | `node --version` |
| **Bun ≥ 1.1** | worker 运行时（claude-mem 强依赖） | `bun --version` |
| **Python 3.11+** | 部分 hook 脚本 + 后续 kg-hub 集成 | `python3 --version` |
| **uv / uvx** | ChromaDB 启动器（可选，建议禁用 chroma） | `uvx --version` |
| **sqlite3** | 数据库查询（一般系统自带） | `sqlite3 --version` |
| **curl + jq** | 健康检查 + JSON 处理 | `curl --version && jq --version` |
| **AI provider 凭证** | LLM 用来生成观察 | 见下方"AI provider 选择" |

---

## macOS（实战覆盖）

### 1. 装 Bun

```bash
curl -fsSL https://bun.sh/install | bash
# 跟随提示，把 ~/.bun/bin 加入 PATH
source ~/.zshrc  # 或 ~/.bashrc
```

### 2. 装 uv（如要语义搜索）

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

> 如果你**不**要语义搜索（FTS5 关键字搜索就够），可以跳过这步并设 `CLAUDE_MEM_CHROMA_ENABLED=false`（见 [TROUBLESHOOTING.md](TROUBLESHOOTING.md#search-timeout) 30s timeout 坑）。

### 3. 安装 claude-mem

```bash
npx -y claude-mem@latest install
# 安装器会问 AI provider 选项（见下）
```

预期输出关键字：
- `Plugin enabled OK`
- `Claude Code: plugin registered OK`
- `Worker autostart skipped` → 需要手动启动

### 4. 启动 worker

```bash
npx -y claude-mem@latest start
curl -s http://localhost:37701/api/health | python3 -m json.tool
# 预期: status=ok, mcpReady=true
```

> 端口号：`37700 + (uid % 100)`。比如 uid 501 → 37701。

### 5. 配置开机自启（LaunchAgent）

创建 `~/Library/LaunchAgents/com.claude-mem.worker.plist`：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTD/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.claude-mem.worker</string>

    <key>ProgramArguments</key>
    <array>
        <string>{{HOME}}/.bun/bin/bun</string>
        <string>{{HOME}}/.claude/plugins/marketplaces/thedotmack/plugin/scripts/worker-service.cjs</string>
        <string>--daemon</string>
    </array>

    <key>WorkingDirectory</key>
    <string>{{HOME}}/.claude/plugins/marketplaces/thedotmack</string>

    <key>EnvironmentVariables</key>
    <dict>
        <key>HOME</key>
        <string>{{HOME}}</string>
        <key>PATH</key>
        <string>{{HOME}}/.bun/bin:{{HOME}}/.npm-global/bin:{{HOME}}/.local/bin:/opt/homebrew/bin:/usr/local/bin:/usr/bin:/bin</string>
        <key>CLAUDE_MEM_CHROMA_ENABLED</key>
        <string>false</string>
    </dict>

    <key>RunAtLoad</key><true/>
    <key>KeepAlive</key>
    <dict>
        <key>SuccessfulExit</key><false/>
        <key>Crashed</key><true/>
    </dict>
    <key>ThrottleInterval</key><integer>10</integer>

    <key>StandardOutPath</key>
    <string>{{HOME}}/.claude-mem/logs/launchd.out.log</string>
    <key>StandardErrorPath</key>
    <string>{{HOME}}/.claude-mem/logs/launchd.err.log</string>

    <key>ProcessType</key><string>Background</string>
</dict>
</plist>
```

**重要**：把所有 `{{HOME}}` 替换成你的实际 home（`echo $HOME`，例如 `/Users/yourname`）。LaunchAgent **不支持** `~` 或 `$HOME` 变量。

加载：

```bash
plutil -lint ~/Library/LaunchAgents/com.claude-mem.worker.plist
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/com.claude-mem.worker.plist
launchctl print gui/$(id -u)/com.claude-mem.worker | grep "state"
# 预期: state = running
```

### 6. 验收

```bash
curl -s http://localhost:37701/api/health | python3 -m json.tool
ls ~/.claude-mem/  # 应该有 claude-mem.db, logs/, settings.json
```

---

## Linux（验证过 Ubuntu 22.04+, Debian 12+）

### 1-4 步骤同 macOS

（用 `~/.bashrc` 替代 `~/.zshrc`）

### 5. 配置开机自启（systemd user service）

创建 `~/.config/systemd/user/claude-mem-worker.service`：

```ini
[Unit]
Description=claude-mem worker (per-user)
After=network.target

[Service]
Type=simple
ExecStart=%h/.bun/bin/bun %h/.claude/plugins/marketplaces/thedotmack/plugin/scripts/worker-service.cjs --daemon
WorkingDirectory=%h/.claude/plugins/marketplaces/thedotmack
Environment="PATH=%h/.bun/bin:%h/.npm-global/bin:%h/.local/bin:/usr/local/bin:/usr/bin:/bin"
Environment="CLAUDE_MEM_CHROMA_ENABLED=false"
Restart=on-failure
RestartSec=10
StandardOutput=append:%h/.claude-mem/logs/systemd.out.log
StandardError=append:%h/.claude-mem/logs/systemd.err.log

[Install]
WantedBy=default.target
```

> `%h` 是 systemd 的 home 变量占位符，**不要**换成实际路径。

加载：

```bash
mkdir -p ~/.claude-mem/logs
systemctl --user daemon-reload
systemctl --user enable --now claude-mem-worker
systemctl --user status claude-mem-worker
# 预期: Active: active (running)

# 让 systemd 用户服务在用户登录前也跑（可选）
sudo loginctl enable-linger $USER
```

### 6. 验收

```bash
curl -s http://localhost:37701/api/health | python3 -m json.tool
journalctl --user -u claude-mem-worker -n 20
```

---

## Windows（**未在 Windows 实战测试** —— 基于上游官方文档 + 推断）

> ⚠️ **重要声明**：本节内容**未在真实 Windows 机器上验证**。所有路径、命令基于 Windows 通用约定与 [docs.claude-mem.ai](https://docs.claude-mem.ai/) 推断。如果你在 Windows 上真的跑通了或踩坑了，请 PR 修正本节。

### 推荐路径 A：WSL2（最稳妥）

WSL2 = 在 Windows 里跑一个 Linux 子系统。直接按本文档 **Linux 部分**操作。

```powershell
# PowerShell（管理员）
wsl --install -d Ubuntu-22.04
```

进入 Ubuntu，按 Linux 章节做。**所有 IDE 也都通过 WSL2 路径访问 worker**，配置不变。

**优势**：
- 路径分隔、launchd/systemd、shell 工具与 Linux 一致
- 不踩 Windows 路径转义、`bun.exe` vs `bun`、Defender 干扰等坑

**劣势**：
- 多一层虚拟化（性能略损）
- Windows 原生 GUI IDE 跨 WSL 访问 `127.0.0.1:37701` 需要正确配置 portforwarding（WSL2 通常自动，偶有失效）

### 推荐路径 B：原生 Windows

#### 1. 装 Bun

```powershell
# PowerShell
powershell -c "irm bun.sh/install.ps1 | iex"
# 重启 PowerShell
$env:PATH += ";$HOME\.bun\bin"
bun --version
```

#### 2. 装 Node + uv

- Node: https://nodejs.org/ —— 装 LTS 版（≥ 18）
- uv: `powershell -c "irm https://astral.sh/uv/install.ps1 | iex"`

#### 3. 安装 claude-mem

```powershell
npx -y claude-mem@latest install
```

数据目录：`%USERPROFILE%\.claude-mem\`（Linux/Mac 的 `~/.claude-mem/` 对应物）。

#### 4. 启动 worker

```powershell
npx -y claude-mem@latest start
curl http://localhost:37701/api/health
```

#### 5. 开机自启（Task Scheduler）

Windows 没有 LaunchAgent/systemd。两种方案：

**方案 5a（用户级，推荐）**：通过 Task Scheduler GUI

1. 打开 `Task Scheduler`
2. `Create Task`（不是 Create Basic Task）
3. General → Run only when user is logged on, Run with highest privileges 不勾
4. Triggers → New → At log on
5. Actions → New → Start a program:
   - Program: `C:\Users\<you>\.bun\bin\bun.exe`
   - Arguments: `C:\Users\<you>\.claude\plugins\marketplaces\thedotmack\plugin\scripts\worker-service.cjs --daemon`
   - Start in: `C:\Users\<you>\.claude\plugins\marketplaces\thedotmack`
6. Conditions → 取消 "Start only if on AC power"
7. Settings → "Allow task to be run on demand" + "If task fails, restart every 1 minute, attempt up to 3 times"

**方案 5b（命令行 schtasks）**：

```powershell
schtasks /Create /TN "claude-mem-worker" /SC ONLOGON `
  /TR "C:\Users\$env:USERNAME\.bun\bin\bun.exe C:\Users\$env:USERNAME\.claude\plugins\marketplaces\thedotmack\plugin\scripts\worker-service.cjs --daemon" `
  /RL LIMITED
```

设环境变量 `CLAUDE_MEM_CHROMA_ENABLED=false`：用户级环境变量 GUI（System Properties → Environment Variables）或 `setx CLAUDE_MEM_CHROMA_ENABLED false`。

#### 6. 验收

```powershell
curl http://localhost:37701/api/health
Get-Process | Where-Object {$_.ProcessName -like "*bun*"}
Get-Content $HOME\.claude-mem\logs\*.log -Tail 20
```

### Windows 已知不确定项

| 项 | Mac/Linux 行为 | Windows 推测 | 验证状态 |
|---|---|---|---|
| `bun-runner.js` stdin pipe | 工作 | 可能撞 Windows pipe buffer 大小限制 | 未实战 |
| 路径中的反斜杠 | N/A | claude-mem 内部如果有路径拼接 hardcoded `/` 可能 break | 未实战 |
| Defender 扫描 worker 频繁 fork | N/A | 性能可能下降 | 未实战 |
| Tailscale / VPN 跟 worker 监听 `127.0.0.1` 兼容性 | OK | 通常 OK | 未实战 |

如遇问题，先试 **WSL2 路径 A**。

---

## AI Provider 选择

claude-mem 调 LLM 来把工具调用变成结构化观察。可选 provider：

| Provider | 凭证来源 | 优势 | 劣势 |
|---|---|---|---|
| **Claude OAuth**（默认） | `claude auth login`（系统 Keychain） | 走 Claude Pro/Max 订阅，免费额度 | 受 Claude 配额限制；要先装 `@anthropic-ai/claude-code` CLI |
| **Anthropic API key** | `ANTHROPIC_API_KEY` env | 不限于订阅 | 按量付费 |
| **Gemini API key** | `GEMINI_API_KEY` env | 免费层大 | claude-mem 对 Gemini 优化稍弱 |
| **国内厂商兼容端点**（如阿里百炼 Coding Plan） | `ANTHROPIC_BASE_URL` + `ANTHROPIC_AUTH_TOKEN` | 国内速度快、配额大 | 模型名需配 `CLAUDE_MEM_MODEL`，详见下文 |

### 走国内厂商兼容端点（实战案例）

如果你用阿里百炼 Coding Plan（兼容 Anthropic Messages 协议）：

```bash
# ~/.claude-mem/.env  (权限 600)
ANTHROPIC_BASE_URL=https://coding.dashscope.aliyuncs.com/apps/anthropic
ANTHROPIC_AUTH_TOKEN=sk-<your-key>
ANTHROPIC_MODEL=qwen3.6-plus
ANTHROPIC_DEFAULT_HAIKU_MODEL=qwen3.6-plus
ANTHROPIC_DEFAULT_SONNET_MODEL=qwen3.6-plus
ANTHROPIC_DEFAULT_OPUS_MODEL=qwen3.6-plus
```

```json
// ~/.claude-mem/settings.json
{
  "CLAUDE_MEM_RUNTIME": "worker",
  "CLAUDE_MEM_PROVIDER": "claude",
  "CLAUDE_MEM_CLAUDE_AUTH_METHOD": "gateway",
  "CLAUDE_MEM_MODEL": "qwen3.6-plus"
}
```

**关键坑**：worker SDK 默认把请求里的 model 写死成 `claude-haiku-4-5-*`——百炼会 400 拒绝。**必须**在 `settings.json` 加 `CLAUDE_MEM_MODEL=qwen3.6-plus` 才能让 worker 用对的模型名调上游。详见 [TROUBLESHOOTING.md#bailian-400](TROUBLESHOOTING.md#bailian-400)。

> ⚠️ **runtime 环境变量必须放 plist/systemd 而不是 .env**：worker 加载 .env 时机晚于 chroma 构造，写在 .env 的 `CLAUDE_MEM_CHROMA_ENABLED=false` **不生效**。要写到 LaunchAgent 的 `EnvironmentVariables` 或 systemd 的 `Environment=` 里。

---

## 下一步

- 选好你常用的 IDE → 看 [ide-setup/](ide-setup/) 对应文档
- 查日常运维命令 → [CHEAT-SHEET.md](CHEAT-SHEET.md)
- 出问题了 → [TROUBLESHOOTING.md](TROUBLESHOOTING.md)
