# Claude Code 接入

> **能力**：读 ✅ + 写 ✅ + system prompt 注入 ✅。最完整的 IDE 集成路径。
> **难度**：低。`npx claude-mem install` 一条命令搞定。

## 前置

确认 [INSTALL.md](../INSTALL.md) 已完成（worker 在跑、Bun / Node / Claude OAuth 或其他 provider 就绪）。

## 安装

```bash
npx -y claude-mem@latest install
```

安装器自动做的事：
1. 把 plugin 拷到 `~/.claude/plugins/marketplaces/thedotmack/`
2. 注册 marketplace + 启用 plugin
3. 把 `auto-memory` 设为禁用（避免和 claude-mem hook 冲突）
4. 检查 Bun / uv 可用
5. 跑 `npm install` 装 plugin 依赖
6. 跳过 worker 自启（你手动启 + LaunchAgent/systemd）

确认安装成功：

```bash
cat ~/.claude/settings.json | python3 -c "
import sys,json
s=json.load(sys.stdin)
plugins = s.get('plugins', {})
mem = [k for k in plugins if 'claude-mem' in k or 'thedotmack' in k]
print('已启用:', mem if mem else '⚠️ 未找到 claude-mem plugin')"
```

预期输出含 `claude-mem@thedotmack` 之类。

## 工作机制

Claude Code 启动一个会话时：

```
SessionStart hook 触发
   ↓
claude-mem worker /api/context/inject?project=<basename(cwd)>
   ↓
拉回该 project 的相关历史观察
   ↓
注入到 Claude Code 的 system prompt
```

你跑工具调用（Read / Edit / Bash / MCP）时：

```
PostToolUse hook 触发
   ↓
worker 把工具调用喂给 LLM
   ↓
生成结构化观察（title / facts / narrative / concepts）
   ↓
写入 SQLite + 通过 SSE 流推给 Web UI
```

## 验收

### 1. SessionStart 注入

打开一个新的 Claude Code 会话（在你之前用过的 project 目录），不要主动调任何 MCP 工具，直接问：

> "总结一下这个项目我最近做了什么"

如果 Claude Code 能答出**具体观察**（含日期 / 文件名 / 操作类型），SessionStart 注入工作了。

如果它说"我不知道"，看 [TROUBLESHOOTING #7](../TROUBLESHOOTING.md#first-session-no-memory)。

### 2. PostToolUse 写入

随便让 Claude Code 跑一个 Bash：

```
跑一下 ls ~/.claude-mem
```

等 20-30 秒（LLM 处理观察需要时间），然后查数据库：

```bash
sqlite3 ~/.claude-mem/claude-mem.db \
  "SELECT id, type, generated_by_model,
          datetime(created_at_epoch/1000,'unixepoch','localtime') as t,
          substr(title,1,60) as title
   FROM observations
   WHERE created_at_epoch/1000 > strftime('%s','now')-120
   ORDER BY id DESC LIMIT 5;"
```

应该看到最近 2 分钟新增的观察，包含你跑的 ls 命令。

### 3. 看 sdk_sessions 表

```bash
sqlite3 ~/.claude-mem/claude-mem.db \
  "SELECT id, platform_source, project, substr(user_prompt, 1, 40)
   FROM sdk_sessions WHERE platform_source='claude'
   ORDER BY id DESC LIMIT 5;"
```

`platform_source='claude'` 那条就是 Claude Code 产生的 session。

## 高级：让某些工具调用不被记录

claude-mem 默认捕获**所有**工具调用。如果某些 tool（如频繁的 TodoWrite / Skill）你不想入库，可以在 `~/.claude-mem/settings.json` 加：

```json
{
  "CLAUDE_MEM_SKIP_TOOLS": "ListMcpResourcesTool,SlashCommand,Skill,TodoWrite,AskUserQuestion"
}
```

逗号分隔的 tool 名。重启 worker 生效。

## 高级：跨 worktree 同 project 共享记忆

如果你用 git worktree 在 `~/proj-A/` 和 `~/proj-A-experiment/` 同时干同一个 repo，默认 project 名按 basename 推断会**不一致**（`proj-A` vs `proj-A-experiment`）。

修复：在每个 worktree 根放一个 `.claude-mem-project`：

```bash
echo '{"project_name": "proj-A"}' > ~/proj-A/.claude-mem-project
echo '{"project_name": "proj-A"}' > ~/proj-A-experiment/.claude-mem-project
```

> 这个机制取决于 claude-mem 版本是否支持。v13.4+ 有，更早版本不一定。看上游 README。

## 能力矩阵

| 能力 | 状态 | 备注 |
|---|---|---|
| `SessionStart` hook（注入历史） | ✅ | 在第二次会话起生效 |
| `UserPromptSubmit` hook（生成 session） | ✅ | 每条用户消息触发 |
| `PreToolUse` hook（工具调用前） | ✅ | 罕用，主要给 file-context |
| `PostToolUse` hook（工具调用后） | ✅ | **核心写入路径** |
| `Stop` hook（会话结束） | ✅ | 触发 session summary |
| MCP 查询工具 (`search` 等) | ✅ | 也能用 |
| MCP 写工具 (`memory_add` 等) | ⚠️ | v13.2+ 要 server-beta runtime ([TS #12](../TROUBLESHOOTING.md#server-beta-runtime)) |
| 跨设备同步 | 取决于你 | 用 cc-switch / WebDAV 同步 settings.json 即可 |

## 已知坑

- [TS #1 Not logged in](../TROUBLESHOOTING.md#not-logged-in) — 没登录 Claude OAuth
- [TS #2 百炼 400](../TROUBLESHOOTING.md#bailian-400) — 切兼容端点要配 CLAUDE_MEM_MODEL
- [TS #3 search 30s timeout](../TROUBLESHOOTING.md#search-timeout) — 禁 chroma
- [TS #4 Claude not found](../TROUBLESHOOTING.md#claude-not-found-launchd) — PATH 没含 claude CLI
- [TS #7 第一次会话没注入](../TROUBLESHOOTING.md#first-session-no-memory) — 设计如此
