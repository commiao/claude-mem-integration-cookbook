# 日常使用

> 装完 claude-mem 之后，怎么用、怎么查、怎么知道它在工作。

## 它是怎么工作的（一图概览）

```
你在 IDE 里发问 / 让 agent 跑工具
            │
            ▼
   IDE hook 捕获工具调用（Read / Edit / Bash / MCP）
            │
            ▼ POST localhost:37701
   claude-mem worker（bun，常驻）
            │
            ▼ 调 LLM（你配的 provider）
   生成结构化观察：title + facts + narrative + concepts
            │
            ▼
   存 SQLite（~/.claude-mem/claude-mem.db）
            │
   ┌────────┴────────┐
   ▼                 ▼
 下次会话           MCP 查询工具
SessionStart        search / timeline /
注入相关历史         get_observations
```

## 查"它在工作吗"

```bash
# Worker 健康
curl -s http://localhost:37701/api/health | python3 -m json.tool

# 数据库有多少条观察
sqlite3 ~/.claude-mem/claude-mem.db "SELECT COUNT(*) FROM observations;"

# 最近 5 条观察（看 generated_by_model 字段确认你的 LLM provider 真的在用）
sqlite3 -header -column ~/.claude-mem/claude-mem.db \
  "SELECT id, type, generated_by_model,
          datetime(created_at_epoch/1000, 'unixepoch', 'localtime') as t,
          substr(title, 1, 60) as title
   FROM observations
   ORDER BY id DESC LIMIT 5;"

# 按 project 分布
sqlite3 -header -column ~/.claude-mem/claude-mem.db \
  "SELECT project, COUNT(*) FROM observations GROUP BY project ORDER BY 2 DESC;"

# 按来源 IDE 分布（platform_source）
sqlite3 -header -column ~/.claude-mem/claude-mem.db \
  "SELECT platform_source, COUNT(*) FROM sdk_sessions GROUP BY platform_source;"
```

## 看 Web UI

```
http://localhost:37701
```

claude-mem 自带一个 SSE 实时查看器。能看到刚产生的观察流。

## 查记忆（MCP 工具）

claude-mem 通过 MCP 暴露一组查询工具。不同 IDE 里的工具名前缀略有差异（看 IDE 是否经 muxcp 聚合）：

| 工具 | 用途 | 用法示例 |
|---|---|---|
| `search` | 关键字搜索 + 过滤（FTS5） | `{"query": "ChromaDB timeout", "limit": 5}` |
| `timeline` | 取某个观察前后的上下文 | `{"anchor": 42, "depth_before": 3, "depth_after": 3}` |
| `get_observations` | 按 ID 取详情 | `{"ids": [42, 110]}` |
| `list_corpora` | 列向量库（chroma 禁用时返回 `[]`） | `{}` |
| `smart_search` / `smart_outline` / `smart_unfold` | 语义/代码搜索（需 chroma 或 tree-sitter） | 视 IDE 是否有支持 |

> 工具名实际路径示例（Claude Code 插件直连）：`mcp__plugin_claude-mem_mcp-search__search`
>
> 经 muxcp 聚合后：`mcp__muxcp__mcp_search__search`

## 在 IDE 里自然语言用记忆

最简单的用法不需要你直接调工具——直接问 IDE 的 agent：

> "之前我们怎么修 ChromaDB 30 秒超时那个问题来着？"

agent 会自动调用 `search` 查记忆，返回的观察里就有答案。**这是 claude-mem 的核心价值**：跨会话失忆 → 跨会话有"记得"。

## 跨设备/跨 project 查询

不需要任何额外配置。默认所有 `search` 都是全局的——会同时返回所有 project 的观察。要按 project 过滤：

```
search 工具传 {"query": "...", "project": "workspace_myrepo"}
```

> 这一点与早期文档 §1 "不同 project → 互不可见" 的说法**冲突**——实测 project 是**默认全局可查 + 可选过滤**，不是隔离。

## 看 worker 日志（出问题第一步）

```bash
# 主应用日志
tail -50 ~/.claude-mem/logs/claude-mem-$(date +%Y-%m-%d).log

# hook 触发记录（grep 你最近跑的命令是否被记录）
grep "PostToolUse" ~/.claude-mem/logs/claude-mem-*.log | tail -10

# LaunchAgent / systemd 自己的输出
tail -20 ~/.claude-mem/logs/launchd.out.log   # macOS
journalctl --user -u claude-mem-worker -n 20  # Linux
```

## 性能预期

| 指标 | 健康值 |
|---|---|
| `/api/health` 响应 | < 50ms |
| `/api/search`（FTS5）| < 100ms |
| `/api/search`（chroma 启用 + 冷启动） | **15-30s ⚠️ 建议禁用 chroma** |
| Hook → 观察落库 | 5-30s（取决于 LLM 延迟 + 队列深度） |
| 数据库 100k 观察查询 | < 1s（FTS5 + SQLite 索引） |

## 备份

```bash
# 简单：把整个 ~/.claude-mem/ 同步到云盘 / WebDAV
rsync -av ~/.claude-mem/ /path/to/backup/.claude-mem/

# 只备份数据库
sqlite3 ~/.claude-mem/claude-mem.db ".backup '/path/to/backup/claude-mem-$(date +%Y%m%d).db'"
```

> ⚠️ `~/.claude-mem/.env` 含 API key —— 备份前确认目标位置加密 / 私有。

## 升级 claude-mem

```bash
# 上游发新版本时：
npx -y claude-mem@latest install --upgrade

# 重启 worker 让新插件代码生效
# macOS:
launchctl kickstart -k gui/$(id -u)/com.claude-mem.worker
# Linux:
systemctl --user restart claude-mem-worker
# Windows:
# Task Scheduler 重启任务，或直接 kill + 重启 npx claude-mem start
```

升级前建议先备份 `~/.claude-mem/`，特别是 `claude-mem.db` 和 `.env`。

## 下一步

- [CHEAT-SHEET.md](CHEAT-SHEET.md) - 命令速查
- [TROUBLESHOOTING.md](TROUBLESHOOTING.md) - 14 条已知坑
- 每个 IDE 自己的 setup 文档：[ide-setup/](ide-setup/)
