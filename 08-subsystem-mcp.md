# 08 MCP 协议集成

## 概述

MCP (Model Context Protocol) 是 Claude Code 的扩展协议层，允许连接外部服务器以获取工具、资源和提示。MCP 工具被封装为标准 `Tool` 接口，与内置工具无缝集成。

## MCP 连接类型

```typescript
export type MCPServerConnection =
  | ConnectedMCPServer
  | FailedMCPServer
  | NeedsAuthMCPServer
  | PendingMCPServer
  | DisabledMCPServer

export type ConnectedMCPServer = {
  client: Client                        // @modelcontextprotocol/sdk Client
  name: string
  type: 'connected'
  capabilities: ServerCapabilities      // tools, resources, prompts
  serverInfo?: { name: string; version: string }
  instructions?: string
  config: ScopedMcpServerConfig
  cleanup: () => Promise<void>
}
```

使用**判别联合类型** (`type` 字段) 区分连接状态，每种状态携带不同的元数据。

## 传输类型

MCP 支持 6 种传输方式：

| 传输 | Schema | 说明 |
|------|--------|------|
| `stdio` | `McpStdioServerConfigSchema` | 标准 I/O (子进程) |
| `sse` | `McpSSEServerConfigSchema` | Server-Sent Events |
| `sse-ide` | - | IDE 专用 SSE |
| `http` | `McpHTTPServerConfigSchema` | HTTP 流式 |
| `ws` | `McpWebSocketServerConfigSchema` | WebSocket |
| `sdk` | `McpSdkServerConfigSchema` | 进程内 SDK |

### Stdio 配置

```typescript
export const McpStdioServerConfigSchema = z.object({
  command: z.string(),
  args: z.array(z.string()).optional(),
  env: z.record(z.string(), z.string()).optional(),
})
```

### HTTP/SSE 配置

```typescript
export const McpSSEServerConfigSchema = z.object({
  url: z.string(),
  headers: z.record(z.string(), z.string()).optional(),
  oauth: z.object({...}).optional(),
})

export const McpHTTPServerConfigSchema = z.object({
  url: z.string(),
  headers: z.record(z.string(), z.string()).optional(),
  oauth: z.object({...}).optional(),
})
```

### Claude.ai 代理

```typescript
export const McpClaudeAIProxyServerConfigSchema = z.object({
  url: z.string(),
  id: z.string(),
})
```

## 工具发现与封装

### 连接流程

```typescript
export async function connectToServer(
  name: string,
  config: ScopedMcpServerConfig,
): Promise<MCPServerConnection>
```

### 工具封装

```typescript
export async function fetchToolsForClient(
  client: ConnectedMCPServer,
): Promise<Tools>
```

MCP 服务器返回的工具被封装为标准 `Tool` 对象：

- **名称标准化**: `normalizeNameForMCP()` 确保名称符合 Claude 工具命名规范
- **描述截断**: `MAX_MCP_DESCRIPTION_LENGTH = 2048` 字符上限
- **标记**: `isMcp: true` 区分 MCP 工具与内置工具
- **错误处理**:
  - Elicitation 错误 (-32042): 触发凭证请求处理
  - Auth 错误 (UnauthorizedError): 触发 OAuth 刷新

### MCP 工具的 Tool 封装

MCP 工具通过 `buildTool()` 构建，其 `isMcp` 字段设为 `true`：

```typescript
// MCPTool 通过 buildTool({...}) 构建, 类型为 BuiltTool<...>
const mcpTool = buildTool({
  name: normalizeNameForMCP(serverName, toolName),
  isMcp: true,
  mcpInfo: { serverName, toolName },

  call(args, context, ...): Promise<ToolResult<MCPToolResult>> {
    // 调用 MCP 服务器执行工具
  },
  // ...
})

// MCPToolResult 类型:
// { type: 'text' | 'image' | 'resource' | 'error', content: ... }
```

### 结果处理

- **截断**: `mcpContentNeedsTruncation()` 检查结果是否超过限制
- **OAuth 刷新**: `wrapFetchWithStepUpDetection()` 在 401 时自动刷新令牌
- **Elicitation**: `handleElicitation()` 回调处理凭证请求

## 进程内传输

```typescript
export function createLinkedTransportPair(): [Transport, Transport]
```

用于内置/进程内 MCP 服务器，无需启动子进程：
- 双向 `queueMicrotask` 投递
- 零进程开销
- 用于 bundled MCP 服务器

### SDK 控制传输

```typescript
type SdkControlClientTransport implements Transport
```

Agent SDK 通过 control_request/control_response 进行通信：
- JSON-RPC over SDK 结构化 I/O
- 无子进程

## 资源管理

MCP 服务器可以提供资源（文件、数据）供 Claude 访问：

```typescript
// 资源列表
client.listResources() -> ServerResource[]

// 资源读取
client.readResource(uri) -> ResourceContent

// 资源过滤
resources: Record<string, ServerResource[]>  // 按服务器名分组
```

## 官方注册表

```typescript
// src/services/mcp/officialRegistry.ts
// Anthropic 维护的官方 MCP 服务器注册表
// - computer-use: 计算机控制
// - claude-for-chrome: Chrome 扩展
// - 第三方市场发现
```

## 核心文件

| 文件 | 说明 |
|------|------|
| `services/mcp/client.ts` | MCP 服务器连接管理 |
| `services/mcp/config.ts` | MCP 配置解析 |
| `services/mcp/types.ts` | 类型定义 |
| `services/mcp/officialRegistry.ts` | 官方服务器注册表 |
| `services/mcp/xaaIdpLogin.ts` | XAA 身份提供者 |
| `services/mcp/utils.ts` | 工具函数 |
| `tools/MCPTool/` | MCP 工具实现 |
