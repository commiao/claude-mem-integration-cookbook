# Qoder 接入

> **能力**：读 ✅（通过 muxcp 中转） + 写 ⚠️（视配置而定） + system prompt 注入 ⚠️（需 transcript watcher）。
> **难度**：中-高。Qoder 自己没原生 hook，靠 muxcp + transcript watcher 桥接。

## 架构

Qoder 是 OpenAI 旗下的 IDE，对接 MCP 的方式是经 `muxcp`（MCP 聚合代理）：

```
Qoder IDE
   │ MCP stdio
   ▼
muxcp (聚合代理)
   │ 路由
   ▼
claude-mem-proxy.py  ←  这是适配层
   │
   ▼
mcp-server.cjs (claude-mem)
   │
   ▼
本地 worker (127.0.0.1:37701)
```

为什么要中间这层 proxy：Qoder 的 MCP 客户端实现跟 Claude Code / Cursor 的细节不同，直接连 `mcp-server.cjs` 可能不稳。proxy 帮忙做：
- 协议转换 / 标准化
- TranscriptWatcher 监听 Qoder 对话文件（用于补 SessionStart 注入）

## 前置

- claude-mem worker 在跑 ([INSTALL.md](../INSTALL.md))
- Qoder IDE 已装
- `~/.config/muxcp/` 已配置 — 见下方

## Step 1: 装 muxcp

muxcp 是个独立工具（不是 claude-mem 一部分）：

```bash
# 假设 muxcp 装到 ~/.local/bin/muxcp（具体看 muxcp 项目自己的安装说明）
which muxcp || echo "需要先装 muxcp"
```

## Step 2: 配置 muxcp 路由 claude-mem

`~/.config/muxcp/config.yaml`（或 `current.yaml` 视版本）：

```yaml
transport: stdio

servers:
  - name: mcp_search
    transport: stdio
    command: "${HOME}/.config/muxcp/bin/claude-mem-proxy.py"
```

`~/.config/muxcp/bin/claude-mem-proxy.py`（这个 proxy 脚本通常由 claude-mem 安装器或独立工具提供）：

```python
#!/usr/bin/env python3
"""
claude-mem MCP proxy for muxcp.
Locates the latest claude-mem mcp-server.cjs and execs it.
"""
import os, sys, subprocess
from pathlib import Path

claude_home = Path(os.environ.get('CLAUDE_CONFIG_DIR', f"{os.environ['HOME']}/.claude"))

# 优先用最新 cache，其次 marketplace
candidates = sorted(
    (claude_home / 'plugins' / 'cache' / 'thedotmack' / 'claude-mem').glob('[0-9]*/'),
    reverse=True
) + [claude_home / 'plugins' / 'marketplaces' / 'thedotmack' / 'plugin']

for path in candidates:
    server = path / 'scripts' / 'mcp-server.cjs'
    if not server.exists():
        server = path / 'plugin' / 'scripts' / 'mcp-server.cjs'
    if server.exists():
        os.execvp('node', ['node', str(server)])

print("claude-mem mcp-server.cjs not found", file=sys.stderr)
sys.exit(1)
```

```bash
chmod +x ~/.config/muxcp/bin/claude-mem-proxy.py
```

## Step 3: 配置 muxcp 启动器

`~/.config/muxcp/run-muxcp.sh`：

```bash
#!/usr/bin/env bash
set -euo pipefail

LOCAL_ENV="$HOME/.config/muxcp/local.env"
CONFIG_PATH="$HOME/.config/muxcp/config.yaml"
MUXCP_BIN="$HOME/.local/bin/muxcp"

if [ -f "$LOCAL_ENV" ]; then
  set -a
  . "$LOCAL_ENV"
  set +a
fi

exec "$MUXCP_BIN" -config "$CONFIG_PATH"
```

```bash
chmod +x ~/.config/muxcp/run-muxcp.sh
```

`~/.config/muxcp/local.env`（设备特定，不进同步）：

```bash
CLAUDE_MEM_RUNTIME=worker
CLAUDE_MEM_TRANSCRIPTS_ENABLED=true
CLAUDE_MEM_TRANSCRIPTS_CONFIG_PATH=$HOME/.config/muxcp/transcripts.json
```

```bash
chmod 600 ~/.config/muxcp/local.env  # 含敏感
```

## Step 4: 配置 Qoder MCP 客户端指向 muxcp

Qoder 的 MCP 配置位置：

| OS | 路径 |
|---|---|
| macOS / Linux | `~/.qoder/mcp.json` 或 `~/Library/Application Support/Qoder/mcp.json` |
| Windows | `%APPDATA%\Qoder\mcp.json` |

往里写：

```json
{
  "mcpServers": {
    "muxcp": {
      "type": "stdio",
      "command": "/Users/<you>/.config/muxcp/run-muxcp.sh"
    }
  }
}
```

替换 `<you>` 为你的用户名（Qoder 不一定支持 `~` 展开）。

## Step 5: 重启 Qoder

完全退出 Qoder + 重新打开。Qoder 会 spawn `run-muxcp.sh`，muxcp 进而启动 `claude-mem-proxy.py`，最终连到 worker。

## 验收

在 Qoder 对话框问：

> "用 mcp-search 工具集查一下我最近的几条 claude-mem 记忆"

Qoder 应该自动调用 `mcp__muxcp__mcp_search__search`，返回真实观察。

后台验证：

```bash
# muxcp 进程
ps -ef | grep -E "muxcp -config|claude-mem-proxy" | grep -v grep

# claude-mem-proxy spawn 的 mcp-server.cjs
ps -ef | grep "mcp-server.cjs" | grep -v grep
```

预期看到 muxcp daemon + claude-mem-proxy + node mcp-server.cjs 一条链。

## TranscriptWatcher（实验性 — 补写记忆能力）

Qoder 写对话到本地文件。如果你启用 TranscriptWatcher，claude-mem worker 会主动监听 Qoder 的对话文件，把每条用户/agent 交互**当作"观察"写入**——绕过 hook 缺失。

配置 `~/.config/muxcp/transcripts.json`：

```json
{
  "watchers": [
    {
      "name": "qoder",
      "path": "~/Library/Application Support/Qoder/transcripts/",
      "format": "qoder-jsonl"
    }
  ]
}
```

> ⚠️ 这个机制依赖 claude-mem v13.4+ 的实验特性。看上游 README 或 changelog 确认你的版本是否支持。

启用后 Qoder 会变成"半写记忆"：
- ✅ 用户消息、agent 回复**被记录**
- ❌ 但工具调用细节（哪些文件被读了、哪些命令跑了）**没法记**——因为 Qoder 不暴露这些到 transcript

## 能力矩阵

| 能力 | 状态 |
|---|---|
| MCP 查询（经 muxcp） | ✅ |
| 工具调用自动捕获 | ❌（Qoder 无 hook） |
| TranscriptWatcher（消息级捕获） | ⚠️ 实验性，看版本 |
| `SessionStart` 注入 | ❌（同 Cursor，Qoder 没对应 hook） |

## 已知坑

- 接入链路长（Qoder → muxcp → proxy → mcp-server → worker），任何一环断都不工作 → 用 §验收 的 ps 链定位
- muxcp 配置文件位置随版本变（`config.yaml` / `current.yaml`）—— 看你的 muxcp 文档
- Qoder agent 沙盒**严格限工作区内写文件**——管理本仓库以外的文档时常踩坑（[LESSONS.md L2](../LESSONS.md#l2-ide-agent-在沙盒里反复试错的反模式)）

## Windows

未在 Windows 实战。muxcp + Python proxy 在 Windows 理论可行，但 `sh -c` 子命令不工作，需要重写 muxcp 配置里的 stdio command 为 PowerShell / cmd 等价物。欢迎 PR。
