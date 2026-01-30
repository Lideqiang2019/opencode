# packages/ 目录说明

本文档介绍 `packages/` 目录下各个包的作用和职责。

## 核心包（Core Packages）

### `opencode/` - CLI 核心包
**作用**：OpenCode CLI 的核心实现，包含所有命令行功能。

**主要功能**：
- CLI 入口和命令注册（`src/index.ts`）
- 所有 CLI 命令实现（`src/cli/cmd/**`）
- TUI（终端用户界面，`src/cli/cmd/tui/**`）
- 服务器实现（`src/server/**`）
- Agent、Session、Provider、Tool 等核心业务逻辑
- 二进制构建脚本（`script/build.ts`）

**依赖关系**：
- 依赖 `@opencode-ai/sdk`、`@opencode-ai/util`、`@opencode-ai/plugin`
- 被 `app/`、`desktop/`、`web/` 等前端包使用

---

### `sdk/js/` - JavaScript SDK
**作用**：OpenCode API 的 TypeScript/JavaScript SDK，用于与 OpenCode 服务器通信。

**主要功能**：
- 自动生成的 OpenAPI 客户端（`src/gen/**`、`src/v2/gen/**`）
- 客户端和服务端 SDK（`src/client.ts`、`src/server.ts`）
- v2 API 客户端（`src/v2/client.ts`）

**使用场景**：
- CLI 内部使用（通过 `@opencode-ai/sdk`）
- Web 应用集成
- 第三方工具集成

---

### `util/` - 工具库
**作用**：共享的工具函数和类型定义。

**主要功能**：
- 错误处理（`src/error.ts`）
- 其他通用工具函数

**依赖关系**：
- 被几乎所有其他包使用

---

### `plugin/` - 插件系统
**作用**：OpenCode 插件开发的基础包，定义插件接口和工具。

**主要功能**：
- 插件类型定义（`src/index.ts`）
- Tool 插件接口（`src/tool.ts`）
- 插件示例（`src/example.ts`、`src/shell.ts`）

**使用场景**：
- 用户开发自定义插件
- CLI 加载和执行插件

---

### `script/` - 构建脚本工具
**作用**：提供版本号、渠道等构建时元数据。

**主要功能**：
- 版本管理（`src/index.ts`）

**使用场景**：
- 构建脚本中获取版本信息
- CLI 显示版本号

---

## 前端应用包（Frontend Applications）

### `app/` - Web 应用前端
**作用**：OpenCode 的 Web 应用前端（SolidJS + Vite）。

**主要功能**：
- Web UI 界面（`src/**`）
- E2E 测试（`e2e/**`）
- 使用 `@opencode-ai/ui` 组件库
- 通过 `@opencode-ai/sdk` 与后端通信

**技术栈**：
- SolidJS
- Vite
- TailwindCSS
- Playwright（E2E 测试）

---

### `desktop/` - 桌面应用
**作用**：基于 Tauri 的桌面应用包装器。

**主要功能**：
- Tauri 配置（`src-tauri/**`）
- 桌面应用入口（`src/**`）
- 复用 `@opencode-ai/app` 的 Web UI

**技术栈**：
- Tauri（Rust + Web）
- 复用 `app/` 包的前端代码

---

### `web/` - 文档网站
**作用**：OpenCode 的官方文档网站（Astro + Starlight）。

**主要功能**：
- 文档内容（`.mdx` 文件）
- 使用 Astro Starlight 主题
- API 文档生成

**技术栈**：
- Astro
- Starlight 文档主题

---

### `ui/` - UI 组件库
**作用**：共享的 UI 组件库（SolidJS 组件）。

**主要功能**：
- 可复用的 SolidJS 组件
- 图标资源（大量 SVG 文件）
- 字体资源

**使用场景**：
- `app/`、`desktop/`、`console/app/` 等前端应用使用

---

## 后端服务包（Backend Services）

### `console/` - 控制台应用（多包）
**作用**：OpenCode 的 SaaS 控制台应用，包含多个子包。

#### `console/app/` - 控制台前端
**作用**：控制台 Web 应用前端（SolidJS Start）。

**主要功能**：
- 用户管理界面
- 会话管理
- 计费/订阅管理（Stripe 集成）
- 使用 `@opencode-ai/console-core` 作为后端

**技术栈**：
- SolidJS Start
- Cloudflare Workers（部署目标）

#### `console/core/` - 控制台后端核心
**作用**：控制台的后端业务逻辑和数据库。

**主要功能**：
- 数据库 Schema（Drizzle ORM）
- 业务逻辑（用户、会话、计费等）
- Stripe 集成
- 数据库迁移脚本

**技术栈**：
- Drizzle ORM
- PlanetScale/PostgreSQL
- Stripe API

#### `console/function/` - Cloudflare Functions
**作用**：控制台的 Cloudflare Workers Functions。

**主要功能**：
- 边缘函数实现

#### `console/mail/` - 邮件模板
**作用**：邮件模板和资源。

**主要功能**：
- JSX Email 模板
- 邮件相关资源

#### `console/resource/` - 资源管理
**作用**：控制台的资源管理相关代码。

---

### `enterprise/` - 企业版应用
**作用**：OpenCode 企业版的前端应用。

**主要功能**：
- 企业级功能界面
- 团队管理
- 部署到 Cloudflare Workers

**技术栈**：
- SolidJS Start
- Cloudflare Workers

---

### `function/` - 通用 Functions
**作用**：通用的 Cloudflare Workers Functions。

**主要功能**：
- GitHub App 认证（`@octokit/auth-app`）
- JWT 处理（`jose`）

---

## 集成包（Integrations）

### `slack/` - Slack 集成
**作用**：OpenCode 的 Slack Bot 集成。

**主要功能**：
- Slack Bolt 框架集成
- Slack 命令和交互处理

**技术栈**：
- Slack Bolt SDK

---

### `extensions/zed/` - Zed 编辑器扩展
**作用**：Zed 编辑器的 OpenCode 扩展。

**主要功能**：
- Zed 扩展配置和资源

---

## 资源包（Resources）

### `docs/` - 文档内容
**作用**：文档网站的 Markdown 内容。

**主要功能**：
- 文档源文件（`.mdx`）
- 图片资源

---

### `identity/` - 品牌资源
**作用**：OpenCode 的品牌标识和图标。

**主要功能**：
- Logo 文件（PNG、SVG）
- 不同尺寸的图标

---

## 依赖关系图

```
opencode (CLI核心)
├── sdk/js (API客户端)
├── util (工具库)
└── plugin (插件系统)

app (Web前端)
├── sdk/js
├── ui
└── util

desktop (桌面应用)
├── app (复用Web UI)
└── ui

console/app (控制台前端)
├── console/core (后端)
├── ui
└── sdk/js

enterprise (企业版)
├── ui
└── util

web (文档网站)
└── opencode (用于生成API文档)
```

## 总结

- **核心包**：`opencode/`、`sdk/js/`、`util/`、`plugin/`、`script/` - 提供核心功能和工具
- **前端应用**：`app/`、`desktop/`、`web/`、`ui/` - 用户界面相关
- **后端服务**：`console/*`、`enterprise/`、`function/` - 服务器端逻辑
- **集成包**：`slack/`、`extensions/` - 第三方集成
- **资源包**：`docs/`、`identity/` - 文档和资源文件
