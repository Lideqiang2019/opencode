# OpenCode 插件系统详解

本文档详细说明 OpenCode 插件系统的工作原理，以及如何通过插件（如 Oh-My-OpenCode）增强 OpenCode 的功能。

## 目录

1. [插件系统概述](#插件系统概述)
2. [插件加载机制](#插件加载机制)
3. [插件可以做什么](#插件可以做什么)
4. [如何增强 OpenCode](#如何增强-opencode)
5. [实际例子：Oh-My-OpenCode](#实际例子oh-my-opencode)

---

## 插件系统概述

OpenCode 的插件系统允许第三方开发者扩展核心功能，包括：

- **添加自定义 Tool**：扩展 AI 可以使用的工具
- **添加自定义 Agent**：创建专门的 AI 代理
- **添加自定义 Skill**：提供可复用的技能模板
- **修改配置**：动态调整 OpenCode 的行为
- **拦截事件**：监听和响应系统事件
- **自定义认证**：为新的 Provider 添加认证流程

---

## 插件加载机制

### 1. 插件配置位置

插件在 `opencode.json` 或 `opencode.jsonc` 配置文件中声明：

```jsonc
{
  "plugin": [
    "oh-my-opencode@2.4.3",           // npm 包
    "file:///path/to/local-plugin.js" // 本地文件
  ]
}
```

### 2. 插件加载流程

```typescript
// packages/opencode/src/plugin/index.ts

// 1. 从配置文件读取插件列表
const plugins = config.plugin ?? []

// 2. 内置插件（自动加载）
const BUILTIN = [
  "opencode-anthropic-auth@0.0.13",
  "@gitlab/opencode-gitlab-auth@1.3.2"
]

// 3. 内部插件（直接导入）
const INTERNAL_PLUGINS = [CodexAuthPlugin, CopilotAuthPlugin]

// 4. 加载每个插件
for (let plugin of plugins) {
  // npm 包：自动安装
  if (!plugin.startsWith("file://")) {
    plugin = await BunProc.install(pkg, version)
  }
  
  // 导入并初始化
  const mod = await import(plugin)
  const init = await fn(pluginInput)
  hooks.push(init)
}
```

### 3. 插件输入上下文

每个插件在初始化时接收 `PluginInput`：

```typescript
type PluginInput = {
  client: OpencodeClient      // SDK 客户端
  project: Project            // 项目信息
  directory: string           // 当前工作目录
  worktree: string            // Git worktree 根目录
  serverUrl: URL              // 服务器 URL
  $: BunShell                 // Shell 执行器
}
```

---

## 插件可以做什么

### 1. 添加自定义 Tool

**定义 Tool**：

```typescript
// 插件代码
import { tool } from "@opencode-ai/plugin"

export const MyPlugin: Plugin = async (input) => {
  return {
    tool: {
      // 定义一个新的 tool
      my_custom_tool: tool({
        description: "执行自定义操作",
        args: {
          param1: tool.schema.string().describe("参数1"),
          param2: tool.schema.number().optional().describe("参数2"),
        },
        async execute(args, context) {
          // context 包含：
          // - sessionID: 会话 ID
          // - messageID: 消息 ID
          // - agent: Agent 名称
          // - directory: 项目目录
          // - worktree: Git worktree
          // - abort: AbortSignal
          // - metadata(): 设置工具元数据
          // - ask(): 请求权限
          
          const result = await doSomething(args.param1)
          return result
        },
      }),
    },
  }
}
```

**Tool 注册流程**：

```typescript
// packages/opencode/src/tool/registry.ts

// 1. 从插件收集 tools
const plugins = await Plugin.list()
for (const plugin of plugins) {
  for (const [id, def] of Object.entries(plugin.tool ?? {})) {
    custom.push(fromPlugin(id, def))
  }
}

// 2. 合并到工具列表
const allTools = [
  BashTool,      // 内置工具
  ReadTool,
  EditTool,
  ...custom,     // 插件工具
]
```

### 2. 添加自定义 Agent

**Agent 不是通过插件直接添加的，而是通过配置文件**：

```markdown
<!-- .opencode/agent/my-agent.md -->
---
model: anthropic/claude-sonnet-4
temperature: 0.7
description: 专门用于代码审查的 Agent
mode: subagent
---

你是一个专业的代码审查 Agent。

你的职责是：
- 检查代码质量和最佳实践
- 发现潜在的 bug
- 提供改进建议
```

**Agent 加载流程**：

```typescript
// packages/opencode/src/agent/agent.ts

// 1. 扫描配置目录
const AGENT_GLOB = new Bun.Glob("{agent,agents}/**/*.md")

for (const dir of await Config.directories()) {
  for await (const item of AGENT_GLOB.scan({ cwd: dir })) {
    const md = await ConfigMarkdown.parse(item)
    const agent = {
      name: agentName,
      ...md.data,        // 从 frontmatter 读取配置
      prompt: md.content // 从 markdown 内容读取 prompt
    }
    result[agentName] = agent
  }
}
```

**插件可以修改 Agent 配置**：

```typescript
export const MyPlugin: Plugin = async (input) => {
  return {
    config: async (config) => {
      // 动态添加或修改 agent 配置
      config.agent = {
        ...config.agent,
        "my-agent": {
          model: "anthropic/claude-sonnet-4",
          prompt: "你是一个...",
        }
      }
    },
  }
}
```

### 3. 添加自定义 Skill

**Skill 也是通过配置文件添加**：

```markdown
<!-- .opencode/skill/my-skill.md -->
---
description: 一个可复用的技能模板
---

这是一个技能模板，可以在会话中通过 SkillTool 调用。
```

**Skill 加载流程**：

```typescript
// packages/opencode/src/skill/skill.ts

const OPENCODE_SKILL_GLOB = new Bun.Glob("skill/**/*.md")

for (const dir of await Config.directories()) {
  for await (const match of OPENCODE_SKILL_GLOB.scan({ cwd: dir })) {
    const md = await ConfigMarkdown.parse(match)
    skills[name] = {
      name,
      description: md.data.description,
      location: match,
    }
  }
}
```

### 4. 自定义认证流程

插件可以为新的 Provider 添加认证：

```typescript
export const MyAuthPlugin: Plugin = async (input) => {
  return {
    auth: {
      provider: "my-provider",
      
      // OAuth 认证
      methods: [{
        type: "oauth",
        label: "使用 OAuth 登录",
        authorize: async (inputs) => {
          return {
            url: "https://auth.example.com/authorize",
            method: "auto",
            callback: async () => {
              // 处理 OAuth 回调
              return {
                type: "success",
                refresh: "...",
                access: "...",
                expires: 3600,
              }
            },
          }
        },
      }],
      
      // 加载 Provider 配置
      loader: async (getAuth, provider) => {
        const auth = await getAuth()
        return {
          apiKey: auth.key,
          baseURL: "https://api.example.com",
        }
      },
    },
  }
}
```

### 5. 拦截和修改行为

插件可以通过 Hook 拦截各种事件：

```typescript
export const MyPlugin: Plugin = async (input) => {
  return {
    // 监听事件
    event: async ({ event }) => {
      if (event.type === "session.created") {
        console.log("新会话创建:", event.properties.sessionID)
      }
    },
    
    // 修改聊天参数
    "chat.params": async (input, output) => {
      // 修改 temperature、topP 等参数
      output.temperature = 0.9
      return output
    },
    
    // 修改消息
    "chat.message": async (input, output) => {
      // 在消息发送前修改
      output.message.text = "[前缀] " + output.message.text
    },
    
    // 拦截工具执行
    "tool.execute.before": async (input, output) => {
      // 修改工具参数
      output.args.customField = "value"
    },
    
    "tool.execute.after": async (input, output) => {
      // 修改工具输出
      output.output = "处理后的输出: " + output.output
    },
  }
}
```

---

## 如何增强 OpenCode

### Oh-My-OpenCode 的工作原理

Oh-My-OpenCode 是一个功能丰富的插件，它通过以下方式增强 OpenCode：

#### 1. 添加大量自定义 Tool

```typescript
// oh-my-opencode 插件代码（示例）
export const OhMyOpenCodePlugin: Plugin = async (input) => {
  return {
    tool: {
      // LSP 相关工具
      lsp_diagnostics: tool({...}),
      lsp_symbols: tool({...}),
      
      // AST 分析工具
      ast_parse: tool({...}),
      ast_transform: tool({...}),
      
      // MCP 工具
      mcp_call: tool({...}),
      
      // 其他增强工具
      ...更多工具
    },
  }
}
```

#### 2. 提供预配置的 Agent

插件可以在配置目录中提供 Agent 文件：

```
oh-my-opencode/
  .opencode/
    agent/
      code-reviewer.md
      test-generator.md
      doc-writer.md
      ...
```

#### 3. 提供预配置的 Skill

```
oh-my-opencode/
  .opencode/
    skill/
      refactoring.md
      testing.md
      ...
```

#### 4. 修改默认配置

```typescript
export const OhMyOpenCodePlugin: Plugin = async (input) => {
  return {
    config: async (config) => {
      // 启用实验性功能
      config.experimental = {
        ...config.experimental,
        batch_tool: true,
        lsp_tool: true,
      }
      
      // 添加默认 Agent
      config.agent = {
        ...config.agent,
        "code-reviewer": {
          model: "anthropic/claude-sonnet-4",
          prompt: "...",
        },
      }
    },
  }
}
```

#### 5. 安装流程

用户安装 Oh-My-OpenCode：

```bash
# 1. 安装 npm 包
npm install -g oh-my-opencode

# 2. 在 opencode.json 中配置
{
  "plugin": ["oh-my-opencode@latest"]
}

# 3. 插件会自动：
#    - 安装到 node_modules
#    - 加载插件代码
#    - 注册所有 tools
#    - 应用配置修改
```

---

## 插件系统架构图

```
┌─────────────────────────────────────────┐
│         OpenCode 核心系统                │
├─────────────────────────────────────────┤
│                                         │
│  ┌──────────────┐  ┌──────────────┐   │
│  │  ToolRegistry │  │ Agent Manager│   │
│  └──────┬───────┘  └──────┬───────┘   │
│         │                  │            │
│         └────────┬─────────┘            │
│                  │                      │
│         ┌────────▼─────────┐            │
│         │   Plugin System  │            │
│         └────────┬─────────┘            │
│                  │                      │
└──────────────────┼──────────────────────┘
                   │
         ┌─────────┴─────────┐
         │                   │
    ┌────▼────┐       ┌─────▼─────┐
    │  Plugin │       │  Plugin   │
    │  (npm)  │       │  (local)  │
    └────┬────┘       └─────┬─────┘
         │                   │
         └─────────┬─────────┘
                   │
         ┌─────────▼─────────┐
         │  Plugin Hooks     │
         │  - tool           │
         │  - auth           │
         │  - config         │
         │  - event           │
         │  - chat.*         │
         └───────────────────┘
```

---

## 总结

OpenCode 插件系统通过以下机制实现扩展：

1. **插件加载**：从 npm 或本地文件加载插件
2. **Tool 注册**：插件可以导出 `tool` 对象，自动注册到 ToolRegistry
3. **Agent/Skill**：通过配置文件（`.opencode/agent/*.md`、`.opencode/skill/*.md`）添加
4. **Hook 系统**：插件可以注册各种 hook 来拦截和修改行为
5. **配置修改**：插件可以动态修改 OpenCode 配置

Oh-My-OpenCode 这样的插件通过组合这些机制，提供了：
- 大量预构建的 Tool（LSP、AST、MCP 等）
- 预配置的 Agent（代码审查、测试生成等）
- 预配置的 Skill（重构、测试等）
- 优化的默认配置

这使得用户可以快速获得一个功能完整的 AI 开发环境，而无需手动配置每个组件。

---

## Claude Code Hooks 兼容性

### 支持情况

OpenCode **支持类似 Claude Code 的 hooks 机制**，但实现方式有所不同：

#### 1. **配置文件中的 Hooks**（实验性功能）

OpenCode 支持在配置文件中定义 hooks，类似于 Claude Code：

```jsonc
{
  "experimental": {
    "hook": {
      // 文件编辑后执行的 hooks
      "file_edited": {
        "*.ts": [
          {
            "command": ["bun", "run", "lint", "{file}"],
            "environment": {
              "NODE_ENV": "production"
            }
          }
        ],
        "*.{ts,tsx}": [
          {
            "command": ["prettier", "--write", "{file}"]
          }
        ]
      },
      // 会话完成时执行的 hooks
      "session_completed": [
        {
          "command": ["git", "add", "."],
          "environment": {
            "GIT_AUTHOR_NAME": "OpenCode"
          }
        }
      ]
    }
  }
}
```

**支持的 Hook 类型**：
- `file_edited`: 文件被编辑后触发（支持 glob 模式匹配）
- `session_completed`: 会话完成时触发

**注意**：这是一个实验性功能，可能需要通过插件系统来完整实现执行逻辑。

#### 2. **插件系统中的 Hooks**

OpenCode 的插件系统提供了更强大的 hooks 机制：

```typescript
export const MyHookPlugin: Plugin = async (input) => {
  return {
    // 监听文件编辑事件
    event: async ({ event }) => {
      if (event.type === "file.edited") {
        const file = event.properties.file
        // 执行自定义逻辑
        await input.$`prettier --write ${file}`
      }
    },
    
    // 拦截工具执行
    "tool.execute.after": async (input, output) => {
      if (input.tool === "edit" || input.tool === "write") {
        // 文件编辑后自动格式化
        await formatFile(output.metadata.filepath)
      }
    },
    
    // 会话完成时执行
    event: async ({ event }) => {
      if (event.type === "session.idle") {
        // 会话完成后的清理工作
        await cleanup()
      }
    },
  }
}
```

#### 3. **Claude Code 文件兼容性**

OpenCode 支持 Claude Code 的文件约定作为后备：

- **`CLAUDE.md`**: 项目级指令文件（如果不存在 `AGENTS.md`）
- **`~/.claude/CLAUDE.md`**: 全局指令文件（如果不存在 `~/.config/opencode/AGENTS.md`）
- **`~/.claude/skills/`**: Skills 目录

可以通过环境变量禁用：
```bash
export OPENCODE_DISABLE_CLAUDE_CODE=1        # 禁用所有 .claude 支持
export OPENCODE_DISABLE_CLAUDE_CODE_PROMPT=1 # 仅禁用 ~/.claude/CLAUDE.md
export OPENCODE_DISABLE_CLAUDE_CODE_SKILLS=1 # 仅禁用 .claude/skills
```

### 对比：Claude Code vs OpenCode Hooks

| 特性 | Claude Code | OpenCode |
|------|------------|----------|
| **配置文件 Hooks** | ✅ 完全支持 | ⚠️ 实验性（配置已定义，执行逻辑可能需插件） |
| **插件 Hooks** | ❌ 无插件系统 | ✅ 完整的插件系统 |
| **事件监听** | ✅ 文件系统事件 | ✅ 完整的事件总线系统 |
| **工具拦截** | ✅ 支持 | ✅ 支持（`tool.execute.before/after`） |
| **自定义认证** | ❌ | ✅ 通过插件系统 |

### 推荐使用方式

1. **简单场景**：使用配置文件中的 `experimental.hook`（如果已实现）
2. **复杂场景**：使用插件系统编写自定义 hooks
3. **迁移用户**：可以继续使用 `CLAUDE.md` 文件，OpenCode 会自动识别

### 示例：实现类似 Claude Code 的 hooks

```typescript
// .opencode/plugin/file-hooks.ts
import { Plugin } from "@opencode-ai/plugin"

export const FileHooksPlugin: Plugin = async (input) => {
  const config = await Config.get()
  const hooks = config.experimental?.hook
  
  return {
    event: async ({ event }) => {
      // 文件编辑 hooks
      if (event.type === "file.edited" && hooks?.file_edited) {
        const file = event.properties.file
        const ext = path.extname(file)
        
        for (const [pattern, commands] of Object.entries(hooks.file_edited)) {
          if (minimatch(file, pattern)) {
            for (const cmd of commands) {
              await input.$`${cmd.command.map(c => 
                c.replace("{file}", file)
              )}`.env(cmd.environment || {})
            }
          }
        }
      }
      
      // 会话完成 hooks
      if (event.type === "session.idle" && hooks?.session_completed) {
        for (const cmd of hooks.session_completed) {
          await input.$`${cmd.command}`.env(cmd.environment || {})
        }
      }
    },
  }
}
```

然后在 `opencode.json` 中启用：
```jsonc
{
  "plugin": ["file://.opencode/plugin/file-hooks.ts"],
  "experimental": {
    "hook": {
      "file_edited": {
        "*.ts": [{"command": ["prettier", "--write", "{file}"]}]
      }
    }
  }
}
```
