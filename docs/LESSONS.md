# 工程教训沉淀

> 在多 IDE / 多设备 / 多 LLM provider 路上踩出来的**通用经验**，不限于 claude-mem 本身。如果你接手类似的 agent / 中间件 / 多 IDE 工具链项目，这些教训值得提前内化。

## L1. `.env` 加载时机不可靠，运行时开关要走 OS 级注入

**症状**：写在 `.env` 的环境变量"看起来生效了"，但**早期模块**（如 chroma module 构造）已经在 `.env` 加载之前跑过，运行时开关无效。

**通用规则**：
- **凭证 / 密钥**（请求时才用到的）→ `.env` OK
- **运行时开关 / 启动期决策**（决定 module 是否加载、连不连后端的）→ **必须放进程启动环境**：
  - macOS LaunchAgent: `<key>EnvironmentVariables</key>`
  - Linux systemd: `Environment=`
  - Docker: `-e` 或 `environment:`
  - Windows: 用户级环境变量 / `setx`

**反模式**：
> "我把 `CLAUDE_MEM_CHROMA_ENABLED=false` 写到了 .env，但 chroma 还在连。" → 因为它在 .env 加载前就构造好了。

## L2. IDE Agent 在沙盒里反复试错的反模式

**症状**：IDE agent（Qoder / Cursor / Copilot 等）想做某个文件操作时，反复换路径 / 反复试错，最后接受次优解。

**反面案例**（2026-05-31，Qoder agent 移文件）：
1. 想用 `mv`，工具不支持 → 改 `cp` + `delete_file`
2. 顺序反向：**先** delete_file，**后** cp → 源没了，失败
3. 改用 `create_file` 到工作区外路径 → 沙盒拒绝
4. 最终在原位重建文件 + 加标签说"我知道这归属错了"

**根因**：agent **没在动手前确认沙盒边界**。

**通用规则**（写进所有 IDE agent 的 onboarding prompt）：
1. **动手前先做能力自检**：我能写到哪些目录？我能跑哪些 shell？
2. **删除前先备份**：「Read → Write 新位置 → Verify → Delete 旧位置」
3. **碰到边界立刻求助**：「我的 X 工具不能 Y，你能不能用 shell 跑一条命令？」**不要在沙盒里反复试错**
4. **不同 agent 沙盒能力不同**：Claude Code 几乎无限制 / Cursor 受限于 run_in_terminal / Qoder 严格限工作区内 / Codex 受 hook 链路约束

详见仓库根目录 `IDE-AGENT-ONBOARDING-TEMPLATE.md`（如果你 fork 了本仓库，可以引用这个）。

## L3. Hook 抢占 LaunchAgent 是普遍的 race

**症状**：开机自启的 worker / daemon，被某个 IDE 的 SessionStart hook"抢救"启动，结果脱离 LaunchAgent / systemd 管理，失去崩溃自愈。

**Codex 案例**：详见 [TROUBLESHOOTING §9](TROUBLESHOOTING.md#codex-hook-race)。

**通用规则**：
- 任何"如果服务没在跑，我就启动它"的 hook 都要小心
- 优先做"检查服务是否在跑，不在就**告诉用户**"，不要静默拉起
- 即使要拉起，也要用 service manager 的接口（`launchctl kickstart` / `systemctl start --no-block`），而不是直接 spawn

## L4. 跨 IDE 集成时，先查 sandbox 文件再查代码

**症状**：装好 claude-mem，但某个 IDE 看不到 MCP 工具。直觉去看上游 issue / 重装 / 重启——通常没用。

**实战方法**：先看 **IDE 自己的 MCP 配置文件**。每个 IDE 都有自己的位置：

| IDE | MCP 配置位置 |
|---|---|
| Claude Code | `~/.claude/settings.json` 里的 `mcpServers` 字段 |
| Cursor | `~/.cursor/mcp.json` |
| Codex CLI / 桌面版 | `~/.codex/config.toml` |
| Qoder | `~/.qoder/mcp.json`（或经 muxcp 聚合） |

**第一步永远是**：`cat 那个文件，看是否有你期望的 server 注册`。

## L5. 上游"宣称的"和"实测的"经常不一致

**实例**：
- OpenClaw 宣称 "179 个胶囊"，盘上实际可用 49 个
- 上游 docs 说"chroma 可选"，实测启用后 search 100% 超时
- Cursor 宣称"完整 MCP 支持"，实测 hook 不支持，只读

**通用规则**：
- 任何关键决策前，先**实测一次最小可用样本**
- 实测数字记进文档 ROADMAP，作为"实际"对照"宣称"
- 上游 README 是入门指引，不是生产手册

## L6. 多 IDE 共存时，project 名是争议焦点

**症状**：Cursor 在 `~/projects/foo` 打开，Claude Code 在 `~/work/foo` 打开（同一个 repo 不同 worktree），claude-mem 的 project 字段不一致，导致跨设备/跨 IDE 的同一 project 记忆查不到一起。

**根因**：claude-mem 默认用 `basename(cwd)` 推断 project 名。worktree / 软链 / 不同设备的 clone 路径都让 basename 漂移。

**通用规则**：
- 在每个 project 根目录放一个 `.claude-mem/project.json` 显式声明 `project_name`
- 或者用 git remote URL 的最后段作为 project 名（更稳）
- 跨设备同步前确认所有设备 project 名一致

## L7. AI provider 兼容端点的"model 名"是隐藏雷区

**症状**：切到国内厂商兼容端点后，claude-mem 报 `invalid_parameter_error: model 'claude-haiku-*'`。

**根因**：兼容端点声明"兼容 Anthropic Messages 协议"，但**只兼容协议**，不兼容 Claude 的 model 名空间。

**通用规则**：
- 任何"兼容 X 协议"的端点，model 名是**它们自己的**
- 客户端 SDK 如果写死了 Claude model 名，**必须有 override 机制**
- 看 SDK 是否暴露 `CLAUDE_MEM_MODEL` / `OVERRIDE_MODEL` 之类的参数；没有就 fork

## L8. 文档归属比文档内容更重要

**症状**：写了一份"Qoder 接入 claude-mem"指南，放到 sd-server 业务仓库的 `docs/` 下，结果团队成员误以为 claude-mem 是 sd-server 的一部分。

**通用规则**：
- **公共能力的文档放公共位置**，不要污染业务仓库
- 文档头部明确"归属说明" + 引用上游
- 跨项目共享的脚本 / 配置 / 文档，集中放 `~/.config/<tool>/` 或独立仓库

## L9. WebDAV / 云盘同步 + 设备特定路径 = 灾难

**症状**：用 WebDAV 同步 IDE 配置，配置里有 `/Users/myname/...` 硬编码路径，到另一台设备上路径不存在。

**通用规则**：
- 同步的配置文件**只放可移植的**（相对路径、`$HOME` 占位符、版本号等）
- 设备特定的部分用本地 override：
  - cc-switch 模式：synced `current.yaml` + local `local.env`
  - 各 IDE 也都有"workspace config" vs "user config" vs "machine config" 三层

## L10. "马上就 push" 之前总要跑一次脱敏 grep

**症状**：HANDOVER.md 含个人邮箱、组织名、API token 暗示，差点 commit 到公共仓库。

**通用规则**：在第一次 commit 前跑：

```bash
grep -rE "(your-real-email|your-org|sk-sp-[a-z0-9]+|/Users/<your-username>/)" . 2>/dev/null
```

把所有结果替换成占位符（`<your-email>` / `<your-org>` / `<your-api-key>` / `~/`）。

加到 `.gitignore`：

```
.env
.env.*
*.bak
PRIVATE-*.md
NOTES-PRIVATE.md
```

---

## 元教训：把这些教训本身放进 KG

如果你建了知识图谱（如 [kg-hub](https://github.com/commiao/kg-hub) 这类项目），把上面每条 lesson 抽成一个 `Lesson` 节点入图。

下次任何 agent 在你这台机器上做类似的事（沙盒移文件 / 配 .env 运行时开关 / 跨 IDE 集成），SessionStart 注入会把相关 lesson 喂过去，避免重复踩坑。

这是 claude-mem + KG 协同的真正价值——不是"记住所有事"，是"自动给未来的 agent 装上下文"。
