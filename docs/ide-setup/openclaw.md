# OpenClaw 接入

> **现状**：OpenClaw **不直接集成 claude-mem**。两者各管一段。需要协同时通过中央 KG（如 [kg-hub](https://github.com/commiao/kg-hub)）桥接。
> **难度**：N/A——不是装一下就行的事，是架构层面的分工。

## 别误会

| 工具 | 角色 |
|---|---|
| **claude-mem** | 自动捕获**低层工具调用**（Read / Edit / Bash）→ qwen 抽 obs → 流水账式记忆 |
| **OpenClaw** | agent 在工作中**人在回路**地手工提炼**高层胶囊**（Capsule）+ 概念条目（MEMORY.md）+ 知识文档 |

二者**没有直接耦合接口**。

## 为什么没接

历史上 OpenClaw 自己维护一个 trivial 的图谱（`notes/knowledge-graph/graph-*.json`，4 节点 4 边的 Task-Session 快照），价值很低。真正有用的关系（fixes / relates_to / diagnosed_by）都以**自然语言隐式存在于胶囊 markdown 里**——没有任何工具能跨工具调用。

## 三方协同路径

如果你既用 claude-mem 也用 OpenClaw，最务实的协同方式是**通过中央 KG 项目**（如 kg-hub）做后端聚合：

```
┌──────────────┐       ┌──────────────┐
│ claude-mem   │       │ OpenClaw     │
│ (obs)        │       │ (capsules)   │
└──────┬───────┘       └──────┬───────┘
       │                      │
       │ kg-hub ingester 拉 │
       └──────────┬───────────┘
                  ▼
          中央 KG (Memgraph/FalkorDB)
                  ▲
                  │ MCP RAG
                  │
       任意 IDE 通过 MCP 查询
```

kg-hub 是社区项目，独立演进。当前阶段在做的事：
- 把 OpenClaw 胶囊解析成 KG entities + edges
- 把 claude-mem obs 解析成 KG events
- 暴露统一 MCP 查询

## OpenClaw 端建议

如果你是 OpenClaw 用户，下面是**与 claude-mem 共存**的几条实操建议：

### 1. 不要让 OpenClaw 自己也跑 hook 写 claude-mem.db

OpenClaw 的工作方式（人在回路提炼胶囊）和 claude-mem（自动流水账）**职责不重叠**。让 claude-mem hook 跑你日常打开的 IDE（Claude Code / Codex），OpenClaw 不动 claude-mem.db。

### 2. 让 OpenClaw 产生的胶囊**通过文件系统**对外暴露

OpenClaw 默认把胶囊存在 `notes/capsules/`（或 VPS 上）。让这个目录可被 kg-hub ingester 读取（rsync / scp / 共享挂载 / WebDAV 同步），让 kg-hub 后端定期抽实体 + 关系入图。

### 3. MEMORY.md 的概念**作为图谱 seed**

OpenClaw 的 MEMORY.md 已经是高度结构化的概念条目（人手工梳理过的），是天然的 KG 节点。让 kg-hub 优先 ingest 这部分——它们是图谱的 "anchor points"。

### 4. 停掉 OpenClaw 自己的 `notes/knowledge-graph/`

如果你正在用 OpenClaw 的内置图谱（trivial 那个），可以停掉了——所有图谱能力交给 kg-hub。

## 查询方式

OpenClaw 产生的知识进入中央 KG 后，你可以从任意 IDE 通过 MCP 查询：

```
"查一下 OpenClaw 胶囊里关于飞书通知的最佳实践"
```

经 kg-hub MCP 工具（`kg_search` / `kg_episode_search`）返回。

## 这是不是"接入"

技术上 OpenClaw 没装 claude-mem 任何东西，叫"接入"有点勉强。准确说法是：**通过中央 KG 桥接，让 OpenClaw 的输出可被 claude-mem 用户消费**。

## 引用

- OpenClaw 仓库 / 项目主页（如果有）
- [kg-hub 项目设计](https://github.com/commiao/kg-hub) — 中央 KG 实现细节

## 不在 OpenClaw 接入范围内的事

- ❌ 让 OpenClaw 实时收到 claude-mem obs 更新
- ❌ 让 claude-mem 知道哪些胶囊存在
- ❌ 让 OpenClaw 跑 claude-mem 的 hook 链路

这些需求都不该往这两个工具的耦合方向做，**应该走中央 KG**。
