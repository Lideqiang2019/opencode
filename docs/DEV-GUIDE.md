# OpenCode CLI 开发调试指南

本文档说明如何在本地开发环境中启动和调试 OpenCode CLI。

## 前置要求

- **Bun**: 项目使用 Bun 作为运行时（推荐版本：1.3.5）
- **Node.js**: 如果使用 npm/yarn/pnpm，需要 Node.js

## 快速开始

### 1. 安装依赖

```bash
# 在项目根目录（必须）
cd /Users/lintang/Documents/ai/opencode
bun install

# 这会安装所有 workspace 包的依赖
# 包括 packages/opencode、packages/sdk/js、packages/util 等
```

### 2. 启动开发模式（推荐方式）

**使用 `bun dev` 命令**（这是官方推荐的开发方式）：

```bash
# 在项目根目录
cd /Users/lintang/Documents/ai/opencode

# 查看所有可用命令
bun dev --help

# 运行具体命令
bun dev run "hello world"
bun dev models
bun dev auth list
bun dev serve              # 启动 API 服务器
bun dev web                # 启动服务器 + Web 界面

# 启动 TUI（默认在当前目录）
bun dev

# 在指定目录启动 TUI
bun dev /path/to/project
bun dev .                  # 在当前仓库根目录启动
```

**`bun dev` 等同于生产环境的 `opencode` 命令**，但运行的是开发版本的代码。

### 3. 直接运行源码（备选方式）

如果 `bun dev` 不工作，可以直接运行源码：

```bash
# 方式一：从根目录运行
bun run --cwd packages/opencode src/index.ts --help
bun run --cwd packages/opencode src/index.ts run "test"

# 方式二：进入包目录运行
cd packages/opencode
bun run src/index.ts --help
bun run src/index.ts run "test"
```

**注意**：直接运行源码可能会遇到模块解析问题，优先使用 `bun dev`。

### 3. 使用 Bun 的 `--watch` 模式（热重载）

```bash
# 监听文件变化并自动重启（需要修改 package.json 的 dev 脚本）
# 或者直接使用
bun --watch dev run "test"
```

### 4. 创建本地符号链接（可选）

如果需要创建一个全局命令别名：

```bash
# 创建一个 shell 别名（推荐）
echo 'alias opencode-dev="cd /Users/lintang/Documents/ai/opencode && bun dev"' >> ~/.zshrc
source ~/.zshrc

# 使用
opencode-dev --help
opencode-dev run "test"
```

或者创建一个简单的包装脚本：

```bash
# 创建 ~/bin/opencode-dev
cat > ~/bin/opencode-dev <<'EOF'
#!/bin/bash
cd /Users/lintang/Documents/ai/opencode
bun dev "$@"
EOF

chmod +x ~/bin/opencode-dev
```

## 调试技巧

### 1. 启用调试日志

```bash
# 设置日志级别
OPENCODE_LOG_LEVEL=DEBUG bun run packages/opencode/src/index.ts run "test"

# 打印日志到 stderr
bun run packages/opencode/src/index.ts --print-logs run "test"
```

### 2. 使用 VS Code 调试

创建 `.vscode/launch.json`：

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Debug OpenCode CLI",
      "type": "node",
      "request": "launch",
      "runtimeExecutable": "bun",
      "runtimeArgs": ["run", "packages/opencode/src/index.ts"],
      "args": ["run", "test message"],
      "cwd": "${workspaceFolder}",
      "console": "integratedTerminal",
      "env": {
        "OPENCODE_LOG_LEVEL": "DEBUG",
        "OPENCODE_PRINT_LOGS": "true"
      }
    },
    {
      "name": "Debug OpenCode TUI",
      "type": "node",
      "request": "launch",
      "runtimeExecutable": "bun",
      "runtimeArgs": ["run", "packages/opencode/src/index.ts"],
      "args": [],
      "cwd": "${workspaceFolder}",
      "console": "integratedTerminal"
    }
  ]
}
```

### 3. 使用 Node.js 调试器

```bash
# 使用 --inspect 标志
bun --inspect run packages/opencode/src/index.ts run "test"

# 然后在 Chrome 中打开 chrome://inspect
```

### 4. 使用 console.log 调试

在代码中添加 `console.log` 或使用 `Log` 模块：

```typescript
import { Log } from "./util/log"

const log = Log.create({ service: "my-service" })
log.debug("debug message", { data: "value" })
log.info("info message")
log.error("error message", { error })
```

## 常用开发命令

### 运行测试

```bash
# 运行所有测试
cd packages/opencode
bun test

# 运行特定测试文件
bun test src/cli/cmd/run.test.ts

# 带覆盖率
bun test --coverage
```

### 类型检查

```bash
# 检查类型
cd packages/opencode
bun run typecheck

# 或从根目录
bun run typecheck
```

### 构建二进制文件

```bash
# 构建当前平台的二进制文件
cd packages/opencode
bun run script/build.ts --single

# 构建所有平台
bun run script/build.ts
```

## 环境变量

开发时常用的环境变量：

```bash
# 日志相关
export OPENCODE_LOG_LEVEL=DEBUG          # DEBUG, INFO, WARN, ERROR
export OPENCODE_PRINT_LOGS=true          # 打印日志到 stderr

# 配置相关
export OPENCODE_CONFIG=./opencode.jsonc  # 使用自定义配置文件
export OPENCODE_CONFIG_DIR=./.opencode   # 自定义配置目录
export OPENCODE_DISABLE_PROJECT_CONFIG=1 # 禁用项目配置

# 功能开关
export OPENCODE_DISABLE_AUTOUPDATE=1     # 禁用自动更新
export OPENCODE_EXPERIMENTAL=1           # 启用实验性功能
export OPENCODE_CLIENT=cli               # 设置客户端类型

# 测试相关
export OPENCODE_TEST_HOME=/tmp/opencode-test  # 测试隔离目录
```

## 项目结构

```
packages/opencode/
├── src/
│   ├── index.ts              # CLI 入口文件
│   ├── cli/
│   │   ├── cmd/              # 所有 CLI 命令
│   │   │   ├── run.ts        # `opencode run` 命令
│   │   │   ├── auth.ts       # `opencode auth` 命令
│   │   │   └── ...
│   │   └── ...
│   ├── config/               # 配置管理
│   ├── provider/             # Provider 管理
│   ├── session/              # Session 管理
│   └── ...
├── bin/
│   └── opencode              # 二进制 shim（用于 npm 安装）
└── script/
    └── build.ts               # 构建脚本
```

## 常见问题

### Q: 如何测试 CLI 命令的修改？

A: 使用 `bun dev`：
```bash
bun dev <command> [args]
# 例如
bun dev run "test message"
bun dev models
bun dev auth list
```

### Q: 如何查看 CLI 的完整帮助？

A:
```bash
bun dev --help
bun dev <command> --help
# 例如
bun dev run --help
bun dev auth --help
```

### Q: 如何测试配置文件的修改？

A: 使用 `--config` 参数：
```bash
bun dev --config ./test-config.jsonc run "test"
```

### Q: 如何测试认证流程？

A:
```bash
# 列出已配置的认证
bun dev auth list

# 添加认证（交互式）
bun dev auth login

# 设置认证（非交互式）
echo "sk-xxx" | bun dev auth set openai --stdin
```

### Q: 如何调试特定命令？

A: 在命令代码中添加断点或日志，然后使用调试器：
```bash
# VS Code: 设置断点，然后按 F5
# 或使用 console.log
```

## 开发工作流示例

### 1. 修改 CLI 命令

```bash
# 1. 编辑文件
vim packages/opencode/src/cli/cmd/run.ts

# 2. 测试修改
bun dev run "test message"

# 3. 如果使用 watch 模式（需要修改 package.json）
bun --watch dev run "test"
```

### 2. 添加新命令

```bash
# 1. 创建新命令文件
vim packages/opencode/src/cli/cmd/mycommand.ts

# 2. 在 src/index.ts 中注册（添加 .command(MyCommand)）
# 3. 测试
bun dev mycommand --help
```

### 3. 调试 Provider 配置

```bash
# 1. 创建测试配置
cat > test-config.jsonc <<EOF
{
  "provider": {
    "openai": {
      "options": {
        "apiKey": "test-key"
      }
    }
  }
}
EOF

# 2. 使用测试配置运行
bun dev --config test-config.jsonc models
```

## 性能分析

```bash
# 使用 Bun 的性能分析
bun --profile run packages/opencode/src/index.ts run "test"

# 查看内存使用
bun --inspect run packages/opencode/src/index.ts run "test"
```

## 相关文档

- [CLI 代码映射](./CLI-CODEMAP.md) - CLI 代码结构说明
- [插件系统](./PLUGIN-SYSTEM.md) - 插件开发指南
- [包结构说明](./PACKAGES.md) - 项目包结构
