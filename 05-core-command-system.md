# 05 命令系统

## 概述

命令系统管理 50+ 个斜杠命令（如 `/commit`、`/review`、`/compact`）。命令从四个来源加载，按优先级合并去重，并根据用户身份和功能开关过滤可用性。

## Command 类型定义

```typescript
// src/types/command.ts

export type Command = {
  name: string
  description: string
  helpText?: string
  handler: (input: string, context: CommandContext) => Promise<void>
  aliases?: string[]
  usage?: string
  type?: 'local-jsx' | 'prompt' | 'local'
  availability?: ('claude-ai' | 'console')[]
}
```

## 命令注册机制

### 内置命令数组

```typescript
// src/commands.ts

const COMMANDS = memoize((): Command[] => [
  // 核心命令
  addDir, advisor, agents, branch, btw, chrome, clear, color, compact,
  config, copy, desktop, context, contextNonInteractive, cost, diff,
  doctor, effort, exit, fast, files, heapDump, help, ide, init,
  keybindings, installGitHubApp, installSlackApp, mcp, memory, mobile,
  model, outputStyle, remoteEnv, plugin, pr_comments, releaseNotes,
  reloadPlugins, rename, resume, session, skills, stats, status,
  statusline, stickers, tag, theme, feedback, review, ultrareview,
  rewind, securityReview, terminalSetup, upgrade, extraUsage,
  extraUsageNonInteractive, rateLimitOptions, usage, usageReport, vim,

  // Feature-gated 命令
  ...(webCmd ? [webCmd] : []),
  ...(forkCmd ? [forkCmd] : []),
  ...(buddy ? [buddy] : []),
  ...(proactive ? [proactive] : []),
  ...(briefCommand ? [briefCommand] : []),
  ...(assistantCommand ? [assistantCommand] : []),
  ...(bridge ? [bridge] : []),
  ...(remoteControlServerCommand ? [remoteControlServerCommand] : []),
  ...(voiceCommand ? [voiceCommand] : []),

  // 更多命令
  thinkback, thinkbackPlay, permissions, plan, privacySettings, hooks,
  exportCommand, sandboxToggle,
  ...(!isUsing3PServices() ? [logout, login()] : []),
  passes, tasks,
  ...(workflowsCmd ? [workflowsCmd] : []),
  ...(torch ? [torch] : []),

  // ANT-only 内部命令
  ...(process.env.USER_TYPE === 'ant' && !process.env.IS_DEMO
    ? INTERNAL_ONLY_COMMANDS : []),
])
```

### 多来源加载管道

```typescript
const loadAllCommands = memoize(async (cwd: string): Promise<Command[]> => {
  const [
    { skillDirCommands, pluginSkills, bundledSkills, builtinPluginSkills },
    pluginCommands,
    workflowCommands,
  ] = await Promise.all([
    getSkills(cwd),                  // 从 /skills 目录加载
    getPluginCommands(),             // 已安装插件
    getWorkflowCommands(cwd),        // 工作流脚本
  ])

  // 优先级: bundled > builtinPlugin > skillDir > workflow > plugin > builtin
  return [
    ...bundledSkills,
    ...builtinPluginSkills,
    ...skillDirCommands,
    ...workflowCommands,
    ...pluginCommands,
    ...pluginSkills,
    ...COMMANDS(),
  ]
})
```

**优先级设计**: 先注册的命令优先。同名命令不会被后续来源覆盖。这确保用户自定义的 Skill 可以覆盖内置命令行为。

### 动态命令过滤

```typescript
export async function getCommands(cwd: string): Promise<Command[]> {
  const allCommands = await loadAllCommands(cwd)
  const dynamicSkills = getDynamicSkills()

  // 每次调用重新检查可用性和启用状态
  const baseCommands = allCommands.filter(
    _ => meetsAvailabilityRequirement(_) && isCommandEnabled(_)
  )

  // 去重并插入动态 Skills
  const uniqueDynamicSkills = dynamicSkills.filter(
    s => !baseCommandNames.has(s.name) &&
         meetsAvailabilityRequirement(s) &&
         isCommandEnabled(s)
  )

  return [...baseCommands.slice(0, insertIndex), ...uniqueDynamicSkills, ...]
}
```

## 可用性过滤

```typescript
export function meetsAvailabilityRequirement(cmd: Command): boolean {
  if (!cmd.availability) return true  // 无限制 = 所有人可用

  for (const a of cmd.availability) {
    switch (a) {
      case 'claude-ai':
        if (isClaudeAISubscriber()) return true
        break
      case 'console':
        // 直接 1P API 客户 (非 3P, 非 claude.ai)
        if (!isClaudeAISubscriber() && !isUsing3PServices() && isFirstPartyAnthropicBaseUrl())
          return true
        break
    }
  }
  return false
}
```

## 命令分类

### 工作流命令

| 命令 | 说明 |
|------|------|
| `/commit` | Git 提交 |
| `/review` | 代码审查 |
| `/ultrareview` | 深度代码审查 |
| `/security-review` | 安全审查 |
| `/diff` | 查看文件差异 |
| `/branch` | Git 分支操作 |
| `/pr_comments` | PR 评论查看 |
| `/compact` | 上下文压缩 |
| `/rewind` | 回退操作 |

### 记忆与上下文

| 命令 | 说明 |
|------|------|
| `/memory` | 持久化记忆管理 |
| `/add-dir` | 添加工作目录 |
| `/context` | 查看当前上下文 |
| `/ctx_viz` | 上下文可视化 |

### 配置管理

| 命令 | 说明 |
|------|------|
| `/config` | 设置管理 |
| `/theme` | 主题选择 |
| `/keybindings` | 键绑定配置 |
| `/model` | 模型选择 |
| `/effort` | 努力级别 |
| `/fast` | 快速模式切换 |
| `/permissions` | 权限管理 |
| `/hooks` | Hooks 配置 |
| `/statusline` | 状态栏配置 |
| `/outputStyle` | 输出风格 |

### 开发工具

| 命令 | 说明 |
|------|------|
| `/ide` | IDE 集成 |
| `/desktop` | 桌面应用切换 |
| `/mobile` | 移动端切换 |
| `/install-github-app` | GitHub 集成 |
| `/install-slack-app` | Slack 集成 |

### 会话管理

| 命令 | 说明 |
|------|------|
| `/resume` | 恢复历史会话 |
| `/session` | 会话管理 |
| `/share` | 分享会话 |
| `/login` | 登录 |
| `/logout` | 登出 |
| `/export` | 导出会话 |

### 分析与诊断

| 命令 | 说明 |
|------|------|
| `/doctor` | 环境诊断 |
| `/cost` | Token 使用成本 |
| `/status` | 会话状态 |
| `/usage` | 使用统计 |
| `/stats` | 详细统计 |
| `/feedback` | 用户反馈 |

### 扩展管理

| 命令 | 说明 |
|------|------|
| `/skills` | 技能管理 |
| `/plugin` | 插件管理 |
| `/mcp` | MCP 服务器管理 |
| `/tasks` | 任务管理 |
| `/plan` | 计划模式 |

### Feature-Gated 命令

| 命令 | Feature Flag | 说明 |
|------|-------------|------|
| `/proactive` | PROACTIVE, KAIROS | 主动模式 |
| `/brief` | KAIROS, KAIROS_BRIEF | 简报 |
| `/assistant` | KAIROS | 助手模式 |
| `/bridge` | BRIDGE_MODE | Bridge 模式 |
| `/remote-control-server` | DAEMON, BRIDGE_MODE | 远程控制服务器 |
| `/voice` | VOICE_MODE | 语音输入 |
| `/workflows` | WORKFLOW_SCRIPTS | 工作流脚本 |
| `/remote-setup` | CCR_REMOTE_SETUP | 远程设置 |

## 远程安全命令过滤

在远程/Bridge 模式下，只有部分命令被允许执行：

```typescript
// 远程模式安全命令
export const REMOTE_SAFE_COMMANDS: Set<Command> = new Set([
  session, exit, clear, help, theme, color, vim, cost, usage, copy,
  btw, feedback, plan, keybindings, statusline, stickers, mobile,
])

// Bridge 模式安全命令
export const BRIDGE_SAFE_COMMANDS: Set<Command> = new Set([
  compact, clear, cost, summary, releaseNotes, files,
])

export function filterCommandsForRemoteMode(commands: Command[]): Command[] {
  return commands.filter(cmd => REMOTE_SAFE_COMMANDS.has(cmd))
}

export function isBridgeSafeCommand(cmd: Command): boolean {
  if (cmd.type === 'local-jsx') return false   // Ink UI 不允许通过 Bridge
  if (cmd.type === 'prompt') return true        // Skills 安全
  return BRIDGE_SAFE_COMMANDS.has(cmd)
}
```

**设计决策**: `local-jsx` 类型的命令（需要 Ink 渲染交互式 UI 的命令）在 Bridge 模式下被禁止，因为 Bridge 传输层不支持终端 UI。`prompt` 类型的命令（Skills）默认安全，因为它们只生成文本提示。

## 命令执行流程

1. 用户输入 `/command [args]`
2. `getCommands()` 获取可用命令列表
3. 按名称和别名匹配命令
4. 检查可用性 (`meetsAvailabilityRequirement`)
5. 检查启用状态 (`isCommandEnabled`)
6. 调用 `command.handler(args, context)`
7. 更新 AppState 和 History
