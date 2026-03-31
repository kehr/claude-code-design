# 12 Skills 与 Plugins

## 概述

Claude Code 通过 Skills 和 Plugins 两种机制实现扩展：
- **Skills**: 轻量级，基于 Markdown 文件，提供专业化的提示和工作流
- **Plugins**: 重量级，带有 manifest、依赖管理、MCP 服务器配置

两者最终都被转化为 `Command` 对象，与内置命令统一调度。

## 一、Skills 系统

### Skill 来源

```typescript
export type LoadedFrom =
  | 'commands_DEPRECATED'    // 旧版命令格式
  | 'skills'                 // 技能目录
  | 'plugin'                 // 插件提供
  | 'managed'                // 管理策略注入
  | 'bundled'                // 内置捆绑
  | 'mcp'                    // MCP 协议
```

### Skill 路径解析

```typescript
export function getSkillsPath(
  source: SettingSource | 'plugin',
  dir: 'skills' | 'commands',
): string {
  // 路由到:
  // - ~/.claude/skills (用户级)
  // - ./.claude/skills (项目级)
  // - managed path (企业管理)
  // - plugin dir (插件提供)
}
```

### Skill 加载流程

Skills 本质上是带有 frontmatter 的 Markdown 文件：

```yaml
---
name: my-skill
description: A custom workflow for X
whenToUse: When the user asks to do X
effort: medium
---

Skill content here...
Instructions for Claude...
```

### Skill 加载管道

```typescript
// src/commands.ts

const loadAllCommands = memoize(async (cwd: string): Promise<Command[]> => {
  const [
    { skillDirCommands, pluginSkills, bundledSkills, builtinPluginSkills },
    pluginCommands,
    workflowCommands,
  ] = await Promise.all([
    getSkills(cwd),                  // 扫描 skills 目录
    getPluginCommands(),             // 插件命令
    getWorkflowCommands(cwd),        // 工作流脚本
  ])

  return [
    ...bundledSkills,                // 最高优先级
    ...builtinPluginSkills,
    ...skillDirCommands,
    ...workflowCommands,
    ...pluginCommands,
    ...pluginSkills,
    ...COMMANDS(),                   // 最低优先级
  ]
})
```

### Token 估算

```typescript
export function estimateSkillFrontmatterTokens(skill: Command): number
// 仅从 name + description + whenToUse 估算 Token
// Skill 完整内容在调用时才懒加载
```

**设计决策**: Skill 的完整内容在调用时才加载，初始阶段仅传递 frontmatter 信息给模型。这显著减少了系统提示的 Token 消耗。

### 内置 Skills

内置 Skills 位于 `src/skills/bundled/`，提供核心工作流能力。

### Skill 执行

Skills 被转化为 `Command` 对象，类型为 `prompt`：

```typescript
type PromptCommand = Command & {
  type: 'prompt'
  // 执行时将 skill 内容作为系统提示注入
  // 支持 shell 命令替换和变量展开
}
```

## 二、Plugins 系统

### Plugin 生命周期

```typescript
export type PluginOperationResult = {
  success: boolean
  message: string
  pluginId?: string                // e.g., 'slack@anthropic'
  pluginName?: string
  scope?: PluginScope              // 'user' | 'project' | 'local' | 'managed'
  reverseDependents?: string[]     // 依赖此插件的其他插件
}

export const VALID_INSTALLABLE_SCOPES = ['user', 'project', 'local']
export const VALID_UPDATE_SCOPES = [..., 'managed']
```

### Plugin 安装范围

| 范围 | 路径 | 说明 |
|------|------|------|
| `user` | `~/.claude/plugins/` | 用户全局 |
| `project` | `.claude/plugins/` | 项目级 (可提交) |
| `local` | `.claude/plugins.local/` | 项目级 (gitignored) |
| `managed` | 管理策略路径 | 企业管理注入 |

### Plugin 能力

Plugins 可以提供：
- **Skills/Commands**: 通过 `.claude/skills` 和 `.claude/commands` 目录
- **MCP 服务器配置**: 完整的 MCP 服务器定义
- **工具**: 通过 MCP 协议注册的工具

### Plugin 与 Skill 对比

| 方面 | Skills | Plugins |
|------|--------|---------|
| 格式 | Markdown 文件 | Archive + manifest + 依赖 |
| 安装范围 | user / project / bundled | user / project / local / managed |
| 生命周期 | 文件即配置 | 安装 / 启用 / 禁用 / 卸载 |
| MCP 支持 | 可在 frontmatter 中定义 | 完整 MCP 配置 |
| 更新方式 | 手动编辑文件 | 版本检查 (manifest) |
| 依赖管理 | 无 | 支持依赖声明和反向依赖检查 |

### Plugin 加载

```typescript
// src/services/plugins/
// Plugin 发现和加载流程:
// 1. 扫描各范围的 plugins 目录
// 2. 读取 manifest
// 3. 检查版本和依赖
// 4. 加载 MCP 配置
// 5. 注册 tools 和 commands
// 6. 启用/禁用状态管理
```

### Plugin 状态 (AppState)

```typescript
plugins: {
  enabled: LoadedPlugin[]          // 已启用的插件
  disabled: LoadedPlugin[]         // 已禁用的插件
  commands: Command[]              // 插件提供的命令
  errors: PluginError[]            // 加载错误
  installationStatus: { ... }      // 安装状态
  needsRefresh: boolean            // 需要刷新标记
}
```

### Plugin 提供的 Skills

插件可以通过自身目录结构提供 Skills：
- 加载时 `source` 标记为 `'plugin'`
- Plugin-only 限制可锁定 MCP 仅供插件使用

## 三、动态 Skills

除了静态加载，系统还支持运行时动态发现的 Skills：

```typescript
export function getDynamicSkills(): Command[]
// 运行时发现的技能 (如 MCP 服务器新注册的工具)

// getCommands() 中的整合:
const uniqueDynamicSkills = dynamicSkills.filter(
  s => !baseCommandNames.has(s.name) &&
       meetsAvailabilityRequirement(s) &&
       isCommandEnabled(s)
)
```

动态 Skills 在 `getCommands()` 每次调用时重新检查可用性。

## 核心文件

| 文件 | 说明 |
|------|------|
| `skills/` (20 文件) | Skill 执行框架 |
| `skills/bundled/` | 内置 Skills |
| `skills/loadSkillsDir.ts` | Skill 目录加载 |
| `services/plugins/` | Plugin 加载器 |
| `services/plugins/builtinPlugins.ts` | 内置 Plugin |
| `tools/SkillTool/` | Skill 工具实现 |
| `commands.ts` | 命令注册与整合 |
