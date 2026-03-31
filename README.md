# Claude Code 产品架构说明手册

## 产品概述

Claude Code 是 Anthropic 开发的终端 AI 编程助手 CLI 工具。用户通过终端与 Claude 模型交互，执行软件工程任务：编辑文件、运行命令、搜索代码、协调工作流。

### 技术栈一览

| 层面 | 技术选型 |
|------|---------|
| 运行时 | Bun |
| 语言 | TypeScript (strict mode) |
| 终端 UI | React 18 + Ink (自定义终端渲染器) |
| CLI 解析 | Commander.js (extra-typings) |
| Schema 验证 | Zod v4 |
| 代码搜索 | ripgrep (rg CLI) |
| LLM API | Anthropic SDK (@anthropic-ai/sdk) |
| 协议 | MCP SDK, LSP |
| 认证 | OAuth 2.0, JWT, macOS Keychain |
| Feature Flags | GrowthBook |
| 遥测 | OpenTelemetry + gRPC |
| IDE 桥接 | 自定义 WebSocket + JWT 协议 |

### 代码规模

| 指标 | 数值 |
|------|------|
| TypeScript 源文件 | ~1,900 |
| 代码行数 | 512,000+ |
| 内置工具 | 40+ |
| 斜杠命令 | 50+ |
| React 组件 | 389 |
| React Hooks | 104 |
| 服务模块 | 130+ |
| 工具模块 | 184 |

## 阅读指南

本手册按**核心流程 -> 子系统详解 -> 附录**三个层次组织。建议首次阅读按序号顺序通读核心部分(01-06)，子系统部分(07-12)可按需查阅。

### 核心架构（主干线，建议顺序阅读）

| 文档 | 内容 |
|------|------|
| [01-core-architecture-overview](01-core-architecture-overview.md) | 系统架构全景、分层设计、模块关系 |
| [02-core-startup-and-initialization](02-core-startup-and-initialization.md) | 启动序列、并行预取、Feature Flag、延迟加载 |
| [03-core-query-engine](03-core-query-engine.md) | QueryEngine 查询循环、流式处理、Token 计数 |
| [04-core-tool-system](04-core-tool-system.md) | Tool 接口、40+ 工具分类、工具级权限模型 |
| [05-core-command-system](05-core-command-system.md) | 50+ 命令注册、分发、可用性过滤 |
| [06-core-state-and-ui](06-core-state-and-ui.md) | AppState 设计、React + Ink 渲染、组件架构 |

### 子系统详解（按需查阅）

| 文档 | 内容 |
|------|------|
| [07-subsystem-bridge](07-subsystem-bridge.md) | IDE/远程桥接协议、状态机、传输抽象 |
| [08-subsystem-mcp](08-subsystem-mcp.md) | MCP 协议集成、传输类型、工具发现与封装 |
| [09-subsystem-agent-and-coordinator](09-subsystem-agent-and-coordinator.md) | Agent 生成、Coordinator 模式、多智能体协作 |
| [10-subsystem-config-and-permissions](10-subsystem-config-and-permissions.md) | 配置层级、MDM 企业管理、权限体系 |
| [11-subsystem-context-and-compaction](11-subsystem-context-and-compaction.md) | CLAUDE.md 发现、上下文收集、压缩策略 |
| [12-subsystem-skills-and-plugins](12-subsystem-skills-and-plugins.md) | Skills 加载与执行、Plugin 生命周期 |

### 附录

| 文档 | 内容 |
|------|------|
| [13-appendix-design-patterns](13-appendix-design-patterns.md) | 设计模式、认证系统、遥测、会话持久化、投机执行、Vim/语音/Buddy 等补充子系统 |

### PlantUML 架构图

所有架构图的 PlantUML 源文件位于 [diagrams/](diagrams/) 目录。可使用 PlantUML 工具或在线渲染器生成图片。

| 图表 | 类型 | 对应文档 |
|------|------|---------|
| [system-architecture.puml](diagrams/system-architecture.puml) | 组件图 | 01-架构全景 |
| [startup-sequence.puml](diagrams/startup-sequence.puml) | 时序图 | 02-启动流程 |
| [query-loop.puml](diagrams/query-loop.puml) | 活动图 | 03-查询引擎 |
| [tool-permission-flow.puml](diagrams/tool-permission-flow.puml) | 活动图 | 04-工具系统 |
| [bridge-states.puml](diagrams/bridge-states.puml) | 状态图 | 07-Bridge |
| [agent-coordination.puml](diagrams/agent-coordination.puml) | 时序图 | 09-Agent 协调 |
| [config-hierarchy.puml](diagrams/config-hierarchy.puml) | 组件图 | 10-配置与权限 |
| [compaction-strategy.puml](diagrams/compaction-strategy.puml) | 活动图 | 11-上下文与压缩 |

## 源码目录结构速览

```
src/
├── main.tsx                 # CLI 入口 (4,683 行)
├── commands.ts              # 命令注册表
├── tools.ts                 # 工具注册表
├── Tool.ts                  # 工具类型定义 (29KB)
├── QueryEngine.ts           # LLM 查询引擎 (46KB)
├── context.ts               # 系统/用户上下文
├── cost-tracker.ts          # Token 成本追踪
├── query.ts                 # 查询处理 (1,729 行)
├── history.ts               # 会话历史
│
├── commands/                # 斜杠命令 (189 文件)
├── tools/                   # 工具实现 (184 文件, 45 子目录)
├── components/              # React/Ink UI 组件 (389 文件)
├── hooks/                   # React Hooks (104 文件)
├── services/                # 外部服务集成 (130 文件)
├── utils/                   # 工具函数 (564 文件)
├── ink/                     # 自定义 Ink 渲染器 (96 文件)
├── state/                   # 全局状态管理
├── bridge/                  # IDE/远程桥接 (31 文件)
├── types/                   # TypeScript 类型定义
├── entrypoints/             # 入口包装
├── bootstrap/               # 引导状态
├── constants/               # 常量定义
├── skills/                  # 技能框架
├── tasks/                   # 任务系统
├── context/                 # Context Providers
├── memdir/                  # 持久化记忆
├── keybindings/             # 键绑定
├── migrations/              # 配置迁移
├── coordinator/             # 多 Agent 协调
├── vim/                     # Vim 模式
├── voice/                   # 语音输入
├── plugins/                 # 插件系统
├── server/                  # 服务器模式
├── screens/                 # 全屏 UI
├── query/                   # 查询管道
├── remote/                  # 远程会话
├── schemas/                 # Zod Schema
├── cli/                     # CLI 工具
├── buddy/                   # Companion 精灵
└── native-ts/               # 原生 TS 工具
```
