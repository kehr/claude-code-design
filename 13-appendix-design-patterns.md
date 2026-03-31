# 13 关键设计模式总结

本章总结 Claude Code 中反复出现的架构模式和设计决策，这些模式贯穿多个子系统。

## 1. 并行预取 (Parallel Prefetch)

**问题**: 启动时需要加载 MDM 设置、Keychain 令牌、模块评估，串行执行耗时过长。

**解决方案**: 在模块导入的 I/O 空闲期并行启动预取：

```typescript
// main.tsx - 在任何 import 之前
profileCheckpoint('main_tsx_entry')
startMdmRawRead()          // 并行: MDM 子进程
startKeychainPrefetch()    // 并行: Keychain 读取
// ... 此处 ~135ms 的模块导入评估 ...
profileCheckpoint('main_tsx_imports_loaded')
```

**应用场景**:
- 启动时 MDM + Keychain (Phase 1)
- 首次渲染后 `startDeferredPrefetches()` (Phase 7)
- `loadAllCommands()` 中并行加载 Skills、Plugins、Workflows

## 2. Async Generator 驱动的查询循环

**问题**: 传统 async/await 需要等待完整响应才能处理，无法实现流式 UI 更新。

**解决方案**: 使用 async generator 实现增量流式处理：

```typescript
async *submitMessage(prompt): AsyncGenerator<SDKMessage, void, unknown> {
  for await (const message of query({...})) {
    switch (message.type) {
      case 'assistant': yield message; break;
      case 'stream_event': /* 实时 UI 更新 */ break;
      // ...
    }
  }
}
```

**优势**:
- 调用方逐消息处理，不等待完整响应
- 自然支持工具循环（yield 后继续）
- 与 React 的 setState 无缝集成（每个 yield 触发渲染）

## 3. Feature Flag 死代码消除 (DCE)

**问题**: 内部版本和外部版本的功能集不同，需要在构建时排除代码。

**解决方案**: 使用 `bun:bundle` 的 `feature()` 函数进行编译期条件编译：

```typescript
import { feature } from 'bun:bundle'

const coordinatorModeModule = feature('COORDINATOR_MODE')
  ? require('./coordinator/coordinatorMode.js')
  : null
```

**优势**:
- Bun 打包器将 `feature()` 替换为字面量 `true`/`false`
- 死代码消除移除不可达分支的全部代码
- 无运行时开销
- 减少包体积和攻击面

## 4. Memoize 模式

**问题**: 初始化函数和上下文收集需要确保只执行一次。

**解决方案**: 广泛使用 `memoize` 缓存函数结果：

```typescript
export const init = memoize(async (): Promise<void> => { ... })
export const getSystemContext = memoize(async () => { ... })
export const getUserContext = memoize(async () => { ... })
export const getMemoryFiles = memoize(async () => { ... })
const COMMANDS = memoize((): Command[] => [...])
const loadAllCommands = memoize(async (cwd) => { ... })
```

**特点**: 支持 async 函数，首次调用执行并缓存 Promise，后续调用直接返回同一 Promise。

## 5. 依赖注入 (Dependency Injection)

**问题**: 模块间直接导入导致循环依赖和测试困难。

**解决方案**: 通过回调函数和上下文对象注入依赖：

```typescript
// Bridge 不导入命令注册表，而是接收回调
export type BridgeCoreParams = {
  getAccessToken: () => string | undefined
  createSession: (opts: {...}) => Promise<string | null>
  archiveSession: (sessionId: string) => Promise<void>
  // ...
}

// 工具通过 ToolUseContext 访问服务
export type ToolUseContext = {
  getAppState(): AppState
  setAppState(f: (prev: AppState) => AppState): void
  handleElicitation?: (serverName, params, signal) => Promise<ElicitResult>
  // ...
}
```

**应用场景**:
- Bridge 核心 (`BridgeCoreParams`)
- 工具执行 (`ToolUseContext`)
- MCP 认证处理 (闭包回调)

## 6. 判别联合类型 (Discriminated Unions)

**问题**: 不同状态的对象需要不同的字段，但 TypeScript 需要类型安全的区分。

**解决方案**: 使用 `type` 字段作为判别器：

```typescript
// MCP 连接状态
export type MCPServerConnection =
  | ConnectedMCPServer      // type: 'connected'
  | FailedMCPServer         // type: 'failed'
  | NeedsAuthMCPServer      // type: 'needs-auth'
  | PendingMCPServer        // type: 'pending'
  | DisabledMCPServer       // type: 'disabled'

// 消息类型
type Message =
  | UserMessage             // type: 'user'
  | AssistantMessage        // type: 'assistant'
  | SystemMessage           // type: 'system'
  | ProgressMessage         // type: 'progress'
  | AttachmentMessage       // type: 'attachment'
  | TombstoneMessage        // type: 'tombstone'
  | ToolUseSummaryMessage   // type: 'tool_use_summary'

// 权限决策
type PermissionDecision =
  | PermissionAllowDecision   // behavior: 'allow'
  | PermissionAskDecision     // behavior: 'ask'
  | PermissionDenyDecision    // behavior: 'deny'
```

**优势**: TypeScript 编译器在 switch/if 分支中自动收窄类型，消除不安全的类型断言。

## 7. Fail-Closed 安全默认值

**问题**: 工具开发者可能遗漏安全声明，导致危险操作不被权限系统拦截。

**解决方案**: `buildTool()` 提供保守默认值：

```typescript
const TOOL_DEFAULTS = {
  isConcurrencySafe: () => false,    // 默认不并发安全
  isReadOnly: () => false,           // 默认假定有写操作
  isDestructive: () => false,
  checkPermissions: (input) =>
    Promise.resolve({ behavior: 'allow', updatedInput: input }),
}
```

**效果**: 遗漏声明只会导致保守行为（多余的权限提示或串行执行），不会导致安全漏洞。

## 8. 权限冒泡 (Permission Bubbling)

**问题**: 多个子 Agent 同时向用户提示权限会造成混乱。

**解决方案**: 子 Agent 的权限请求冒泡到父 Agent：

```typescript
const FORK_AGENT = {
  permissionMode: 'bubble',  // 权限冒泡到父级
  // ...
}
```

权限冒泡链：`Worker -> Coordinator -> User`

## 9. 传输抽象层

**问题**: Bridge 协议从 v1(WebSocket) 演进到 v2(SSE)，上层代码不应感知差异。

**解决方案**: `ReplBridgeTransport` 接口抽象传输细节：

```typescript
export type ReplBridgeTransport = {
  write(message: StdoutMessage): Promise<void>
  writeBatch(messages: StdoutMessage[]): Promise<void>
  connect(): void
  close(): void
  isConnectedStatus(): boolean
  getLastSequenceNum(): number
  // ...
}
```

| 方法 | v1 (HybridTransport) | v2 (SSETransport + CCRClient) |
|------|----------------------|-------------------------------|
| 读取 | WebSocket | SSE |
| 写入 | HTTP POST | CCR v2 API |
| 重放 | N/A | SSE 序号 |
| 投递确认 | N/A | `reportDelivery()` |

## 10. 循环依赖处理

**问题**: 大型 TypeScript 项目中模块间容易形成循环导入。

**解决方案组合**:

| 策略 | 示例 | 说明 |
|------|------|------|
| 懒加载 require | `() => require('./path.js')` | 延迟到运行时解析 |
| 类型分离 | `types/` 独立目录 | 类型定义不产生运行时导入 |
| 模块级缓存 | `bootstrap/state.ts` | 作为中间缓存打破环 |
| Getter 函数 | `getTeammateUtils()` | 避免模块级直接导入 |
| Re-exports | `Tool.ts` re-exports permission types | 集中重导出 |
| Lazy Schema | `lazySchema(() => z.object({...}))` | Zod Schema 延迟初始化 |

## 11. DeepImmutable + 可变混合

**问题**: 全部不可变导致高频更新字段的性能问题；全部可变导致意外突变。

**解决方案**: AppState 混合使用不可变和可变：

```typescript
export type AppState = DeepImmutable<{
  // 核心配置 - 不可变 (类型系统保护)
  settings: SettingsJson
  toolPermissionContext: ToolPermissionContext
  // ...
}> & {
  // 高频变更 - 可变 (性能优先)
  tasks: { [taskId: string]: TaskState }
  mcp: { clients: MCPServerConnection[] }
  plugins: { enabled: LoadedPlugin[] }
  // ...
}
```

## 12. 消息标准化

**问题**: 消息来自多个来源（API、本地命令、Legacy 格式），格式不统一。

**解决方案**: `utils/messages.ts` 统一消息格式：

```typescript
normalizeMessage(msg) -> SDKMessage
// 支持:
// - Anthropic API 格式
// - 本地命令格式
// - Legacy 格式
// - SDK 格式
```

## 13. Token 预算管理

**问题**: 上下文窗口有限，需要精确管理 Token 分配。

**解决方案**: 多级阈值系统：

| 阈值 | 距离限制 | 行为 |
|------|---------|------|
| Warning | 20K tokens | 显示警告 |
| Error | 20K tokens | 显示错误 |
| Auto-compact | 13K tokens | 自动压缩 |
| Blocking | 3K tokens | 停止，必须手动压缩 |

Post-compact 也有 Token 预算：
- 文件恢复: 50K 总预算, 5K/文件, 最多 5 文件
- Skill 恢复: 25K 总预算, 5K/skill

## 14. 消息去重

**问题**: Bridge 传输层切换和网络抖动可能导致消息重复。

**解决方案**: `BoundedUUIDSet` 有界集合：

- **回显去重**: 追踪发送 UUID，过滤服务端回传
- **入站去重**: 追踪接收 UUID，过滤重投递
- **有界**: 集合大小有限，自动淘汰旧条目

## 15. Prompt Cache 优化

**问题**: Fork 子 Agent 的系统提示高度相似，重复发送浪费 Token。

**解决方案**: 构造 cache-identical 前缀：

```typescript
export function buildForkedMessages(directive, assistantMessage): MessageType[] {
  // 所有 fork 子 Agent 共享:
  // - 父 Agent 渲染后的系统提示字节
  // - 工具定义
  // - 消息历史
  // 只有最后的 directive 不同
  // -> 最大化 Prompt Cache 前缀复用
}
```

## 16. 补充子系统说明

以下子系统规模较小，不单独成文，但对理解完整架构不可或缺。

### 认证系统

认证系统 (`utils/auth.ts`, 65KB) 支持多种认证路径：

| 认证方式 | 说明 | 适用场景 |
|---------|------|---------|
| OAuth 2.0 + PKCE | claude.ai 账户认证 | 默认消费者路径 |
| API Key | `ANTHROPIC_API_KEY` 环境变量 | 直接 API 访问 |
| AWS Bedrock | AWS STS 凭证交换 | 企业 AWS 部署 |
| GCP Vertex AI | Google Auth Library | 企业 GCP 部署 |
| xaaIdp | 第三方身份提供者 (XAA) | 企业 SSO |
| JWT | Bridge 会话认证 | IDE/远程模式 |
| macOS Keychain | 安全令牌存储 | 本地持久化 |
| apiKeyHelper | 外部脚本获取密钥 | 自定义凭证管理 |

**核心文件**:
- `utils/auth.ts` (65KB) - 认证逻辑、订阅检查、AWS/GCP 集成
- `services/oauth/oauthFlow.ts` - OAuth 2.0 PKCE 流程
- `services/oauth/sessionStorage.ts` - OAuth 令牌持久化
- `bridge/jwtUtils.ts` - JWT 认证
- `bridge/trustedDevice.ts` - Trusted Device Token (90 天令牌, Keychain 持久化, Bridge 提权认证)
- `utils/secureStorage/` - 安全存储抽象 (macOS Keychain + fallback plaintext)

### 遥测与分析系统

遥测系统采用多层架构，启动时延迟加载以减少启动开销：

| 组件 | 说明 |
|------|------|
| GrowthBook | Feature Flags、A/B 实验、远程评估 |
| OpenTelemetry | 分布式追踪 (traces, metrics, logs) |
| First-party Event Logger | Anthropic 自有事件日志 |
| BigQuery Exporter | 事件导出到 BigQuery |
| Perfetto Tracing | 性能追踪 (Chrome DevTools 格式) |

**GrowthBook 集成**:
- Payload-based Feature Definitions
- 远程评估 (Remote Eval)
- 用户属性 (`GrowthBookUserAttributes`) 用于定向投放
- 通过 `initializeAnalyticsGates()` 延迟初始化

**OpenTelemetry 配置**:
- OTLP gRPC 端点 (`ANT_OTEL_EXPORTER_OTLP_ENDPOINT`)
- 自定义指标端点 (`ANT_CLAUDE_CODE_METRICS_ENDPOINT`)
- 懒加载 (~400KB 的 OTel 包按需导入)

**核心文件**:
- `services/analytics/growthbook.ts` - Feature Flags
- `services/analytics/index.ts` - 事件日志入口
- `services/analytics/sink.ts` - 分析管道
- `utils/telemetry/` - BigQuery、Perfetto、First-party 导出器

### 会话持久化与转录

系统存在三套独立的持久化机制：

| 系统 | 存储位置 | 格式 | 用途 |
|------|---------|------|------|
| 输入历史 | `~/.claude/history.jsonl` | JSONL (逐行追加) | 用户输入行记录, 上箭头/Ctrl+R 检索 |
| 会话转录 | `~/.claude/projects/<hash>/<session-id>.jsonl` | JSONL (结构化条目) | 完整消息流、文件历史、归因快照 |
| 远程会话历史 | Anthropic API `/v1/sessions/{id}/events` | JSON (分页, cursor-based) | claude.ai/Bridge 会话回放 |

**输入历史** (`history.ts`, 14KB):
- 存储用户的 prompt 输入行
- 支持 paste store 的 hash 引用机制
- 用于终端的 history 导航

**会话转录** (`utils/sessionStorage.ts`):
- 按项目 hash 和 session ID 组织
- 包含完整消息流、工具调用结果
- 文件历史快照 (`FileHistoryState`)
- 归因快照 (`AttributionState`) - 追踪 Claude 对代码的贡献百分比

### 文件历史与归因

**File History** (`fileHistory` in AppState):
- 基于 overlay 的文件状态追踪
- 工具执行前后自动快照
- 支持投机执行 (speculation) 的回滚

**Attribution** (`attribution` in AppState):
- 追踪 Claude 生成/修改的代码占比
- 用于 commit message 和 PR 描述中的归因标注
- `settings.attribution.commit` 和 `settings.attribution.pr` 控制格式

### 投机执行系统 (Speculation)

投机执行预测 Claude 的下一步操作，在用户确认前在 overlay 文件系统上执行：

- `speculation` (AppState 字段) - 当前投机执行状态
- `speculationSessionTimeSavedMs` - 累计节省的时间
- 最多 20 轮投机工具调用
- 用户确认后将 overlay 提升为真实文件操作

### Vim 模式

Vim 模式 (`src/vim/`, 5 文件) 实现了终端输入的 Vi 键绑定：

- **双模式状态机**: INSERT 模式 (正常输入) 和 NORMAL 模式 (命令导航)
- **CommandState 子状态机**: 处理 operator-pending (如 `d`, `c`) + motion (如 `w`, `e`, `$`) 组合
- **Dot-repeat**: `RecordedChange` 机制记录最近的编辑操作，`.` 键重放
- 通过 `/vim` 命令切换

### 语音模式

语音模式 (`src/voice/`, 1 文件) 受两个 GrowthBook kill-switch 门控：

- 仅 Anthropic OAuth 用户可用 (不支持 API key/Bedrock/Vertex)
- 需要 `VOICE_MODE` Feature Flag
- 语音输入处理由 `services/voice.ts` (17KB) 实现

### Buddy 精灵系统

Companion 精灵 (`src/buddy/`, 6 文件) 是一个趣味性功能：

- 基于 `userId` 的 seeded PRNG 生成确定性骨骼 (`CompanionBones`)
- LLM 生成个性化的 `CompanionSoul`
- Rarity 系统: common / uncommon / rare / epic / legendary
- `companionReaction` 和 `companionPetAt` (AppState 字段) 追踪交互状态
- 不影响核心功能

### 配置迁移系统

`src/migrations/` (11 文件) 在启动时自动执行配置迁移：

| 迁移 | 说明 |
|------|------|
| `migrateOpusToOpus1m` | Opus 模型重命名 |
| `migrateSonnet45ToSonnet46` | Sonnet 模型升级 |
| `migrateReplBridgeEnabledToRemoteControlAtStartup` | Bridge 配置字段重命名 |
| `migrateBypassPermissionsAcceptedToSettings` | 权限模式迁移到 settings |
| `migrateEnableAllProjectMcpServersToSettings` | MCP 配置迁移 |
| `resetAutoModeOptInForDefaultOffer` | Auto 模式重置 |
| 其他 | 权限规则格式、默认值重置等 |

## 核心文件索引

| 文件 | 行数/大小 | 核心职责 | 关键接口 |
|------|----------|---------|---------|
| `main.tsx` | 4,683 行 | CLI 入口, 启动编排 | `run()`, `renderAndRun()` |
| `QueryEngine.ts` | 46KB | LLM 查询引擎 | `QueryEngine`, `submitMessage()` |
| `Tool.ts` | 29KB | 工具类型系统 | `Tool<>`, `ToolUseContext`, `ToolResult` |
| `commands.ts` | 25KB | 命令注册表 | `getCommands()`, `loadAllCommands()` |
| `query.ts` | 1,729 行 | 消息处理 | `query()`, `processUserInput()` |
| `context.ts` | 6.4KB | 上下文收集 | `getSystemContext()`, `getUserContext()` |
| `state/AppState.tsx` | 23KB | 全局状态 | `AppState`, `AppStateStore` |
| `state/AppStateStore.ts` | - | Store 实现 | `Store<AppState>` |
| `bridge/bridgeMain.ts` | 115KB | Bridge 主循环 | `BridgeCoreParams`, `WorkResponse` |
| `bridge/replBridge.ts` | 100KB | REPL 桥接 | `ReplBridgeHandle`, `ReplBridgeTransport` |
| `utils/auth.ts` | 65KB | 认证 | OAuth 2.0, JWT, AWS/GCP |
| `utils/config.ts` | 63KB | 配置管理 | `getGlobalConfig()` |
| `utils/claudemd.ts` | 46KB | CLAUDE.md 发现 | `getMemoryFiles()`, `MemoryFileInfo` |
| `utils/attachments.ts` | 127KB | 附件处理 | 文件/图片附件 |
| `utils/ansiToPng.ts` | 214KB | ANSI 渲染 | ANSI 转 PNG |
| `commands/insights.ts` | 115KB | 诊断逻辑 | 复杂诊断 |
| `cost-tracker.ts` | 10KB | 成本追踪 | Token 费用计算 |
| `history.ts` | 14KB | 会话历史 | 会话持久化 |
| `services/mcp/client.ts` | - | MCP 客户端 | `connectToServer()`, `fetchToolsForClient()` |
| `services/compact/compact.ts` | - | Full Compact | `compactConversation()`, `CompactionResult` |
| `services/compact/autoCompact.ts` | - | Auto Compact | `shouldAutoCompact()`, 阈值常量 |
| `utils/permissions/permissions.ts` | - | 权限规则 | `getAllowRules()`, `getDenyRules()` |
| `utils/permissions/yoloClassifier.ts` | - | Auto 分类器 | `YoloClassifierResult`, `AutoModeRules` |
| `hooks/toolPermission/PermissionContext.ts` | - | 权限上下文 | `PermissionContext`, handlers |
| `coordinator/coordinatorMode.ts` | - | 协调器模式 | `getCoordinatorSystemPrompt()` |
| `tools/AgentTool/` | 17 文件 | Agent 工具 | `resolveAgentTools()`, `FORK_AGENT` |
| `ink/` | 96 文件 | 终端渲染 | React Reconciler, Layout |
