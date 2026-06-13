# claude-mem Integration Cookbook

> **实战配方集**：在不同 OS（macOS / Linux / Windows）和不同 AI IDE（Claude Code / Cursor / Codex / Qoder / OpenClaw）上跑通 [claude-mem](https://github.com/thedotmack/claude-mem) 持久化记忆系统的部署手册、踩坑记录与运维经验。

## 这不是什么

- ❌ **不是 claude-mem 的源码** —— 见上游 [thedotmack/claude-mem](https://github.com/thedotmack/claude-mem)
- ❌ **不是 claude-mem 的官方安装文档** —— 见上游 [docs.claude-mem.ai](https://docs.claude-mem.ai/)
- ❌ **不是 claude-mem 的替代品** —— 上游的能力完整且持续演进

## 这是什么

- ✅ **跨 IDE 接入实战**：上游官方文档只覆盖 Claude Code / Gemini CLI / OpenCode；本仓库补 **Cursor / Codex (CLI + 桌面版) / Qoder / OpenClaw** 实测过的接入步骤
- ✅ **跨 OS 部署对照**：macOS（深度实战） + Linux（验证过）+ Windows（基于上游文档 + 推断，**未实战**，明确标注）
- ✅ **故障排查 14 条**：实测踩过的坑，每条含**触发条件 + 检测命令 + 修复方法 + 自然恢复条件**
- ✅ **多 IDE 共存运维**：当你想同时在 Claude Code + Cursor + Codex 用 claude-mem，会遇到的 race condition、worker 抢占、配置漂移等问题与解
- ✅ **教训沉淀**：抽象出"在 IDE 沙盒里反复试错的反模式"等通用经验

## 谁该读

| 你是 | 怎么用 |
|---|---|
| 第一次装 claude-mem | 从 [docs/INSTALL.md](docs/INSTALL.md) 开始 |
| 已经装好，想接入新 IDE | 直奔 [docs/ide-setup/](docs/ide-setup/) 对应文档 |
| 用着用着遇到问题 | [docs/TROUBLESHOOTING.md](docs/TROUBLESHOOTING.md) |
| 想知道日常运维命令 | [docs/CHEAT-SHEET.md](docs/CHEAT-SHEET.md) |
| 想了解每个 IDE 的能力边界 | [docs/ide-setup/](docs/ide-setup/) 各文档末尾的"能力矩阵" |
| 想给自己/团队建个新机器部署 | [docs/INSTALL.md](docs/INSTALL.md) + 对应 IDE setup |

## 文档地图

```
claude-mem-integration-cookbook/
├── README.md                     ← 你正在看
├── LICENSE                       ← Apache-2.0（与上游一致）
├── .gitignore
└── docs/
    ├── INSTALL.md                ← 跨平台从零安装（macOS / Linux / Windows）
    ├── USAGE.md                  ← 日常使用：MCP 工具、查询、运维
    ├── CHEAT-SHEET.md            ← 常用命令速查
    ├── TROUBLESHOOTING.md        ← 14 条已知坑
    ├── LESSONS.md                ← 工程教训沉淀
    └── ide-setup/
        ├── claude-code.md        ← Claude Code（读+写完整）
        ├── cursor.md             ← Cursor（只读 via MCP）
        ├── codex-desktop.md      ← Codex 桌面版（读+写）
        ├── codex-cli.md          ← Codex CLI
        ├── qoder.md              ← Qoder IDE
        └── openclaw.md           ← OpenClaw（高层胶囊，配合 kg-hub）
```

## 各 IDE 能力一览

| IDE | 读记忆 | 写记忆 | 跨设备同步 | 备注 |
|---|---|---|---|---|
| Claude Code | ✅ | ✅ | ✅ | 主力 IDE，hook 完整 |
| Cursor | ✅ | ❌ | ✅ (读) | 通过 MCP 查询 |
| Codex 桌面版 | ✅ | ✅ | ✅ | 需在 GUI `/plugins` 启用 + Approve 7 hooks |
| Codex CLI | ✅ | ✅ | ✅ | hook 与桌面版同一套 |
| Qoder | ✅ | ⚠️ 视配置 | ✅ (读) | 经 muxcp 中转 |
| OpenClaw | — | — | — | 不直接集成；用 kg-hub 桥接 |

详见 [docs/ide-setup/](docs/ide-setup/)。

## 用到的关联项目

| 项目 | 角色 | 链接 |
|---|---|---|
| **claude-mem** | 持久化记忆核心 | https://github.com/thedotmack/claude-mem |
| **cc-switch** | 跨设备/跨 IDE 配置同步 | https://github.com/farion1231/cc-switch |
| **muxcp** | 多 MCP server 聚合代理 | 配置位于 `~/.config/muxcp/` |
| **kg-hub** | 多源知识图谱（在 claude-mem 之上聚合的中央 KG） | https://github.com/commiao/kg-hub |

## 全栈架构

本仓库是**个人 AI 工程记忆栈**的一部分。完整全栈视图（含 kg-hub / cc-switch / muxcp / 各 IDE 协同关系）见：

→ **[kg-hub/ARCHITECTURE.md](https://github.com/commiao/kg-hub/blob/main/ARCHITECTURE.md)** ← single source of truth for the whole stack

## 贡献

任何 IDE 接入新踩到的坑、新发现的 race / 边界 case，欢迎 PR 到 `docs/TROUBLESHOOTING.md` 或对应的 `docs/ide-setup/*.md`。

## 声明

本仓库与上游 [thedotmack/claude-mem](https://github.com/thedotmack/claude-mem) 无官方关系，是社区维护的实战经验文档。所有 claude-mem 本身的能力、bug、限制以**上游为准**。

## License

Apache-2.0，与上游一致。详见 [LICENSE](LICENSE)。
