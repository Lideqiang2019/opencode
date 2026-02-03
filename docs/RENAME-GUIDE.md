# OpenCode 重命名指南

本文档列出所有需要修改 OpenCode 名称的位置。

## 核心位置（必须修改）

### 1. CLI 命令名称

**文件**: `packages/opencode/src/index.ts`

```typescript
// 第 44 行
.scriptName("tuna-code")  // 改为你的新名称，例如 "mycode"
```

**影响**: 这是用户在命令行中使用的命令名称。

---

### 2. npm 包名

**文件**: `packages/opencode/package.json`

```json
{
  "name": "opencode",  // 第 4 行 - 改为你的新包名
  "bin": {
    "opencode": "./bin/opencode"  // 第 21 行 - 改为你的新命令名
  }
}
```

**影响**: npm 包名和二进制命令名。

---

### 3. 二进制文件路径

**文件**: `packages/opencode/bin/opencode`

这个文件是二进制 shim，文件名本身需要重命名：
- 文件名：`packages/opencode/bin/opencode` → `packages/opencode/bin/<新名称>`
- 文件内容中可能也有引用，需要检查

---

### 4. VSCode 扩展名称

**文件**: `sdks/vscode/package.json`

```json
{
  "name": "opencode",           // 第 2 行
  "displayName": "opencode",    // 第 3 行
  "description": "opencode for VS Code"  // 第 4 行
}
```

**文件**: `sdks/vscode/src/extension.ts`

```typescript
// 第 6 行
const TERMINAL_NAME = "opencode"  // 改为你的新名称

// 命令名称（第 13、24 行等）
vscode.commands.registerCommand("opencode.openTerminal", ...)
vscode.commands.registerCommand("opencode.addFilepathToTerminal", ...)
```

**影响**: VSCode 扩展的标识和命令。

---

## 环境变量和常量

### 5. 环境变量

**文件**: `packages/opencode/src/index.ts`

```typescript
// 第 92 行
process.env.OPENCODE = "1"  // 改为你的新环境变量名，例如 MYCODE

// 第 51、55、59 行 - 配置相关的环境变量
process.env.OPENCODE_CONFIG
process.env.OPENCODE_CONFIG_DIR
process.env.OPENCODE_CONFIG_CONTENT
```

**文件**: `packages/opencode/src/flag/flag.ts`

查找所有 `OPENCODE_*` 开头的环境变量定义。

---

### 6. 日志和标识

**文件**: `packages/opencode/src/index.ts`

```typescript
// 第 94 行
Log.Default.info("opencode", {  // 改为你的新名称
  version: Installation.VERSION,
  args: process.argv.slice(2),
})
```

---

## 配置文件和路径

### 7. 配置文件路径

查找所有包含以下路径的地方：
- `~/.config/opencode/`
- `~/.local/share/opencode/`
- `.opencode/`（项目目录）

**常见位置**:
- `packages/opencode/src/global.ts` - 全局路径定义
- `packages/opencode/src/auth.ts` - 认证文件路径
- `packages/opencode/src/config/config.ts` - 配置文件路径

---

### 8. 配置文件 schema URL

**文件**: `packages/opencode/src/config/config.ts` 或相关配置文件

查找：
```json
{
  "$schema": "https://opencode.ai/config.json"  // 可能需要改为新的域名
}
```

---

## UI 和显示文本

### 9. Logo 和 UI 文本

**文件**: `packages/opencode/src/cli/logo.ts`
**文件**: `packages/opencode/src/cli/ui.ts`

查找所有显示 "opencode" 或 "OpenCode" 的文本。

---

### 10. 错误消息和帮助文本

搜索所有包含 "opencode" 的字符串：
```bash
# 在项目根目录运行
grep -r "opencode" --include="*.ts" --include="*.tsx" --include="*.json" packages/opencode/src/
```

---

## 构建和发布

### 11. 构建脚本

**文件**: `packages/opencode/script/build.ts`

检查二进制文件名和输出路径。

---

### 12. 安装脚本

**文件**: `install`（项目根目录）

检查脚本中的命令名称和路径。

---

## 其他包和依赖

### 13. 工作区包名

**文件**: `package.json`（根目录）

检查 `workspaces` 配置中的包名引用。

---

### 14. 其他包的依赖

搜索所有 `@opencode-ai/*` 的引用，可能需要改为新的组织名。

**常见位置**:
- `packages/opencode/package.json` - dependencies
- `packages/*/package.json` - 其他包的依赖

---

## 快速修改清单

使用以下命令快速查找所有需要修改的位置：

```bash
# 查找所有包含 "opencode" 的文件（不区分大小写）
grep -r "opencode" --include="*.ts" --include="*.tsx" --include="*.json" --include="*.md" \
  packages/opencode/ sdks/vscode/ | grep -v node_modules | grep -v dist

# 查找所有包含 "OpenCode" 的文件
grep -r "OpenCode" --include="*.ts" --include="*.tsx" --include="*.json" --include="*.md" \
  packages/opencode/ sdks/vscode/ | grep -v node_modules | grep -v dist

# 查找环境变量
grep -r "OPENCODE" --include="*.ts" --include="*.tsx" packages/opencode/src/
```

---

## 修改步骤建议

1. **先修改核心位置**：
   - CLI 命令名称（`scriptName`）
   - npm 包名
   - 二进制文件名

2. **然后修改环境变量**：
   - 所有 `OPENCODE_*` 环境变量
   - 配置文件路径

3. **最后修改显示文本**：
   - UI 文本
   - 错误消息
   - 文档

4. **测试**：
   ```bash
   # 重新构建
   bun run packages/opencode/script/build.ts
   
   # 测试命令
   ./packages/opencode/bin/<新名称> --help
   ```

---

## 注意事项

1. **向后兼容**: 如果希望保持向后兼容，可以考虑：
   - 保留旧的命令作为别名
   - 支持旧的环境变量名

2. **配置文件迁移**: 用户现有的配置文件路径可能需要迁移

3. **VSCode 扩展**: 如果修改了扩展名称，需要：
   - 更新 Marketplace 上的扩展
   - 用户需要重新安装扩展

4. **文档更新**: 记得更新所有文档中的名称引用

---

## 示例：重命名为 "mycode"

假设要将 OpenCode 重命名为 "mycode"，主要修改：

1. `packages/opencode/src/index.ts`:
   ```typescript
   .scriptName("mycode")
   ```

2. `packages/opencode/package.json`:
   ```json
   {
     "name": "mycode",
     "bin": {
       "mycode": "./bin/mycode"
     }
   }
   ```

3. `packages/opencode/bin/mycode`（重命名文件）

4. 环境变量：
   ```typescript
   process.env.MYCODE = "1"
   process.env.MYCODE_CONFIG = ...
   ```

5. `sdks/vscode/package.json`:
   ```json
   {
     "name": "mycode",
     "displayName": "mycode"
   }
   ```

6. `sdks/vscode/src/extension.ts`:
   ```typescript
   const TERMINAL_NAME = "mycode"
   ```

---

## 需要帮助？

如果遇到问题，可以：
1. 使用 `grep` 命令查找所有引用
2. 逐步修改并测试
3. 检查构建和运行是否正常
