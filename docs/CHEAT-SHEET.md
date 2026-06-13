# 命令速查表

> 一页纸，闭着眼能找到的命令。按"想干什么"组织。

## 健康检查 / 状态

```bash
# Worker 是否在跑
curl -s http://localhost:37701/api/health | python3 -m json.tool

# Worker PID 与版本
curl -s http://localhost:37701/api/health | python3 -c "import sys,json;d=json.load(sys.stdin);print(f'pid={d[\"pid\"]} version={d[\"version\"]} authMethod={d[\"ai\"][\"authMethod\"]}')"

# 端口监听验证（37700 + uid%100）
lsof -nP -iTCP:37701 -sTCP:LISTEN

# 数据库观察数
sqlite3 ~/.claude-mem/claude-mem.db "SELECT COUNT(*) FROM observations;"
```

## 启停 worker

### macOS (LaunchAgent)
```bash
# 重启（保持开机自启）
launchctl kickstart -k gui/$(id -u)/com.claude-mem.worker

# 看 LaunchAgent 状态
launchctl print gui/$(id -u)/com.claude-mem.worker | grep -E "state|active count"

# 关闭并禁用自启
launchctl bootout gui/$(id -u)/com.claude-mem.worker

# 重新启用
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/com.claude-mem.worker.plist
```

### Linux (systemd)
```bash
systemctl --user restart claude-mem-worker
systemctl --user stop claude-mem-worker
systemctl --user start claude-mem-worker
systemctl --user status claude-mem-worker
journalctl --user -u claude-mem-worker -n 50
```

### Windows (Task Scheduler)
```powershell
# 重启
schtasks /End /TN "claude-mem-worker"
schtasks /Run /TN "claude-mem-worker"

# 看状态
schtasks /Query /TN "claude-mem-worker" /V /FO LIST
```

### 任意系统（暴力 fallback）
```bash
# 杀掉所有 worker 进程，让 LaunchAgent / systemd / TaskScheduler 自动拉起
kill $(lsof -nP -iTCP:37701 -sTCP:LISTEN -t)
sleep 5
curl http://localhost:37701/api/health
```

## 查看数据

```bash
# 最近 N 条观察
sqlite3 -header -column ~/.claude-mem/claude-mem.db \
  "SELECT id, type, generated_by_model,
          datetime(created_at_epoch/1000,'unixepoch','localtime') as t,
          substr(title,1,60) as title
   FROM observations ORDER BY id DESC LIMIT 10;"

# 按 project 分布
sqlite3 -header -column ~/.claude-mem/claude-mem.db \
  "SELECT project, COUNT(*) FROM observations GROUP BY project ORDER BY 2 DESC;"

# 按 LLM 分布（看你的 provider 是否生效）
sqlite3 -header -column ~/.claude-mem/claude-mem.db \
  "SELECT generated_by_model, COUNT(*) FROM observations
   GROUP BY generated_by_model ORDER BY 2 DESC;"

# 按来源 IDE 分布
sqlite3 -header -column ~/.claude-mem/claude-mem.db \
  "SELECT platform_source, COUNT(*) FROM sdk_sessions
   GROUP BY platform_source;"

# 取一条完整观察
sqlite3 -line ~/.claude-mem/claude-mem.db \
  "SELECT * FROM observations WHERE id = 42;"

# 最近 5 分钟有没有新观察（验证 hook 在工作）
sqlite3 ~/.claude-mem/claude-mem.db \
  "SELECT COUNT(*) FROM observations
   WHERE created_at_epoch/1000 > strftime('%s','now')-300;"
```

## 日志

```bash
# 主应用日志（最新一天）
tail -100 ~/.claude-mem/logs/claude-mem-$(date +%Y-%m-%d).log

# hook 触发记录
grep "\[HOOK " ~/.claude-mem/logs/claude-mem-*.log | tail -20

# 错误
grep ERROR ~/.claude-mem/logs/claude-mem-*.log | tail -20

# LaunchAgent / systemd 自己的输出
tail -20 ~/.claude-mem/logs/launchd.out.log    # macOS
journalctl --user -u claude-mem-worker -n 50   # Linux

# Hook runner 错误（issue #2188 类的）
tail -30 ~/.claude-mem/logs/runner-errors.log
```

## 配置 / 凭证

```bash
# 看 settings.json（runtime / provider / model）
cat ~/.claude-mem/settings.json

# 看 .env（脱敏显示，不打印 token）
sed -E 's/(TOKEN|KEY)=.*/\1=<redacted>/' ~/.claude-mem/.env

# .env 权限必须 600
chmod 600 ~/.claude-mem/.env
ls -la ~/.claude-mem/.env  # 预期 -rw-------
```

## 升级 / 卸载

```bash
# 升级到最新版本
npx -y claude-mem@latest install --upgrade

# 看当前装的版本
ls ~/.claude/plugins/marketplaces/thedotmack/plugin.json 2>/dev/null && \
  jq -r '.version' ~/.claude/plugins/marketplaces/thedotmack/plugin.json

# 卸载（保留数据）
launchctl bootout gui/$(id -u)/com.claude-mem.worker  # macOS
rm ~/Library/LaunchAgents/com.claude-mem.worker.plist
rm -rf ~/.claude/plugins/marketplaces/thedotmack/

# 完全卸载（包括数据，注意备份）
rm -rf ~/.claude-mem/
```

## 切 LLM Provider

```bash
# 备份当前配置
cp ~/.claude-mem/.env ~/.claude-mem/.env.bak.$(date +%s)

# 编辑 .env 改 ANTHROPIC_BASE_URL / ANTHROPIC_AUTH_TOKEN
${EDITOR:-vim} ~/.claude-mem/.env

# 改 model（要在 settings.json，不是 .env）
jq '.CLAUDE_MEM_MODEL = "qwen3.6-plus"' ~/.claude-mem/settings.json > /tmp/s.json && \
  mv /tmp/s.json ~/.claude-mem/settings.json

# 重启 worker
launchctl kickstart -k gui/$(id -u)/com.claude-mem.worker  # macOS
systemctl --user restart claude-mem-worker                  # Linux

# 验证新 provider 生效
curl -s http://localhost:37701/api/health | python3 -c "import sys,json;print(json.load(sys.stdin)['ai'])"

# 跑几条命令触发新观察后，看是否用新模型生成
sleep 30
sqlite3 ~/.claude-mem/claude-mem.db \
  "SELECT id, generated_by_model FROM observations ORDER BY id DESC LIMIT 3;"
```

## 备份 / 恢复

```bash
# 全量备份
tar -czf claude-mem-backup-$(date +%Y%m%d).tar.gz -C $HOME .claude-mem/

# 只备份数据库（不停服务，sqlite hot backup）
sqlite3 ~/.claude-mem/claude-mem.db ".backup '/tmp/claude-mem-$(date +%Y%m%d).db'"

# 恢复
# 1. 停 worker
launchctl bootout gui/$(id -u)/com.claude-mem.worker
# 2. 解压回原位
tar -xzf claude-mem-backup-XXXXXXXX.tar.gz -C $HOME
# 3. 起 worker
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/com.claude-mem.worker.plist
```

## 性能 / 调试

```bash
# 看 worker 资源占用
ps aux | grep -E "bun.*worker-service" | grep -v grep

# 看 chroma-mcp 是否在跑（如果你启用了语义搜索）
ps aux | grep chroma-mcp | grep -v grep

# 看 SQLite 数据库大小
ls -lh ~/.claude-mem/claude-mem.db

# 看 chroma 数据目录大小（如启用）
du -sh ~/.claude-mem/chroma/

# 强制清理 hooks 卡死时的 sdk session
sqlite3 ~/.claude-mem/claude-mem.db "DELETE FROM sdk_sessions WHERE status = 'stuck';"
```
