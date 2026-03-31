# 11 上下文管理与压缩

## 架构图

> 完整 PlantUML 源文件: [diagrams/compaction-strategy.puml](diagrams/compaction-strategy.puml)

## 一、上下文系统

### 系统上下文

```typescript
// src/context.ts

export const getSystemContext = memoize(
  async (): Promise<{ [k: string]: string }> => {
    const gitStatus = await getGitStatus()
    const injection = getSystemPromptInjection()

    return {
      ...(gitStatus && { gitStatus }),
      ...(injection ? { cacheBreaker: `[CACHE_BREAKER: ${injection}]` } : {}),
    }
  },
)
```

系统上下文在会话开始时收集一次，通过 `memoize` 缓存。包含：

- **Git 状态**: 当前分支、主分支、用户名、最近 5 次提交、工作树状态
- **Cache Breaker**: ANT-only 测试注入

### Git 状态收集

```typescript
export const getGitStatus = memoize(
  async (): Promise<string | null> => {
    // 执行: git status --short, git log -n 5, git config user.name
    // 格式化输出包含:
    //   - 当前分支
    //   - 主分支 (用于 PR 引用)
    //   - Git 用户名
    //   - 简短状态
    //   - 最近 5 次提交
    // 超过 2000 字符时截断
  },
)
```

### 用户上下文

```typescript
export const getUserContext = memoize(
  async (): Promise<{ [k: string]: string }> => {
    const claudeMd = getClaudeMds(filterInjectedMemoryFiles(
      await getMemoryFiles()
    ))

    // 缓存供 yoloClassifier 使用 (打破导入环)
    setCachedClaudeMdContent(claudeMd || null)

    return {
      ...(claudeMd && { claudeMd }),
      currentDate: `Today's date is ${getLocalISODate()}.`,
    }
  },
)
```

### CLAUDE.md 文件发现

```typescript
// src/utils/claudemd.ts

export const getMemoryFiles = memoize(
  async (): Promise<MemoryFileInfo[]> => {
    // 按优先级发现和加载:
    // 1. Managed memory (~/managed/CLAUDE.md)
    // 2. User memory (~/.claude/CLAUDE.md)
    // 3. Project memory (CLAUDE.md, .claude/CLAUDE.md, .claude/rules/*.md)
    // 4. Local memory (CLAUDE.local.md)
    //
    // 越靠近 cwd 的文件优先级越高 (后加载)
  },
)

export type MemoryFileInfo = {
  path: string
  type: MemoryType
  content: string
  parent?: string                     // 包含此文件的父文件路径
  globs?: string[]                    // frontmatter 中的 glob 模式
  contentDiffersFromDisk?: boolean    // 自动注入是否修改了内容
  rawContent?: string                 // 原始磁盘字节
}
```

### 文件发现路径

| 类型 | 路径 | 说明 |
|------|------|------|
| 用户记忆 | `~/.claude/CLAUDE.md` | 全局用户指令 |
| 项目记忆 | `CLAUDE.md` | 项目根目录 |
| 项目记忆 | `.claude/CLAUDE.md` | .claude 目录内 |
| 项目规则 | `.claude/rules/*.md` | 规则目录 (按字母排序) |
| 本地记忆 | `CLAUDE.local.md` | gitignored 的个人配置 |
| 管理记忆 | `~/managed/CLAUDE.md` | 企业管理注入 |

项目记忆从 cwd 向根目录遍历，每一级都检查上述路径。

### @include 指令

```
语法: @path, @./relative/path, @~/home/path, @/absolute/path
```

- 在文本叶节点中工作 (不在代码块内)
- 防止循环引用
- 不存在的文件静默忽略
- 支持格式: .md, .txt, .json, .yaml, .py, .js, .ts, .java, .go, .rs 等

### Frontmatter 支持

```yaml
---
globs:
  - "src/**/*.ts"
  - "!src/test/**"
---
```

带 glob 的 CLAUDE.md 文件只在匹配的文件被操作时加载，减少不相关上下文的注入。

### 上下文拼接

```typescript
export const getClaudeMds = (files: MemoryFileInfo[]): string | null
// 将所有记忆文件用分隔符拼接为单个字符串
// 包含来源路径注释，帮助 Claude 理解指令来源
```

## 二、上下文压缩

### 压缩策略

Claude Code 使用三种压缩策略：

| 策略 | 触发条件 | 激进程度 | 说明 |
|------|---------|---------|------|
| Microcompact | 时间/轮次触发 | 低 | 清理旧工具结果内容 |
| Full Compact | Token 接近限制 | 中 | 摘要整个对话历史 |
| Reactive Compact | API 返回 prompt_too_long | 高 | 被动触发的强制压缩 |

### Token 阈值

```typescript
export const AUTOCOMPACT_BUFFER_TOKENS = 13_000
export const WARNING_THRESHOLD_BUFFER_TOKENS = 20_000
export const ERROR_THRESHOLD_BUFFER_TOKENS = 20_000
export const MANUAL_COMPACT_BUFFER_TOKENS = 3_000
```

```typescript
export function calculateTokenWarningState(tokenUsage: number, model: string): {
  percentLeft: number
  isAboveWarningThreshold: boolean       // contextWindow - 20K
  isAboveErrorThreshold: boolean         // contextWindow - 20K
  isAboveAutoCompactThreshold: boolean   // contextWindow - 13K
  isAtBlockingLimit: boolean             // contextWindow - 3K
}
```

### Auto Compact 决策

```typescript
export type AutoCompactTrackingState = {
  compacted: boolean
  turnCounter: number
  turnId: string
  consecutiveFailures?: number
}

export function getAutoCompactThreshold(model: string): number
// effectiveContextWindow - AUTOCOMPACT_BUFFER_TOKENS
// 可通过 CLAUDE_AUTOCOMPACT_PCT_OVERRIDE 环境变量覆盖

export function getEffectiveContextWindowSize(model: string): number
// contextWindow - reservedTokensForSummary (最大 20K)
// 可通过 CLAUDE_CODE_AUTO_COMPACT_WINDOW 环境变量覆盖

export function isAutoCompactEnabled(): boolean
// 尊重: DISABLE_COMPACT, DISABLE_AUTO_COMPACT, settings.autoCompactEnabled

export async function shouldAutoCompact(
  messages: Message[],
  model: string,
  querySource?: QuerySource,
  snipTokensFreed?: number,
): Promise<boolean>
```

**递归防护**:
- 跳过 `session_memory` 和 `compact` querySource (防止死锁)
- 跳过 `marble_origami` 如果启用 CONTEXT_COLLAPSE (防止状态损坏)

**仅反应式模式**:
- `tengu_cobalt_raccoon` Feature Flag 可抑制主动 autocompact
- 依赖 API 的 prompt-too-long 错误触发反应式压缩

### Full Compact

```typescript
export interface CompactionResult {
  boundaryMarker: SystemMessage          // 压缩边界标记
  summaryMessages: UserMessage[]         // 摘要消息
  attachments: AttachmentMessage[]       // 保留的附件
  hookResults: HookResultMessage[]       // Hook 结果
  messagesToKeep?: Message[]             // 保留的近期消息
  userDisplayMessage?: string            // 用户显示消息
  preCompactTokenCount?: number          // 压缩前 Token 数
  postCompactTokenCount?: number         // 压缩后 Token 数
  truePostCompactTokenCount?: number     // 真实压缩后 Token 数
  compactionUsage?: ReturnType<typeof getTokenUsage>
}

export async function compactConversation(
  messages: Message[],
  context: ToolUseContext,
  cacheSafeParams: CacheSafeParams,
  suppressFollowUpQuestions: boolean,
  customInstructions?: string,
  isAutoCompact: boolean = false,
  recompactionInfo?: RecompactionInfo,
): Promise<CompactionResult>
```

Full Compact 步骤：
1. `stripImagesFromMessages()` - 移除图片，替换为 `[image]` 标记
2. 构建 compact prompt
3. 调用 Claude API 生成摘要
4. 创建边界标记
5. 构建摘要消息

### Post-Compact 消息重建

```typescript
export function buildPostCompactMessages(result: CompactionResult): Message[]
// 顺序:
// 1. boundaryMarker (SystemMessage)
// 2. summaryMessages (UserMessage[])
// 3. messagesToKeep
// 4. attachments (AttachmentMessage[])
// 5. hookResults (HookResultMessage[])
```

### Post-Compact 清理

```typescript
export const POST_COMPACT_MAX_FILES_TO_RESTORE = 5
export const POST_COMPACT_TOKEN_BUDGET = 50_000
export const POST_COMPACT_MAX_TOKENS_PER_FILE = 5_000
export const POST_COMPACT_MAX_TOKENS_PER_SKILL = 5_000
export const POST_COMPACT_SKILLS_TOKEN_BUDGET = 25_000
```

清理策略：
- 恢复最多 5 个最常访问的文件 (50K token 预算, 每文件 5K)
- 恢复已调用的技能 (25K token 预算, 每技能 5K)
- 超过预算时截断

### Microcompact

```typescript
export const TIME_BASED_MC_CLEARED_MESSAGE = '[Old tool result content cleared]'

export function evaluateTimeBasedTrigger(): boolean
// 基于时间/轮次决定是否运行 microcompact

export function estimateMessageTokens(messages: Message[]): number
// 快速 Token 估算 (不调用 API)
```

Microcompact 特点：
- 清理旧工具结果的详细内容
- 替换为 `[Old tool result content cleared]`
- 基于时间触发评估
- 轻量级，不需要 API 调用

### 重压缩 (Recompaction)

```typescript
export type RecompactionInfo = {
  isRecompactionInChain: boolean       // 是否在压缩链中
  turnsSincePreviousCompact: number    // 距上次压缩的轮数
  previousCompactTurnId?: string       // 上次压缩的轮 ID
  autoCompactThreshold: number
  querySource?: QuerySource
}
```

当一次压缩后 Token 仍然超过阈值时：
1. 递增 `consecutiveFailures`
2. 如果允许重压缩，带上 `RecompactionInfo` 递归执行
3. 如果不允许，向用户显示警告

## 核心文件

| 文件 | 说明 |
|------|------|
| `context.ts` | 系统/用户上下文收集 |
| `utils/claudemd.ts` (46KB) | CLAUDE.md 发现与加载 |
| `services/compact/compact.ts` | Full Compact 实现 |
| `services/compact/autoCompact.ts` | 自动压缩决策 |
| `services/compact/microCompact.ts` | Microcompact 实现 |
| `services/compact/postCompactCleanup.ts` | 压缩后清理 |
| `services/compact/prompt.ts` | 压缩提示模板 |
| `services/tokenEstimation.ts` | Token 估算 |
