# 03 核心引擎 QueryEngine

## 架构图

> 完整 PlantUML 源文件: [diagrams/query-loop.puml](diagrams/query-loop.puml)

## 概述

QueryEngine 是 Claude Code 的核心执行引擎，负责：
- 管理会话消息历史
- 调用 Claude API（流式）
- 分发工具调用并收集结果
- Token 计数与成本追踪
- 重试与降级逻辑
- 上下文压缩触发

每个会话创建一个 QueryEngine 实例，支持多轮对话。

## QueryEngine 类结构

```typescript
// src/QueryEngine.ts

export type QueryEngineConfig = {
  cwd: string                              // 工作目录
  tools: Tools                             // 可用工具集
  commands: Command[]                      // 命令列表
  mcpClients: MCPServerConnection[]        // MCP 连接
  agents: AgentDefinition[]                // Agent 定义
  canUseTool: CanUseToolFn                 // 权限检查函数
  getAppState: () => AppState              // 状态读取
  setAppState: (f: (prev: AppState) => AppState) => void  // 状态更新
  initialMessages?: Message[]              // 初始消息
  readFileCache: FileStateCache            // 文件状态缓存
  customSystemPrompt?: string              // 自定义系统提示
  appendSystemPrompt?: string              // 追加系统提示
  userSpecifiedModel?: string              // 用户指定模型
  fallbackModel?: string                   // 降级模型
  thinkingConfig?: ThinkingConfig          // 思考模式配置
  maxTurns?: number                        // 最大工具循环轮数
  maxBudgetUsd?: number                    // 预算上限 (USD)
  taskBudget?: { total: number }           // 任务预算
  jsonSchema?: Record<string, unknown>     // 结构化输出 Schema
  verbose?: boolean                        // 详细日志
}

export class QueryEngine {
  private config: QueryEngineConfig
  private mutableMessages: Message[]           // 会话消息 (可变数组)
  private abortController: AbortController     // 取消控制
  private permissionDenials: SDKPermissionDenial[]  // 权限拒绝记录
  private totalUsage: NonNullableUsage         // 累计 Token 用量
  private discoveredSkillNames: Set<string>    // 已发现技能名
  private loadedNestedMemoryPaths: Set<string> // 已加载记忆路径
}
```

## 查询循环 (Query Loop)

QueryEngine 的核心是一个 **async generator 驱动的查询循环**：

```typescript
async *submitMessage(
  prompt: string | ContentBlockParam[],
  options?: { uuid?: string; isMeta?: boolean },
): AsyncGenerator<SDKMessage, void, unknown>
```

### 循环流程

1. **构建 API 请求**
   - 系统提示 (system context + tool schemas + instructions)
   - 用户上下文 (CLAUDE.md, memory files)
   - 消息历史
   - 思考模式配置

2. **调用 Claude API (流式)**
   - 使用 Anthropic SDK 发起流式请求
   - 处理 SSE 事件: `message_start`, `content_block_delta`, `message_delta`, `message_stop`

3. **消息类型分发**
   ```typescript
   for await (const message of query({...})) {
     switch (message.type) {
       case 'assistant':     // 助手响应 + 累计用量
       case 'stream_event':  // 流事件追踪
       case 'attachment':    // 结构化输出、max_turns 信号
       case 'progress':      // 工具进度更新
       case 'user':          // 重放用户消息
       case 'tombstone':     // 控制信号 (跳过)
     }
   }
   ```

4. **工具调用处理**
   - 检测 AssistantMessage 中的 ToolUseBlocks
   - 对每个 ToolUseBlock:
     - `findToolByName()` 查找工具
     - `canUseTool()` 权限检查
     - `tool.call()` 执行工具
     - 收集 ToolResultBlock
   - 将所有 ToolResult 封装为 UserMessage
   - 回到步骤 2，开始下一轮循环

5. **终止条件**
   - 助手响应不含 ToolUseBlocks (纯文本回复)
   - 达到 `maxTurns` 限制
   - 达到预算上限 `maxBudgetUsd`
   - 用户取消 (AbortController)

## 流式事件处理

```typescript
// 流事件追踪
message_start  -> 记录初始 usage (input_tokens, output_tokens)
content_block_delta -> 累积文本/tool_use 内容
message_delta  -> 累积增量 usage
message_stop   -> 完成消息，汇总 totalUsage

// Usage 累积
if (message.event.type === 'message_stop') {
  this.totalUsage = accumulateUsage(
    this.totalUsage,
    currentMessageUsage,
  )
}
```

## 权限包装

QueryEngine 在实例级别包装权限检查，追踪整个查询过程中的所有拒绝：

```typescript
const wrappedCanUseTool: CanUseToolFn = async (
  tool, input, toolUseContext, assistantMessage, toolUseID, forceDecision
) => {
  const result = await canUseTool(...)

  // 追踪拒绝记录用于 SDK 报告
  if (result.behavior !== 'allow') {
    this.permissionDenials.push({
      tool_name: sdkCompatToolName(tool.name),
      tool_use_id: toolUseID,
      tool_input: input,
    })
  }

  return result
}
```

## Token 计数与成本

Token 计数不在 QueryEngine 内部实现，而是从 Claude API 的流式响应事件中获取：

```typescript
type NonNullableUsage = {
  input_tokens: number
  output_tokens: number
  cache_creation_input_tokens: number
  cache_read_input_tokens: number
}
```

成本追踪由 `cost-tracker.ts` 独立模块处理：
- 按模型定价计算费用
- 区分输入/输出/缓存 Token
- 支持 Prompt Caching 折扣

## 重试与降级

```typescript
// 错误处理策略
FallbackTriggeredError -> 切换到 fallbackModel 重试
isPromptTooLongMessage() -> 触发自动 compact
withRetry() -> 指数退避重试可恢复的 API 错误
```

**降级模型**: 当主模型不可用时，QueryEngine 可切换到 `fallbackModel`（通常是较小的模型）继续执行。

**Prompt 过长**: 当 API 返回 prompt_too_long 错误时，触发反应式压缩（reactive compact），压缩后重试。

## 与其他模块的关系

| 模块 | 关系 |
|------|------|
| `query.ts` | 消息处理、标准化、API 调用封装 |
| `Tool.ts` | 工具类型定义、执行上下文 |
| `tools.ts` | 工具注册表，提供 `findToolByName()` |
| `services/api/claude.ts` | Anthropic SDK 封装 |
| `services/compact/` | 上下文压缩（自动触发） |
| `state/AppStateStore.ts` | 全局状态读写 |
| `cost-tracker.ts` | 成本追踪 |

## 消息类型

QueryEngine 处理以下消息类型：

| 类型 | 说明 |
|------|------|
| `UserMessage` | 用户输入 + 附件 |
| `AssistantMessage` | Claude 响应，可能包含 ToolUseBlocks |
| `SystemMessage` | 系统信息/警告 (权限、记忆保存等) |
| `AttachmentMessage` | 文件/图片附件 |
| `ProgressMessage` | 工具执行进度 |
| `ToolUseSummaryMessage` | 压缩后的工具调用摘要 |
| `TombstoneMessage` | 控制信号 (跳过) |

消息标准化由 `utils/messages.ts` 处理，确保不同来源（API、本地命令、Legacy 格式）的消息统一为内部格式。
