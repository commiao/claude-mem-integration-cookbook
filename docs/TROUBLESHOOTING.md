# 故障排查

> 14 条实测踩过的坑。按"症状"组织。每条含：触发条件 / 检测命令 / 修复 / 自然恢复条件。

---

## 索引

1. [`Not logged in` —— LLM 调用失败](#not-logged-in)
2. [`API Error: 400 model 'claude-haiku-4-5-*'`](#bailian-400) ← 走国内厂商端点的人都会遇到
3. [`search` 工具响应 30 秒超时](#search-timeout)
4. [`Claude executable not found in $PATH`（LaunchAgent 起的 worker）](#claude-not-found-launchd)
5. [`Executable not found in $PATH: "uvx"`（chroma sync 失败）](#uvx-not-found)
6. [`issue #2188` —— hook empty stdin payload](#issue-2188)
7. [第一次开会话没注入记忆](#first-session-no-memory)
8. [Codex 桌面版 install 完没自动启用 plugin](#codex-plugin-not-enabled)
9. [Codex hook 抢占 LaunchAgent，孤儿 worker](#codex-hook-race)
10. [`.env` 里的运行时开关不生效](#env-load-timing)
11. [Cursor `smart_search` 走了 Cursor 自家 codebase_search](#cursor-smart-search-misroute)
12. [v13.2 写工具 `requires CLAUDE_MEM_RUNTIME=server-beta`](#server-beta-runtime)
13. [`worker PID file points to a live process, skipping duplicate spawn`](#duplicate-spawn)
14. [新会话产生的观察看不到 / 数据库没增长](#no-new-obs)
15. [**某个 IDE 的采集静默停摆数周无人察觉**（claude-mem 升级后 hook 失信）](#silent-capture-halt) ← 最隐蔽

---

<a id="not-logged-in"></a>
## 1. `Not logged in · Please run /login`

**症状**：worker 日志反复出 `Not logged in`，observation 表不增长。

**触发**：claude-mem 默认用 Claude OAuth，但本机 `claude` CLI 没登录过。worker spawn 时读不到 Keychain 里的 OAuth token。

**检测**：
```bash
claude auth status
# 预期 loggedIn: true
```

**修复**：
```bash
claude auth login   # 浏览器走 OAuth
# 重启 worker 让它重新读 Keychain
launchctl kickstart -k gui/$(id -u)/com.claude-mem.worker
```

**自然恢复**：登录后 worker 重启即恢复，**不是** Keychain 自动注入。

---

<a id="bailian-400"></a>
## 2. `API Error: 400 invalid_parameter_error: model 'claude-haiku-4-5-*'`

**症状**：切到阿里百炼 / 其他兼容端点后，worker 调 LLM 报 400，所有观察生成失败。

**触发**：claude-mem worker 的 Claude Agent SDK **写死** model 名为 `claude-haiku-4-5-20251001`。百炼等端点不认这个 model 名，必须用 `qwen3.6-plus` 等真实模型名。

**检测**：
```bash
grep "API Error: 400" ~/.claude-mem/logs/claude-mem-*.log | tail -3
# 看到 "invalid_parameter_error: model 'claude-haiku-*'" 就是这个坑
```

**修复**：在 `~/.claude-mem/settings.json` 加 `CLAUDE_MEM_MODEL`：

```json
{
  "CLAUDE_MEM_RUNTIME": "worker",
  "CLAUDE_MEM_PROVIDER": "claude",
  "CLAUDE_MEM_CLAUDE_AUTH_METHOD": "gateway",
  "CLAUDE_MEM_MODEL": "qwen3.6-plus"
}
```

并在 `~/.claude-mem/.env` 配 `ANTHROPIC_DEFAULT_HAIKU_MODEL=qwen3.6-plus`（worker 还会回退查这个）。然后重启 worker。

**关键**：**单设 `.env` 里的 `ANTHROPIC_MODEL` 不够**。SDK 内部会用自己写死的 model 名 override `.env`。必须 `CLAUDE_MEM_MODEL` 落到 `settings.json`。

---

<a id="search-timeout"></a>
## 3. `search` 工具响应 30 秒超时

**症状**：IDE 调 `search` 工具，等 30 秒后报 timeout；但 `timeline` / `get_observations` 都很快。

**触发**：worker 的 `/api/search` 端点**先**试 ChromaDB 语义搜索，chroma-mcp 冷启动需 15-30s，超过 MCP 客户端 30s 默认 timeout。其他工具不走 chroma 所以不受影响。

**检测**：
```bash
time curl -s "http://localhost:37701/api/search?query=test&limit=3" -o /dev/null
# 慢于 5 秒就是这个问题
grep "CHROMA_MCP" ~/.claude-mem/logs/claude-mem-*.log | tail -5
```

**修复**（彻底）：在 plist / systemd / Task Scheduler 的环境变量里加：

```
CLAUDE_MEM_CHROMA_ENABLED=false
```

worker 启动时直接不构造 chroma path，search 走 SQLite FTS5。响应时间 26s → 0.05s（500x 提速）。

**代价**：失去 `smart_search` / `smart_outline` / `smart_unfold` 等语义搜索工具。普通 `search` (FTS5) 不受影响。

**关键**：**写在 `~/.claude-mem/.env` 不生效**，必须写到 plist `EnvironmentVariables` / systemd `Environment=` 里。详见 §10 .env 加载时机。

---

<a id="claude-not-found-launchd"></a>
## 4. `Claude executable not found in $PATH`（LaunchAgent 启动的 worker）

**症状**：worker 跑着，但 SDK 调用 LLM 时报 `Claude executable not found`，观察一直 0 增。

**触发**：LaunchAgent / systemd **不继承登录 shell 的 PATH**。worker 内部 spawn `claude` 子进程时找不到它（通常在 `~/.npm-global/bin/claude`）。

**检测**：
```bash
ps eww -p $(curl -s http://localhost:37701/api/health | python3 -c "import sys,json;print(json.load(sys.stdin)['pid'])") | tr ' ' '\n' | grep '^PATH='
# 看 PATH 是否包含 ~/.npm-global/bin
which claude  # 确认实际位置
```

**修复**：plist `EnvironmentVariables.PATH` 至少要包含 4 类目录：

```xml
<key>PATH</key>
<string>{{HOME}}/.bun/bin:{{HOME}}/.npm-global/bin:{{HOME}}/.local/bin:/opt/homebrew/bin:/usr/local/bin:/usr/bin:/bin</string>
```

四个关键路径：
- `~/.bun/bin` — bun
- `~/.npm-global/bin` — claude CLI（SDK 要 spawn）
- `~/.local/bin` — uvx（chroma 启动器）
- 系统路径

---

<a id="uvx-not-found"></a>
## 5. `Executable not found in $PATH: "uvx"`

**症状**：worker 日志反复刷 `[CHROMA_MCP] Connection failed ... uvx`，chroma 同步失败。

**触发**：同 §4，PATH 没含 `~/.local/bin`（uvx 默认装这）。

**修复**：要么加 `~/.local/bin` 到 PATH（同 §4），要么直接 `CLAUDE_MEM_CHROMA_ENABLED=false` 干脆禁 chroma（§3）。**禁用 chroma 是更省事的方案**，因为冷启动 30s 本身就是个问题。

---

<a id="issue-2188"></a>
## 6. issue #2188 — hook empty stdin payload

**症状**：`~/.claude-mem/CAPTURE_BROKEN` 文件存在，`logs/runner-errors.log` 反复刷：

```
[bun-runner] empty stdin payload received — issue #2188
```

**触发**：升级 / 重装 claude-mem 后，**旧的 IDE 会话**还在用过期的 hook 注册。表现为 stdin pipe 收到 0 字节。

**检测**：
```bash
ls -la ~/.claude-mem/CAPTURE_BROKEN
tail -5 ~/.claude-mem/logs/runner-errors.log
# 看时间戳是否在最近 30 分钟内
```

**修复**：**重启 IDE**（不是重启 worker）。新 IDE 会话用新 hook 注册即可。

**自然恢复**：升级后开新 IDE 会话即恢复。**老会话不会自愈**。

---

<a id="first-session-no-memory"></a>
## 7. 第一次在新 project 里开会话，没历史记忆注入

**症状**：在新目录开 IDE，问"之前这里发生过什么"，agent 说没有上下文。

**触发**：上游设计——**第一次会话只负责播种记忆，第二次起才注入历史**。属于 feature，不是 bug。

**检测**：看 `/api/context/inject?project=<your-project>`：

```bash
curl "http://localhost:37701/api/context/inject?project=$(basename $PWD)"
# 第一次返回 "This project has no memory yet"
```

**修复 / 加速**：
- 自然方式：用就是了，第二次会话就有
- 主动方式：在 IDE 里跑 `/learn-codebase`（如果该 IDE 支持），让 claude-mem 一次性扫描整个 repo

---

<a id="codex-plugin-not-enabled"></a>
## 8. Codex 桌面版 `npx claude-mem install --ide codex-cli` 完没自动启用 plugin

**症状**：install 输出说"please run `/plugins` to enable"，但 Codex 桌面版**没有 `/plugins` slash command**——输到对话框被发给 agent。

**触发**：Codex 桌面版的 plugin 管理走 GUI 而不是 slash command。

**修复**：
- 打开 Codex 桌面版
- 顶栏菜单 → Settings / Preferences → Plugins / Extensions / Marketplace
- 找到 `claude-mem (local)` marketplace 下的 `claude-mem` plugin
- 点 `+` 或 Install
- 会弹"7 hooks need review" → 点 Review → Approve all
- 完全退出 Codex（Cmd+Q）→ 重新打开

详见 [ide-setup/codex-desktop.md](ide-setup/codex-desktop.md)。

---

<a id="codex-hook-race"></a>
## 9. Codex hook 抢占 LaunchAgent，产生孤儿 worker

**症状**：`launchctl print` 显示 `state = not running`，但 `lsof :37701` 看到 worker 在跑。worker 用了旧版本（cache/13.0.1 等），不在 LaunchAgent 管理下。

**触发**：Codex 的 SessionStart hook 会跑 `worker-service.cjs start`——如果发现 worker 没在 37701 监听，**自己 spawn 一个**。和 LaunchAgent 的 KeepAlive 拉起逻辑产生竞争。10 秒 ThrottleInterval 内 Codex 抢先就出这事。

**检测**：
```bash
launchctl print gui/$(id -u)/com.claude-mem.worker | grep state
# state = not running
lsof -nP -iTCP:37701 -sTCP:LISTEN
# 但端口在被某个 bun 进程占着
```

**后果**：
- 失去 LaunchAgent 的崩溃自愈
- plist 里的 `EnvironmentVariables` 不生效（CLAUDE_MEM_CHROMA_ENABLED 等不应用到孤儿 worker）

**修复**：杀孤儿 worker + 重新 bootstrap LaunchAgent：
```bash
kill $(lsof -nP -iTCP:37701 -sTCP:LISTEN -t)
launchctl bootout gui/$(id -u)/com.claude-mem.worker 2>/dev/null
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/com.claude-mem.worker.plist
```

**自然恢复**：Mac 重启时 LaunchAgent 启动顺序在 Codex 之前，自动恢复。会话中不会自愈。

---

<a id="env-load-timing"></a>
## 10. `~/.claude-mem/.env` 里的运行时开关不生效

**症状**：把 `CLAUDE_MEM_CHROMA_ENABLED=false` 写到 `.env` 里，worker 重启后 chroma 还在 try connect。

**触发**：worker 自己解析 `.env` 注入 `process.env`，但**加载时机晚于早期初始化**（如 chroma module 构造）。

**修复规则**：
- **凭证类**（`ANTHROPIC_AUTH_TOKEN` 等）放 `.env`：用在请求时，那会 `.env` 已加载
- **运行时开关**（`CLAUDE_MEM_CHROMA_ENABLED`、`CLAUDE_MEM_MODEL` 等启动期决策）放 plist / systemd / 系统环境变量

---

<a id="cursor-smart-search-misroute"></a>
## 11. Cursor 调 `smart_search` 返回代码搜索结果而不是记忆

**症状**：在 Cursor 里说"语义搜索一下 X"，返回的是代码片段，不是记忆库观察。

**触发**：**Cursor 内置 `codebase_search` 工具**——Cursor LLM 看到"smart search" / "semantic search" 字眼时优先选自家工具，而不是 `mcp__mcp-search__smart_search`。**这不是 bug，是 Cursor 的工具路由偏向**。

**修复**：显式指定 server 名：
- ❌ "用 smart_search 搜一下 ..."
- ✅ "用 mcp-search 工具集里的 search 工具搜一下 ..."
- ✅ "查 claude-mem 本地记忆 ..."（强调"本地记忆"四字会让 Cursor 选 MCP）

普通 `search` 不受影响——Cursor 端没有同名工具。

---

<a id="server-beta-runtime"></a>
## 12. v13.2+ 写工具 `requires CLAUDE_MEM_RUNTIME=server-beta`

**症状**：调 `memory_add` / `observation_add` / `memory_search` / `observation_search` 等 v13.2 新工具报：

```
Server beta error: requires CLAUDE_MEM_RUNTIME=server-beta.
Current runtime is "worker"; use the existing search/timeline/get_observations
tools for worker-mode memory access.
```

**触发**：claude-mem v13.2+ 新增的 8 个 MCP 工具（`memory_*` / `observation_*`）需要 `CLAUDE_MEM_RUNTIME=server-beta`，但 worker 模式默认拒绝。**这是上游全局限制，不是某个 IDE 的问题**。

**当前对策**：
- 用 `search` / `timeline` / `get_observations` 替代——它们覆盖 90% 场景
- 想用新工具：切到 server-beta 模式（涉及 `npx claude-mem server start` 子命令族，上游文档有，但与现有 worker 部署不兼容，迁移成本高）

---

<a id="duplicate-spawn"></a>
## 13. `Worker PID file points to a live process, skipping duplicate spawn`

**症状**：worker 日志反复刷这一行，但功能正常。

**触发**：claude-mem worker 用 `~/.claude-mem/worker.pid` 防止重复启动。多个 hook / launchctl / 手动 `claude-mem start` 同时来时，会触发这个保护。

**修复**：通常不用管，这是**预期保护行为**。如果你确实想重启 worker，用 `kickstart -k`（macOS）或 `systemctl restart`，而不是直接跑 `claude-mem start`。

---

<a id="no-new-obs"></a>
## 14. 新会话产生的观察看不到 / 数据库没增长

**症状**：在 IDE 里跑了一堆工具调用，但 `SELECT COUNT(*)` 没增长。

**诊断流程（按顺序）**：

```bash
# 14.1 worker 在跑吗
curl http://localhost:37701/api/health || echo "worker 不在"

# 14.2 hook 触发了吗
grep "PostToolUse" ~/.claude-mem/logs/claude-mem-*.log | tail -5
# 如果空 → 你的 IDE 没装 hook（看对应 ide-setup/*.md）

# 14.3 SDK 调 LLM 失败？
grep "API Error\|Not logged in\|invalid_parameter" ~/.claude-mem/logs/claude-mem-*.log | tail -5
# 命中某条 → 走对应章节

# 14.4 hook 触发了但被 ingest filter 过滤？（v13.4+）
grep "filter_decision" ~/.claude-mem/logs/claude-mem-*.log | tail -5

# 14.5 数据库写入正常但查的不对？
sqlite3 ~/.claude-mem/claude-mem.db \
  "SELECT id, project FROM observations ORDER BY id DESC LIMIT 3;"
# 看 project 字段是否符合预期（cwd basename）
```

按返回结果走对应章节。

---

<a id="silent-capture-halt"></a>
## 15. 某个 IDE 的采集静默停摆，数周无人察觉

**症状**：一切"看起来正常"——worker 健康、config 里 hook 配置都在、其他 IDE 每天都在写。但**某一个 IDE 的数据停在几周前的某一天**，之后一条没有。没有任何错误日志。

**实测案例**：Codex 采集从 2026-07-13 07:22 停摆，到 08-20 才被发现，**空窗 5 周**。

### 为什么这么难发现

| 干扰因素 | 具体表现 |
|---|---|
| 没有错误信号 | Codex 的 hook 审批是安全设计，hash 不匹配就**静默不执行**，不报错 |
| 配置完好 | `~/.codex/config.toml` 里 plugin `enabled = true`、5-7 条 `trusted_hash` 全在 |
| 其他 IDE 正常 | 同一个 worker，claude 每天 100-400 条 → 一眼看去"系统在工作" |
| **COUNT 看不出来** | `codex: 37 sessions` 这个数字本身毫无异常感 |

### 🔑 检测方法：看 MAX，不要看 COUNT

这是本条最重要的一句话。

```bash
sqlite3 -header -column ~/.claude-mem/claude-mem.db "
SELECT platform_source,
       COUNT(*) AS sessions,
       datetime(MAX(started_at_epoch)/1000,'unixepoch','localtime') AS last_seen
FROM sdk_sessions GROUP BY platform_source ORDER BY MAX(started_at_epoch) DESC;"
```

停摆时的输出长这样：

```
platform_source  sessions  last_seen
---------------  --------  -------------------
claude           390       2026-08-20 15:37:31   ← 今天
cursor           47        2026-08-17 17:01:31   ← 3 天前，正常（偶发使用）
codex            37        2026-07-13 07:22:39   ← 🚨 5 周前！
```

`COUNT(*)=37` 完全正常；**`MAX(started_at)` 一眼暴露问题**。

**同时验 observations 层**（session 建了 ≠ 观察生成成功，两层可能只坏一层）：

```bash
sqlite3 -header -column ~/.claude-mem/claude-mem.db "
SELECT s.platform_source, COUNT(o.id) AS obs,
       datetime(MAX(o.created_at_epoch)/1000,'unixepoch','localtime') AS last_obs
FROM observations o JOIN sdk_sessions s ON o.memory_session_id = s.memory_session_id
GROUP BY s.platform_source ORDER BY MAX(o.created_at_epoch) DESC;"
```

### 根因排查（以 Codex 为例）

```bash
# 1. approve 了几个 hook
grep -o 'codex-hooks.json:[a-z_]*:[0-9]*:[0-9]*' ~/.codex/config.toml | sed 's/.*json://' | sort

# 2. 磁盘上实际有几个 hook
python3 -c "
import json,glob
for p in glob.glob('$HOME/.codex/plugins/cache/*/claude-mem/*/hooks/codex-hooks.json'):
    d=json.load(open(p))['hooks']
    n=sum(len(g.get('hooks',[])) for arr in d.values() for g in arr)
    print(f'{n} hooks in {p}')"

# 3. worker 有没有收到过该来源的请求（换成你的 IDE 名）
grep 'platformSource=codex' ~/.claude-mem/logs/claude-mem-*.log | tail -3
```

**判据**：approve 的 hook key 集合 ≠ 磁盘上实际的 hook 集合 → 就是这个问题。

典型差异（claude-mem v13.0 → v13.15）：

```
approve 了 7 个              磁盘上只剩 5 个
session_start:0:0  ✓         session_start:0:0  ✓
session_start:0:1  ✗ 已删     （SessionStart 3 个合并成 1 个）
session_start:0:2  ✗ 已删
pre/post_tool_use, user_prompt_submit, stop  ✓
```

### 修复

见 [ide-setup/codex-desktop.md → 每次升级后必须重新 Approve](ide-setup/codex-desktop.md#️-每次-claude-mem-升级后必须重新-approve-hooks)。一句话：GUI → Settings → Plugins → Review hooks → Approve all → Cmd+Q 重启。**不改任何代码，不手改 config.toml**（手改 trusted_hash = 绕过安全审批）。

### 预防

1. **每次 claude-mem 升级后主动跑一次上面的 MAX 查询**
2. 把这条查询做成 cron / 探针，任一来源 `last_seen` 超过 N 天就告警
3. 别用 `platform_source` 的**分布**推断"谁在干活"——采集通道坏掉的那个来源会直接从统计里消失，造成"只有 X 在工作"的假象

> **元教训**：用任何分布数据下结论前，先验证**该数据的采集通道当前是否健康**。这个坑在本次事故里被踩过一次：看到"最近 400 条观察全是 claude"就推断"只有 claude 在干活"，实际是 codex 的活根本没进采集。

---

## 通用诊断脚本

如果你在新机器上排查，把下面这段存为 `check-claude-mem.sh`：

```bash
#!/usr/bin/env bash
echo "=== 1. worker 健康 ==="
curl -s --max-time 3 http://localhost:37701/api/health | python3 -m json.tool 2>&1 | head -15

echo
echo "=== 2. 端口绑定 ==="
lsof -nP -iTCP:37701 -sTCP:LISTEN 2>/dev/null | head -3 || echo "[37701 无人监听]"

echo
echo "=== 3. settings.json 关键字段 ==="
python3 -c "
import json, os
s = json.load(open(os.path.expanduser('~/.claude-mem/settings.json')))
for k in ['CLAUDE_MEM_RUNTIME','CLAUDE_MEM_PROVIDER','CLAUDE_MEM_CLAUDE_AUTH_METHOD','CLAUDE_MEM_MODEL']:
    print(f'  {k} = {s.get(k, \"(unset)\")}')"

echo
echo "=== 4. .env 关键字段（脱敏）==="
[ -f ~/.claude-mem/.env ] && grep -E "^(ANTHROPIC_BASE_URL|ANTHROPIC_MODEL|CLAUDE_MEM_)" ~/.claude-mem/.env | sed -E 's/=.*token.*/=<redacted>/i'

echo
echo "=== 5. 数据库统计 ==="
sqlite3 ~/.claude-mem/claude-mem.db "
SELECT 'total='||COUNT(*) FROM observations
UNION ALL SELECT 'last_id='||MAX(id) FROM observations
UNION ALL SELECT 'last_model='||generated_by_model FROM observations ORDER BY id DESC LIMIT 1
UNION ALL SELECT 'last_time='||datetime(created_at_epoch/1000,'unixepoch','localtime') FROM observations ORDER BY id DESC LIMIT 1;
" 2>/dev/null || echo "[数据库读不到]"

echo
echo "=== 6. 最近 10 分钟内的错误 ==="
find ~/.claude-mem/logs -name "claude-mem-*.log" -mmin -10 -exec \
  grep -hE "ERROR|API Error|Not logged in" {} \; 2>/dev/null | tail -5
```

跑这一份基本能定位 80% 问题。
