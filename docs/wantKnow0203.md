# 你现在是一个老师，我需要在1h了解项目的构造，方便我对项目进行更改，你需要引导我慢慢了解项目的构造
## 目标 1h之后我可以将opencode改成我自己的CLI工具
1. 虽然我想自己做一个CLI，但是upstream有一些更新，我希望跟踪

### 未来我想要加长记忆能力和定时器任务
1. 参考github中的openclaw的长记忆能力和定时器能力。

### vscode扩展增强
1. 在修改文件的时候添加是否接受，accept undo操作

# 注意
1. 你先不要急着自己做，而是逐步引导我了解项目的结构和架构设计之后，然后我们再交流
2. 我们的交流结果都可以写入到docs中

# 在用户结束的时候增加 信息里面应该要包含使用的token数量，sessionId，使用时长，调用Tool Calls数量和时间，
 Agent Active
 Interaction Summary                                                                                               │
│  Session ID:                 a8805a31-6edf-4f57-be4e-e88e0572568e                                                  │
│  Tool Calls:                 0 ( ✓ 0 x 0 )                                                                         │
│  Success Rate:               0.0%                                                                                  │
│                                                                                                                    │
│  Performance                                                                                                       │
│  Wall Time:                  30.8s                                                                                 │
│  Agent Active:               0s                                                                                    │
│    » API Time:               0s (0.0%)                                                                             │
│    » Tool Time:              0s (0.0%)                                                                             
╭────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────╮
│                                                                                                                                                    │
│  opencode CLI已经关闭。再见！                                                                                                                         │
│                                                                                                                                                    │
│  性能                                                                                                                                              │
│  总耗时：                    18.2s                                                                                                                 │
│  iFlow CLI活动时间：         0s                                                                                                                    │
│    » API 时间：              0s (0.0%)                                                                                                             │
│    » 工具时间：              0s (0.0%)                                                                                                             │
│                                                                                                                                                    │
│                                                                                                                                                    │
╰───────────────────────────────────────────────────

## OpenCode CLI/TUI 架构学习笔记（2025-02-03）

### 1. 总体分层

- **CLI 层**（命令入口）  
  - `packages/opencode/src/index.ts`：`yargs` 注册所有 CLI 命令（如 `run`、`tui`、`acp` 等），负责解析参数、初始化日志、统一错误处理。  
  - `packages/opencode/src/cli/cmd/**`：每个子命令一个文件（如 `tui/thread.ts`、`auth.ts`、`serve.ts`）。

- **Worker / Server 层**（本地后端）  
  - `tui/thread.ts`：`opencode` 默认命令，起一个 `Worker` 子进程，通过 RPC 与 `worker.ts` 通信，并调用 `tui()` 渲染 TUI。  
  - `tui/worker.ts`：在子进程内调用 `Server.App()` 处理请求，并暴露 `fetch`/`server`/`shutdown` 等 RPC 方法。  
  - `server/server.ts`：基于 Hono 构建 HTTP 应用，挂载 `/session`、`/project`、`/file`、`/provider`、`/mcp`、`/tui`、`/event` 等路由。

- **领域层**（业务与状态）  
  - `session/**`：会话、消息、总结、回滚、prompt 主循环。  
  - `agent/**`：Agent 定义与选择逻辑。  
  - `tool/**`：各种工具（读文件、列目录、任务等），通过 `ToolRegistry` 被会话执行。  
  - `storage/storage.ts`：基于 JSON 文件的 KV 存储。  
  - `project/**`：项目实例、VCS 信息、worktree 解析。

### 2. 从命令到 TUI 的调用链

1. 用户执行 `opencode`（或重命名后的 CLI）：  
   - `index.ts` 把默认命令派发到 `TuiThreadCommand`。
2. `TuiThreadCommand`：  
   - 解析 `project`、`--model`、`--agent`、`--session` 等参数；  
   - 选择 `worker.ts` 或编译后的 `worker.js`，起 `Worker` 子进程；  
   - 根据网络参数决定是否起 HTTP server；  
   - 组装 `url`/`fetch`/`events`/`args`，调用 `tui({ ... })`。
3. `tui/app.tsx`：  
   - 用 `SDKProvider` 创建 `createOpencodeClient` 的实例 `sdk`，并根据 `events` 或 SSE 订阅事件流；  
   - 组件通过 `useSDK()` 调用 `sdk.client.session.*`、`sdk.client.project.*` 等 API；  
   - `SyncProvider` 监听 `sdk.event`，把 `session.updated`、`message.updated`、`todo.updated` 等事件同步到本地 store。

### 3. Server 路由与会话 API

- `server/server.ts`：  
  - 中间件：错误处理、Basic Auth（`OPENCODE_SERVER_PASSWORD`）、请求日志、CORS、`Instance.provide(directory)`。  
  - 关键路由：  
    - `/session` → `SessionRoutes()`：会话增删改查、消息发送、回滚、分享、总结等。  
    - `/project` → `ProjectRoutes()`：项目列表/初始化。  
    - `/file` → `FileRoutes()`：文件读写、diff 等。  
    - `/provider` → `ProviderRoutes()`：Provider / 模型信息。  
    - `/mcp` → `McpRoutes()`：MCP 服务。  
    - `/event`：SSE 事件流（订阅 `Bus`）。

- `server/routes/session.ts`（路由层薄封装）：  
  - 负责把 HTTP 请求校验为参数，然后调用领域函数：  
    - `Session.create / get / list / update / remove / fork / share / summarize ...`  
    - `SessionPrompt.prompt / command / shell / cancel`  
    - `SessionRevert.revert / unrevert`、`SessionSummary.diff`、`Todo.get` 等。

### 4. 会话与 Prompt 执行

- `Session`（`session/index.ts`）：  
  - 定义 `Session.Info` 结构；  
  - 通过 `Storage` 持久化到 `<data>/storage/session/<projectID>/<sessionID>.json`；  
  - 提供 `create/get/list/update/remove/touch/fork` 等操作，并通过 `Bus` 发送事件。

- `SessionPrompt`（`session/prompt.ts`）：  
  - `PromptInput` 描述一次用户输入（会话 ID、parts、模型、agent、权限等）；  
  - `prompt()`：  
    - 读取会话 → 清理回滚状态 → 创建用户消息 → 更新会话时间；  
    - 兼容旧的 `tools` 字段为新的权限体系；  
    - 根据 `noReply` 决定是否进入 `loop(sessionID)`。  
  - `loop(sessionID)`：  
    - 管理会话“忙碌/可中断”状态；  
    - 组装 LLM 上下文，调用 `LLM` + `SessionProcessor`；  
    - 通过 `ToolRegistry` 调用各种 Tool（文件、shell、MCP 等）；  
    - 更新消息、会话，并通过事件总线通知 TUI / ACP / Web。

### 5. 持久化模型

- 根目录：`<Global.Path.data>/storage`（用户本地目录，例如 macOS 上的 `~/Library/Application Support/opencode/storage`）。  
- 典型 key → 路径：  
  - 会话：`["session", projectID, sessionID]` → `storage/session/<projectID>/<sessionID>.json`  
  - 消息：`["message", sessionID, messageID]` → `storage/message/<sessionID>/<messageID>.json`  
  - 消息 part：`["part", messageID, partID]` → `storage/part/<messageID>/<partID>.json`  
  - Todo：`["todo", sessionID]` → `storage/todo/<sessionID>.json`  
  - Diff：`["session_diff", sessionID]` → `storage/session_diff/<sessionID>.json`

领域代码（Session/MessageV2/Todo/Summary/Revert）调用 `Storage.*`，TUI/CLI 不直接操作磁盘。

### 6. Tools / LSP / ACP

- **Tools**（`tool/**`）：  
  - 通过 `ToolRegistry` 注册；在 `SessionPrompt.loop` 中由 `SessionProcessor` 根据 LLM 的 tool call 调用。  
  - 若要新增长记忆/定时任务 Tool：  
    1. 在 `tool/` 增加实现（定义 ID、输入 schema、`execute`）；  
    2. 在 `ToolRegistry` 注册；  
    3. 通过 Agent 配置 / Session 权限控制暴露；  
    4. 如有需要，再加 CLI/TUI 入口或专门的 HTTP API。

- **LSP**：  
  - `src/lsp` + `/lsp` 路由，管理语言服务器进程并提供状态；  
  - 为 Tool / Agent 提供诊断、符号等语义信息，提升代码相关操作智能度。

- **ACP（Agent Client Protocol）**：  
  - `opencode acp` 命令（`cli/cmd/acp.ts`）启动 ACP server；  
  - `acp/agent.ts` 把 OpenCode 封装为 ACP Agent，通过 `OpencodeClient` 驱动会话与权限；  
  - 方便外部 ACP 客户端把 OpenCode 当成标准化 Agent 后端来调用。

### 7. 后续改造思路（为“自己的 CLI”做准备）

- **改名与品牌**：`index.ts` 的 `.scriptName()`、`packages/opencode/package.json` 的 `name`/`bin`、`bin/opencode` 文件、`logo.ts`/`ui.ts` 的文案。  
- **保持可升级**：尽量用“增量改动”（新增命令、Tool、配置），避免大范围重写核心 Session/Prompt/Storage 逻辑，以便后续跟 upstream `dev` 分支 rebase。  
- **长记忆 / 定时任务**：  
  - 在 `Storage` 下新增前缀（如 `["memory", projectID, memoryID]`、`["task", projectID, taskID]`），通过 Tool 或专门命令读写；  
  - 调度逻辑可以作为新命令或后台进程，基于现有 `SessionPrompt` 再封装。