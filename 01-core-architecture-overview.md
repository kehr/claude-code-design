# 01 系统架构全景

## 架构图

> 完整 PlantUML 源文件: [diagrams/system-architecture.puml](diagrams/system-architecture.puml)

```
┌─────────────────────────────────────────────────────────────────┐
│                        Entry Layer                               │
│  main.tsx -> cli.tsx (Commander.js) -> init.ts                   │
└──────────────────────────┬──────────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────────┐
│                       Command Layer                              │
│  commands.ts (Registry) -> 50+ Slash Commands                    │
└──────────────────────────┬──────────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────────┐
│                    Query Engine Layer                             │
│  QueryEngine.ts (LLM Loop) + query.ts + cost-tracker.ts         │
└──────────────────────────┬──────────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────────┐
│                        Tool Layer                                │
│  tools.ts (Registry) -> 40+ Tools (Bash/FileRead/Grep/Agent...) │
│  ┌─────────────┐ ┌──────────────┐ ┌────────────────┐           │
│  │  AgentTool   │ │   MCPTool    │ │   SkillTool    │           │
│  │ (Sub-agents) │ │ (MCP Proto)  │ │ (Skill Exec)   │           │
│  └──────┬──────┘ └──────┬───────┘ └───────┬────────┘           │
└─────────┼───────────────┼─────────────────┼─────────────────────┘
          │               │                 │
┌─────────▼───────────────▼─────────────────▼─────────────────────┐
│                       Service Layer                              │
│  api/claude.ts  services/mcp/  services/oauth/  analytics/       │
│  compact/       lsp/           plugins/         policyLimits/    │
└──────────────────────────┬──────────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────────┐
│                     State & UI Layer                             │
│  AppState (Store) -> React + Ink -> 389 Components + 104 Hooks  │
└──────────────────────────┬──────────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────────┐
│                    Bridge & Remote Layer                         │
│  bridge/ (IDE Bridge)  coordinator/ (Multi-Agent)  remote/       │
└─────────────────────────────────────────────────────────────────┘
```

## 分层设计原则

Claude Code 采用六层架构，每一层有明确的职责边界：

### Entry Layer (入口层)

**职责**: 应用启动、CLI 参数解析、初始化编排

**核心文件**:
- `src/main.tsx` (4,683 行) - 总入口，编排启动序列
- `src/entrypoints/cli.tsx` (39KB) - Commander.js CLI 包装
- `src/entrypoints/init.ts` (13KB) - 全局初始化（配置、Feature Flags、策略加载）

**设计决策**: 入口层负责并行预取（MDM 设置、Keychain 令牌），在模块导入的 ~135ms 窗口期内完成 I/O，最大化启动速度。

### Command Layer (命令层)

**职责**: 斜杠命令注册、路由、可用性过滤

**核心文件**:
- `src/commands.ts` (25KB) - 命令注册表，支持内置命令、Skills、Plugins 三类来源
- `src/commands/` (189 文件) - 具体命令实现

**设计决策**: 命令按优先级分层加载: bundled skills > builtinPlugin skills > skill dir commands > workflow commands > plugin commands > plugin skills > builtin commands。先注册的同名命令优先，后注册的不会覆盖。

### Query Engine Layer (查询引擎层)

**职责**: LLM API 调用、流式响应处理、工具分发循环、Token 计数

**核心文件**:
- `src/QueryEngine.ts` (46KB) - 查询引擎类，async generator 驱动
- `src/query.ts` (1,729 行) - 消息处理、API 调用封装
- `src/cost-tracker.ts` (10KB) - Token 成本追踪

**设计决策**: 使用 async generator 模式实现流式处理，允许调用方逐消息处理而非等待完整响应。查询循环自动执行工具调用并将结果作为下一轮用户消息回传。

### Tool Layer (工具层)

**职责**: 工具注册、输入验证、权限检查、执行、进度追踪

**核心文件**:
- `src/tools.ts` - 工具注册表
- `src/Tool.ts` (29KB) - 核心类型定义 (Tool, ToolUseContext, PermissionResult)
- `src/tools/` (184 文件, 45 子目录) - 具体工具实现

**设计决策**: 统一的 `Tool` 接口使得内置工具、MCP 工具、Plugin 工具可以用相同方式注册和调用。`buildTool()` 函数提供安全默认值（fail-closed: 默认不并发安全、默认非只读）。

### Service Layer (服务层)

**职责**: 外部服务集成（API、MCP、OAuth、分析、LSP、插件）

**核心文件**:
- `src/services/api/claude.ts` - Anthropic API 客户端
- `src/services/mcp/` (25 文件) - MCP 服务器管理
- `src/services/oauth/` - OAuth 2.0 认证
- `src/services/analytics/` - GrowthBook + OpenTelemetry
- `src/services/compact/` (11 文件) - 上下文压缩
- `src/services/lsp/` - 语言服务器协议
- `src/services/plugins/` - 插件加载器

**设计决策**: 服务层通过依赖注入与上层解耦。QueryEngine 和 Tool 通过 `ToolUseContext` 中的回调访问服务，而非直接导入，避免了循环依赖。

### State & UI Layer (状态与 UI 层)

**职责**: 全局状态管理、终端 UI 渲染

**核心文件**:
- `src/state/AppState.tsx` (23KB) - 全局状态定义与 Provider
- `src/state/AppStateStore.ts` - Store 实现
- `src/ink/` (96 文件) - 自定义 Ink 渲染器
- `src/components/` (389 文件) - React 组件

**设计决策**: 使用类 Zustand 的 Store 模式，通过 `useSyncExternalStore` 与 React 集成。核心字段标记为 `DeepImmutable`，部分高频变更字段（tasks, mcp.clients）保持可变以优化性能。

### Bridge & Remote Layer (桥接与远程层)

**职责**: IDE 集成、远程会话、多 Agent 协调

**核心文件**:
- `src/bridge/` (31 文件, 532KB) - IDE 桥接协议
- `src/coordinator/` - 多 Agent 协调器
- `src/remote/` - 远程会话逻辑

**设计决策**: Bridge 通过传输抽象层支持 v1(WS + POST) 和 v2(SSE + CCRClient) 两种协议版本，上层代码无需感知协议差异。

## 模块依赖关系

```
main.tsx
├── commands.ts (50+ slash commands)
├── tools.ts (40+ tools)
│   ├── BashTool, FileReadTool, FileEditTool, GlobTool, GrepTool ...
│   ├── AgentTool -> QueryEngine (recursive)
│   ├── MCPTool -> services/mcp/
│   └── SkillTool -> services/plugins/
├── QueryEngine.ts
│   ├── services/api/claude.ts (Anthropic API)
│   ├── services/compact/ (context compaction)
│   └── state/AppStateStore.ts
├── state/AppState.tsx
│   └── React + Ink components
├── bridge/replBridge.ts
│   └── QueryEngine (bridge-initiated queries)
├── utils/ (330+ modules)
│   ├── permissions/ (permission checks)
│   ├── settings/ (config loading)
│   ├── claudemd.ts (context discovery)
│   └── auth.ts, git.ts, messages.ts ...
└── bootstrap/state.ts (pre-init caches)
```

**循环依赖处理策略**:
- 懒加载 `require()`: 延迟到运行时解析
- 类型提取到独立 `types/` 文件
- `bootstrap/state.ts` 作为模块级缓存打破导入环
- Getter 函数: `getTeammateUtils()`, `getTeammateModeSnapshot()` 避免直接导入

## 代码分布统计

| 模块分类 | 文件数 | 占比 | 说明 |
|---------|--------|------|------|
| utils/ | 564 | ~31% | 设置、权限、Bash、Git、认证、模型选择等 |
| components/ | 389 | ~21% | UI 渲染、模态框、对话框、消息视图等 |
| commands/ | 189 | ~10% | 斜杠命令处理器 |
| tools/ | 184 | ~10% | 工具实现 |
| services/ | 130 | ~7% | API、MCP、OAuth、LSP、分析、插件 |
| hooks/ | 104 | ~6% | React Hooks (权限、UI 状态等) |
| ink/ | 96 | ~5% | 自定义 React reconciler 终端渲染 |
| bridge/ | 31 | ~2% | IDE/远程桥接 |
| 其他 | ~197 | ~8% | 类型、状态、键绑定、迁移、上下文等 |
