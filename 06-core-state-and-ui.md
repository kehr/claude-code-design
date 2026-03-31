# 06 状态管理与 UI 渲染

## 概述

Claude Code 使用类 Zustand 的全局 Store 管理状态，配合 React + Ink 进行终端 UI 渲染。状态层与 UI 层通过 `useSyncExternalStore` 桥接，实现响应式更新。

## AppState 全局状态

### 状态结构

```typescript
// src/state/AppStateStore.ts

export type AppState = DeepImmutable<{
  // 模型与渲染
  settings: SettingsJson                       // 用户设置
  verbose: boolean
  mainLoopModel: ModelSetting                  // 当前模型
  mainLoopModelForSession: ModelSetting        // 会话级模型
  statusLineText: string | undefined           // 状态栏文本
  isBriefOnly: boolean

  // UI 状态
  expandedView: 'none' | 'tasks' | 'teammates'
  selectedIPAgentIndex: number
  coordinatorTaskIndex: number
  viewSelectionMode: 'none' | 'selecting-agent' | 'viewing-agent'
  footerSelection: FooterItem | null
  spinnerTip?: string

  // Agent 上下文
  agent: string | undefined
  kairosEnabled: boolean

  // 远程连接状态
  remoteSessionUrl: string | undefined
  remoteConnectionStatus: 'connecting' | 'connected' | 'reconnecting' | 'disconnected'
  remoteBackgroundTaskCount: number

  // Bridge 状态 (10+ 字段)
  replBridgeEnabled: boolean
  replBridgeExplicit: boolean
  replBridgeOutboundOnly: boolean
  replBridgeConnected: boolean
  replBridgeSessionActive: boolean
  replBridgeReconnecting: boolean
  replBridgeConnectUrl?: string
  replBridgeSessionUrl?: string
  replBridgeEnvironmentId?: string
  replBridgeSessionId?: string
  replBridgeError?: string
  replBridgeInitialName?: string
  showRemoteCallout: boolean

  // 权限
  toolPermissionContext: ToolPermissionContext
}> & {
  // 可变状态 (排除在 DeepImmutable 之外)
  tasks: { [taskId: string]: TaskState }
  agentNameRegistry: Map<string, AgentId>
  foregroundedTaskId?: string
  viewingAgentTaskId?: string

  // MCP & 插件
  mcp: {
    clients: MCPServerConnection[]
    tools: Tool[]
    commands: Command[]
    resources: Record<string, ServerResource[]>
    pluginReconnectKey: number
  }
  plugins: {
    enabled: LoadedPlugin[]
    disabled: LoadedPlugin[]
    commands: Command[]
    errors: PluginError[]
    installationStatus: { ... }
    needsRefresh: boolean
  }
  agentDefinitions: AgentDefinitionsResult

  // 会话数据
  fileHistory: FileHistoryState              // 文件操作历史
  attribution: AttributionState              // Claude 贡献归因
  todos: { [agentId: string]: TodoList }     // TODO 列表
  remoteAgentTaskSuggestions: { summary: string; task: string }[]
  notifications: { current: Notification | null; queue: Notification[] }
  elicitation: { queue: ElicitationRequestEvent[] }

  // 功能开关
  thinkingEnabled: boolean | undefined
  promptSuggestionEnabled: boolean
  sessionHooks: SessionHooksState

  // Tmux (Tungsten)
  tungstenActiveSession?: { sessionName: string; socketName: string; target: string }
  tungstenLastCapturedTime?: number
  tungstenLastCommand?: { command: string; timestamp: number }
  tungstenPanelVisible?: boolean
  tungstenPanelAutoHidden?: boolean

  // WebBrowser (Bagel)
  bagelActive?: boolean
  bagelUrl?: string
  bagelPanelVisible?: boolean

  // Computer Use MCP (Chicago)
  computerUseMcpState?: {
    allowedApps?: [...]
    grantFlags?: {...}
    lastScreenshotDims?: {...}
  }

  // REPL VM 上下文
  replContext?: {
    vmContext: vm.Context
    registeredTools: Map<...>
    console: {...}
  }

  // 团队上下文
  teamContext?: { teamName: string; teammates: {...}; ... }
  standaloneAgentContext?: { name: string; color?: AgentColorName }

  // Worker/Sandbox 权限
  inbox: { messages: [...] }
  workerSandboxPermissions: { queue: [...]; selectedIndex: number }
  pendingWorkerRequest: { toolName: string; ... } | null
  pendingSandboxRequest: { requestId: string; host: string } | null

  // 推理增强
  promptSuggestion: {
    text: string | null
    promptId: ...
    shownAt: number
    acceptedAt: number
    generationRequestId: string | null
  }
  speculation: SpeculationState              // 投机执行
  speculationSessionTimeSavedMs: number
  skillImprovement: { suggestion: {...} | null }

  // 其他
  companionReaction?: string                 // Buddy 精灵反应
  companionPetAt?: number
}>
```

### DeepImmutable 设计

状态被分为两类：
- **不可变部分** (`DeepImmutable<{...}>`)：核心配置、UI 状态、Bridge 状态。通过类型系统防止直接修改。
- **可变部分** (`& {...}`)：高频变更的字段（tasks, mcp.clients, plugins, fileHistory）保持可变，避免每次更新都创建深拷贝。

### Store 实现

```typescript
export type AppStateStore = Store<AppState>

// Store<T> 提供:
// - getState(): T
// - setState((prev: T) => T): void
// - useSyncExternalStore 集成
```

状态更新通过 `setAppState` 函数式更新：

```typescript
setAppState((prev) => ({
  ...prev,
  replBridgeConnected: true,
  replBridgeReconnecting: false,
}))
```

### 状态变更回调

```typescript
// src/state/onChangeAppState.ts
// 状态变更时触发的副作用:
// - 会话历史保存
// - 文件历史快照
// - 分析事件发送
```

## 持久化

| 数据 | 存储位置 | 说明 |
|------|---------|------|
| 用户设置 | `~/.claude/settings.json` | 全局配置 |
| 输入历史 | `~/.claude/history.jsonl` | 用户输入行记录 (JSONL 格式, 支持上箭头/Ctrl+R 检索) |
| 会话转录 | `~/.claude/projects/<hash>/<session-id>.jsonl` | 完整消息流、文件历史快照、归因快照 |
| OAuth 令牌 | macOS Keychain | 安全存储 |
| API 密钥 | macOS Keychain / env var | 安全存储 |
| 会话缓存 | 内存 | 当前会话 |
| 团队记忆 | `TEAM_MEMORY_SYNC_URL` | 集中化团队记忆 |

## React + Ink 终端渲染

### 自定义 Ink 渲染器

Claude Code 在标准 Ink 基础上构建了自定义渲染器 (`src/ink/`, 96 文件)：

| 模块 | 说明 |
|------|------|
| `ink.tsx` | 主渲染器入口 |
| `reconciler.ts` | React Reconciler 实现 |
| `components/` | 底层 Ink 组件 (Box, Text, etc.) |
| `hooks/` | Ink 专用 Hooks |
| `layout/` | 终端布局引擎 (类 flexbox) |
| `termio/` | 终端 I/O 抽象 |
| `parse-keypress.ts` | 键盘输入解析 |
| `terminal.ts` | 终端能力检测 |
| `focus.ts` | 焦点管理 |
| `selection.ts` | 文本选择 |

### 增强功能

- **ANSI 颜色/样式**: 完整 ANSI 转义码支持
- **双向文本**: bidi-js 支持 RTL 语言
- **CJK 宽字符**: get-east-asian-width 正确计算字符宽度
- **Emoji 处理**: emoji-regex 检测和宽度计算
- **超链接**: supports-hyperlinks 检测终端超链接支持
- **React 19 编译器**: 使用 React Compiler 优化渲染性能

## 组件架构 (389 文件)

### 渲染层次

```
main.tsx
  |
  v
AppStateProvider (Zustand Store)
  |
  v
REPL.tsx (主布局)
  ├── MessagesList (消息列表渲染)
  ├── PromptInput (用户输入栏)
  ├── Spinner (工具进度)
  ├── Footer (状态栏)
  └── Dialogs (模态框)
```

### 组件分类

| 分类 | 说明 | 示例 |
|------|------|------|
| 基础组件 | Ink 原语 | Box, Text, Button, Link |
| 设计系统 | 主题化包装 | ThemedBox, ThemedText, Color palette |
| 应用组件 | 核心功能 | REPL, PromptInput, Messages, Footer |
| 对话框 | 交互式 | Resume, Config, Permissions, Onboarding |
| Agent 组件 | 多 Agent | AgentProgressLine, CoordinatorAgentStatus |
| Diff 组件 | 代码差异 | DiffView, InlineDiff |

### Context Providers

```typescript
// src/context/
notifications.tsx     // 通知上下文
modalContext.tsx       // 模态框状态
stats.tsx             // 统计信息上下文
overlays.tsx          // 覆盖层
prompts.tsx           // 提示上下文
```

## Hooks 架构 (104 文件)

### 核心 Hooks

| Hook | 说明 |
|------|------|
| `useCanUseTool` | 工具权限检查门控 |
| `useCommandKeybindings` | 命令快捷键 |
| `useExitOnCtrlCD` | Ctrl+C/D 退出处理 |
| `useAfterFirstRender` | 首次渲染后回调 |
| `useDynamicConfig` | 动态配置 |

### 权限 Hooks (`hooks/toolPermission/`)

| Hook | 说明 |
|------|------|
| `PermissionContext.ts` | 权限上下文定义 |
| `coordinatorHandler` | 主线程权限处理 |
| `interactiveHandler` | 交互模式权限处理 |
| `swarmWorkerHandler` | 子 Agent 权限处理 |

### 状态更新流

```
工具执行 -> setAppState((prev) => ({...prev, ...changes}))
    |
    v
AppState 更新 -> onChangeAppState() 回调
    |
    v
副作用:
  - 会话历史保存
  - 文件历史快照
  - 分析事件发送
    |
    v
React 组件 (通过 useSyncExternalStore)
    |
    v
Ink 渲染 -> 终端输出
```
