# OpenCode CLI 代码清单（提取/二次开发用）

> 目标：回答 `CLI.md` 里的三件事：**哪些是 CLI 代码**、**模型/密钥配置与调用在哪里**、以及**未来怎么从上游升级**。

## 1) 哪些是 CLI 的代码

### CLI 入口与分发

- **CLI 入口（yargs 注册命令）**：`packages/opencode/src/index.ts`
- **npm bin shim（定位平台二进制并转发参数）**：`packages/opencode/bin/opencode`
- **二进制打包（Bun compile + 平台矩阵 + TUI worker）**：`packages/opencode/script/build.ts`

### CLI 命令实现（yargs command modules）

主目录：`packages/opencode/src/cli/cmd/`

- **核心交互命令**：`run.ts`（支持 `--attach`/本地内置 server 两种执行模式）
- **服务命令**：`serve.ts`、`web.ts`
- **模型与统计**：`models.ts`、`stats.ts`
- **认证与凭据**：`auth.ts`
- **升级/卸载**：`upgrade.ts`、`uninstall.ts`
- **导入导出**：`import.ts`、`export.ts`
- **调试**：`debug/*`
- **TUI**：`tui/*`（包含 `worker.ts`、组件、路由、上下文等）

### CLI UI/错误处理/网络参数

- **CLI UI（输出/样式/logo）**：`packages/opencode/src/cli/ui.ts`、`packages/opencode/src/cli/logo.ts`
- **CLI 错误格式化**：`packages/opencode/src/cli/error.ts`
- **通用网络参数（port/hostname/mdns/cors 等）**：`packages/opencode/src/cli/network.ts`
- **CLI bootstrap（为命令提供 Instance 生命周期）**：`packages/opencode/src/cli/bootstrap.ts`

> 备注：桌面端也有 `packages/desktop/src/cli.ts` / `packages/desktop/src-tauri/src/cli.rs`，但它们属于 Desktop 产品线，不是本仓库 Node/Bun CLI 的命令实现主路径。

## 2) 模型配置与调用在哪里（你要“清楚标记”的点）

### 2.1 配置来源与优先级（Model / Provider / APIKey / baseURL）

配置解析入口：`packages/opencode/src/config/config.ts`

合并优先级（低 → 高）大致为：

- 远端 `.well-known/opencode`（当存在 wellknown auth 时）
- 全局用户配置（`~/.config/opencode/*` 等）
- `OPENCODE_CONFIG` 指定的自定义 config 文件
- 项目内 `opencode.jsonc` / `opencode.json`（可被 `OPENCODE_DISABLE_PROJECT_CONFIG` 关闭）
- `OPENCODE_CONFIG_CONTENT`（inline JSON，最高优先级）

关键字段：

- **默认模型**：`config.model`（格式：`provider/model`）
- **小模型**：`config.small_model`
- **Provider 覆盖**：`config.provider[providerId].options`（支持 `apiKey`、`baseURL` 等）

你后续要做“像 claude code proxy 一样的开源配置”，建议就围绕 `opencode.jsonc` + `{env:VAR}` / `{file:path}` 这两个能力来做：

- `{env:OPENAI_API_KEY}`：从环境变量注入
- `{file:~/.secrets/openai_key}`：从文件内容注入（会自动转义换行/引号）

### 2.2 APIKey/Token 的存储与注入

凭据存储与读取：

- **凭据命令（交互式 login/list/logout + 新增 set）**：`packages/opencode/src/cli/cmd/auth.ts`
- **凭据存储模块**：`packages/opencode/src/auth.ts`（实际文件通常在 `Global.Path.data/auth.json`）

Provider 注入点（把 key/baseURL/headers 变成真正给 SDK 的 options）：

- `packages/opencode/src/provider/provider.ts`
  - `getSDK()` 内会合并：
    - `provider.options`（来自 config 的 `provider.<id>.options`）
    - `provider.key`（来自 env 或 auth 存储）
    - `model.api.url`（默认 baseURL）
    - `model.headers`（每个模型级别 headers）

### 2.3 “模型被调用”的关键路径

模型最终会在 session 里被解析为 language model 并发起请求，关键入口（常用）：

- **LLM 会话层**：`packages/opencode/src/session/llm.ts`
- **Provider 选择/模型解析**：`packages/opencode/src/provider/provider.ts`

> 一句话总结：`Config` 决定“用哪个 provider/model + provider options”，`Provider` 把它们变成 AI SDK 可用的 client，`session/*` 负责真正发起模型调用与流式事件。

## 3) 让用户自定义模型和 APIKey（当前能力 + 本分支增强）

### 3.1 当前就支持的方式（推荐）

- **配置文件**：项目根目录 `opencode.jsonc`
  - `model`: `"openai/gpt-5"` 这类
  - `provider.openai.options.apiKey`: `"{env:OPENAI_API_KEY}"` 或 `"{file:...}"`
- **环境变量**：各 provider 的 env key（`Auth list` 会显示当前命中的 env）
- **凭据存储**：`opencode auth login`（交互式）

### 3.2 本分支新增（更适合自动化/脚本）

- **全局 CLI 参数**（在任意命令前生效）：
  - `--config <path>`：等价设置 `OPENCODE_CONFIG`
  - `--config-dir <dir>`：等价设置 `OPENCODE_CONFIG_DIR`
  - `--config-content <json>`：等价设置 `OPENCODE_CONFIG_CONTENT`
- **非交互式写入 APIKey**：
  - `opencode auth set <provider> [key]`
  - `echo "$KEY" | opencode auth set openai --stdin`

## 4) 未来上游 opencode 迭代了，怎么升级（保持可同步）

建议策略：**把“提取/增强”尽量做成“增量改动”（新增文件 + 小范围补丁），避免大规模删包/重构**，这样升级成本最低。

- **做法 A：长期维护一个分支（推荐）**
  - 上游在 `dev` 迭代，你的改动在 `cli-extract`
  - 定期执行：
    - `git fetch origin`
    - `git rebase origin/dev`（或 `merge origin/dev`）
  - 由于本分支主要是“文档 + 少量 CLI flags/命令补充”，冲突会很少

- **做法 B：保持补丁队列**
  - 把提取相关改动拆成少量 commit
  - 每次升级用 `git cherry-pick` 重新打到新 base 上

