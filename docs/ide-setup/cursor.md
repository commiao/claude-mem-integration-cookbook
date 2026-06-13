# Cursor 接入

> **能力**：读 ✅ + 写 ❌（无自动 hook） + system prompt 注入 ❌。
> **难度**：低。merge 一段 JSON 到 `~/.cursor/mcp.json` 即可。

## 重要预期

Cursor **不提供** PostToolUse / SessionStart 这种 hook 机制。所以 claude-mem 在 Cursor 端**只能读**：

- ❌ Cursor 跑的工具调用不会被记录到 claude-mem
- ❌ Cursor 新会话不会自动注入历史
- ✅ Cursor 可以通过 MCP **查询**claude-mem 已经记录的观察（来自 Claude Code / Codex 写入的）

实际用法：**让 Claude Code 写记忆，Cursor 跨会话读取**。两个 IDE 协同。

## 前置

- claude-mem worker 在跑 ([INSTALL.md](../INSTALL.md))
- Cursor 已安装

## 安装

### 1. 找到 Cursor MCP 配置文件

| OS | 路径 |
|---|---|
| macOS / Linux | `~/.cursor/mcp.json` |
| Windows | `%USERPROFILE%\.cursor\mcp.json` |

如果不存在，新建它。如果存在，备份再改：

```bash
cp ~/.cursor/mcp.json ~/.cursor/mcp.json.bak.$(date +%s) 2>/dev/null || true
```

### 2. 添加 mcp-search server

用 Python（最稳，处理 JSON 合并）：

```bash
python3 <<'PY'
import json, os

target = os.path.expanduser('~/.cursor/mcp.json')

# claude-mem 自带的 MCP server 定义（路径解析逻辑跨设备稳定）
mcp_search = {
    "type": "stdio",
    "command": "sh",
    "args": [
        "-c",
        "_C=\"${CLAUDE_CONFIG_DIR:-$HOME/.claude}\"; "
        "_P=$({ ls -dt \"$_C/plugins/cache/thedotmack/claude-mem\"/[0-9]*/ 2>/dev/null; "
        "printf '%s\\n' \"$_C/plugins/marketplaces/thedotmack/plugin\"; } | "
        "while IFS= read -r _R; do _R=\"${_R%/}\"; "
        "[ -d \"$_R/plugin/scripts\" ] && _Q=\"$_R/plugin\" || _Q=\"$_R\"; "
        "[ -f \"$_Q/scripts/mcp-server.cjs\" ] && { printf '%s\\n' \"$_Q\"; break; }; done); "
        "[ -n \"$_P\" ] || { echo 'claude-mem: mcp server not found' >&2; exit 1; }; "
        "exec node \"$_P/scripts/mcp-server.cjs\""
    ]
}

cur = {}
if os.path.exists(target):
    cur = json.load(open(target))

cur.setdefault('mcpServers', {})['mcp-search'] = mcp_search

json.dump(cur, open(target, 'w'), indent=2)
print(f"已写入 {target}")
print(f"现有 servers: {list(cur['mcpServers'].keys())}")
PY
```

### 3. 验证 mcp-server 能跑起来

```bash
# 模拟 Cursor 启动 MCP server
cmd=$(python3 -c "import json;print(json.load(open(f'{__import__(\"os\").path.expanduser(\"~/.cursor/mcp.json\")}'))['mcpServers']['mcp-search']['args'][-1])")
echo '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2024-11-05","capabilities":{},"clientInfo":{"name":"test","version":"0"}}}' | sh -c "$cmd" 2>&1 | head -c 300
```

预期看到 `{"result":{"protocolVersion":...,"serverInfo":{"name":"claude-mem",...}}}`。

### 4. 重启 Cursor

完全退出 Cursor（macOS: Cmd+Q / Windows: 右键托盘 Quit），重新打开。

Cursor 会自动 hot-reload `mcp.json`，下次开会话时尝试 spawn `mcp-server.cjs`。

## 验收

在 Cursor 里发起一个新会话，问：

> "用 mcp-search 工具集查一下我最近的几条 claude-mem 记忆"

Cursor 应该自动调用 `mcp__mcp-search__search` 工具，返回真实观察。

后台验证：

```bash
ps -ef | grep mcp-server.cjs | grep -v grep
# 预期看到 Cursor 拉起的 node 进程
```

## 用例（最有价值的实战）

### 跨 IDE 引用记忆

- 你在 Claude Code 里**做了**：调试一个 bug，最后改了 `src/foo.ts` 加了某个 workaround
- 第二天你在 **Cursor** 打开同 project，问："那个 foo.ts 的 workaround 当时为什么这么改？"
- Cursor 调 `search` 工具拉历史 → 答出来

这是 Cursor + claude-mem 最大的价值——**Cursor 自己不写记忆，但能消费别的 IDE 写的记忆**。

### 在 Cursor Rules 里固化

`~/.cursor/rules/` 下加一条 rule，强制 Cursor 答任何工程问题前先查 claude-mem：

```markdown
# .cursor/rules/use-memory.md

Before answering any engineering question about this codebase, FIRST call
`mcp-search.search` with relevant keywords from the question. Reference the
returned observations in your answer with format: "Per past observation #<id>: ..."
```

## 能力矩阵

| 能力 | 状态 | 备注 |
|---|---|---|
| 工具调用自动捕获 | ❌ | Cursor 无 PostToolUse hook |
| `SessionStart` 注入 | ❌ | Cursor 无对应 hook |
| MCP 查询 `search` | ✅ | FTS5，<100ms |
| MCP 查询 `timeline` | ✅ | |
| MCP 查询 `get_observations` | ✅ | |
| MCP 查询 `list_corpora` | ✅ | 但 chroma 禁用时返 [] |
| MCP `smart_search` | ⚠️ | Cursor LLM 倾向选自家 codebase_search ([TS #11](../TROUBLESHOOTING.md#cursor-smart-search-misroute)) |
| MCP 写工具 (`memory_add` 等) | ❌ | 全局 server-beta 限制 |
| 跨设备同步 | ✅ | 同步 `~/.cursor/mcp.json` 到其它设备即可 |

## 已知坑

- [TS #11 Cursor smart_search 路由错误](../TROUBLESHOOTING.md#cursor-smart-search-misroute)
- [TS #12 server-beta runtime 限制](../TROUBLESHOOTING.md#server-beta-runtime)

## Windows 注意

`~/.cursor/mcp.json` 在 Windows 是 `%USERPROFILE%\.cursor\mcp.json`。

mcp-server 启动命令里的 `sh -c "..."` 在 Windows 原生 PowerShell 下**不工作**——`sh` 不存在。两个选择：

**A**：用 WSL2 跑 Cursor（最稳，但 Cursor 在 Windows 原生跑更常见）

**B**：把 mcp-server 命令改成 PowerShell 等价物。这个我没在 Windows 上实测，建议直接写一个绝对路径：

```json
{
  "mcpServers": {
    "mcp-search": {
      "type": "stdio",
      "command": "node.exe",
      "args": ["C:\\Users\\<you>\\.claude\\plugins\\marketplaces\\thedotmack\\plugin\\scripts\\mcp-server.cjs"]
    }
  }
}
```

替换 `<you>` 为你的 Windows 用户名。**未实测，欢迎 PR 修正**。
