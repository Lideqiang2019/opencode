# OpenCode VSCode 扩展 - 选中代码感知功能

## 概述

**是的，OpenCode 有 VSCode 扩展，可以感知 VSCode 中选中的代码！**

OpenCode 提供了一个官方的 VSCode 扩展，可以：
- ✅ 自动检测当前打开的文件
- ✅ 自动检测选中的代码行
- ✅ 将文件引用和选中行号发送到 OpenCode 终端
- ✅ 支持快捷键快速插入文件引用

---

## 安装方式

### 自动安装（推荐）

1. 在 VSCode 中打开集成终端
2. 运行 `opencode` 命令
3. 扩展会自动安装

### 手动安装

在 VSCode Extension Marketplace 搜索 **OpenCode** 并安装。

---

## 功能详解

### 1. 自动上下文感知（Context Awareness）

当你打开 OpenCode 终端时，扩展会**自动**：

1. **检测当前文件**：获取当前打开的文件路径
2. **检测选中代码**：如果有选中代码，提取行号范围
3. **发送到终端**：将文件引用发送到 OpenCode 终端

**代码实现** (`sdks/vscode/src/extension.ts:103-136`):

```typescript
function getActiveFile() {
  const activeEditor = vscode.window.activeTextEditor
  if (!activeEditor) {
    return
  }

  const document = activeEditor.document
  const workspaceFolder = vscode.workspace.getWorkspaceFolder(document.uri)
  if (!workspaceFolder) {
    return
  }

  // 获取相对于工作区根目录的路径
  const relativePath = vscode.workspace.asRelativePath(document.uri)
  let filepathWithAt = `@${relativePath}`

  // 检查是否有选中代码
  const selection = activeEditor.selection
  if (!selection.isEmpty) {
    // 转换为 1-based 行号
    const startLine = selection.start.line + 1
    const endLine = selection.end.line + 1

    if (startLine === endLine) {
      // 单行选中
      filepathWithAt += `#L${startLine}`
    } else {
      // 多行选中
      filepathWithAt += `#L${startLine}-${endLine}`
    }
  }

  return filepathWithAt
}
```

**示例输出**：
- 无选中：`@src/index.ts`
- 单行选中：`@src/index.ts#L42`
- 多行选中：`@src/index.ts#L37-42`

### 2. 快捷键功能

#### 打开 OpenCode 终端

- **Mac**: `Cmd+Esc` 或 `Cmd+Shift+Esc`
- **Windows/Linux**: `Ctrl+Esc` 或 `Ctrl+Shift+Esc`

**行为**：
- `Cmd+Esc`: 如果已有 OpenCode 终端，聚焦它；否则创建新的
- `Cmd+Shift+Esc`: 总是创建新的终端会话

#### 插入文件引用

- **Mac**: `Cmd+Option+K`
- **Windows/Linux**: `Ctrl+Alt+K`

**功能**：将当前文件引用（包括选中行号）插入到 OpenCode 终端输入框。

**代码实现** (`sdks/vscode/src/extension.ts:24-41`):

```typescript
let addFilepathDisposable = vscode.commands.registerCommand(
  "opencode.addFilepathToTerminal",
  async () => {
    const fileRef = getActiveFile()
    if (!fileRef) {
      return
    }

    const terminal = vscode.window.activeTerminal
    if (!terminal) {
      return
    }

    if (terminal.name === TERMINAL_NAME) {
      // 如果 OpenCode 服务器已启动，通过 HTTP API 发送
      const port = terminal.creationOptions.env?.["_EXTENSION_OPENCODE_PORT"]
      port
        ? await appendPrompt(parseInt(port), fileRef)
        : terminal.sendText(fileRef, false)
      terminal.show()
    }
  }
)
```

### 3. 工作原理

#### 终端启动流程

```typescript
async function openTerminal() {
  // 1. 生成随机端口
  const port = Math.floor(Math.random() * (65535 - 16384 + 1)) + 16384
  
  // 2. 创建终端
  const terminal = vscode.window.createTerminal({
    name: TERMINAL_NAME,
    location: {
      viewColumn: vscode.ViewColumn.Beside,  // 分屏显示
      preserveFocus: false,
    },
    env: {
      _EXTENSION_OPENCODE_PORT: port.toString(),
      OPENCODE_CALLER: "vscode",
    },
  })

  // 3. 启动 OpenCode（指定端口）
  terminal.sendText(`opencode --port ${port}`)

  // 4. 等待服务器就绪
  let tries = 10
  let connected = false
  do {
    await new Promise((resolve) => setTimeout(resolve, 200))
    try {
      await fetch(`http://localhost:${port}/app`)
      connected = true
      break
    } catch (e) {}
    tries--
  } while (tries > 0)

  // 5. 如果连接成功，发送文件引用
  if (connected) {
    const fileRef = getActiveFile()
    if (fileRef) {
      await appendPrompt(port, `In ${fileRef}`)
    }
  }
}
```

#### 发送文件引用到终端

```typescript
async function appendPrompt(port: number, text: string) {
  await fetch(`http://localhost:${port}/tui/append-prompt`, {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
    },
    body: JSON.stringify({ text }),
  })
}
```

---

## 使用示例

### 场景 1: 快速打开并发送文件引用

1. 在 VSCode 中打开文件 `src/utils.ts`
2. 选中第 10-15 行代码
3. 按 `Cmd+Esc` 打开 OpenCode 终端
4. **自动发送**：`In @src/utils.ts#L10-15` 到终端

### 场景 2: 手动插入文件引用

1. 打开 OpenCode 终端（`Cmd+Esc`）
2. 在编辑器中选中代码
3. 按 `Cmd+Option+K`（Mac）或 `Ctrl+Alt+K`（Windows/Linux）
4. **手动插入**：`@src/utils.ts#L10-15` 到终端输入框

### 场景 3: 在对话中使用文件引用

在 OpenCode 终端中，你可以直接使用文件引用：

```
请解释 @src/utils.ts#L10-15 这段代码的作用
```

OpenCode 会自动读取该文件的指定行号范围。

---

## 技术细节

### 文件引用格式

OpenCode 使用 `@` 符号表示文件引用：

```
@<相对路径>#L<起始行>-<结束行>
```

**示例**：
- `@src/index.ts` - 整个文件
- `@src/index.ts#L42` - 第 42 行
- `@src/index.ts#L37-42` - 第 37-42 行

### HTTP API 端点

扩展通过 HTTP API 与 OpenCode 服务器通信：

- **检查服务器状态**: `GET http://localhost:{port}/app`
- **追加提示**: `POST http://localhost:{port}/tui/append-prompt`
  - Body: `{ "text": "@file.ts#L10-15" }`

### 环境变量

扩展通过环境变量传递信息给 OpenCode：

- `_EXTENSION_OPENCODE_PORT`: OpenCode 服务器端口
- `OPENCODE_CALLER`: 调用来源（`"vscode"`）

---

## 相关文件

- **扩展代码**: `sdks/vscode/src/extension.ts`
- **扩展配置**: `sdks/vscode/package.json`
- **文档**: `packages/web/src/content/docs/ide.mdx`

---

## 常见问题

### Q: 扩展没有自动安装？

**A**: 确保：
1. 在集成终端中运行 `opencode`（不是外部终端）
2. VSCode 有安装扩展的权限
3. CLI 命令已正确安装（`code`、`cursor` 等）

### Q: 快捷键不工作？

**A**: 
1. 检查快捷键是否被其他扩展占用
2. 在命令面板（`Cmd+Shift+P`）中搜索 "OpenCode" 命令
3. 手动绑定快捷键：`Preferences > Keyboard Shortcuts`

### Q: 文件引用格式不正确？

**A**: 
- 确保文件在工作区根目录下
- 相对路径从工作区根目录开始
- 行号是 1-based（第一行是 1）

### Q: 如何禁用自动发送文件引用？

**A**: 目前扩展会在打开终端时自动发送。如果需要禁用，可以：
1. 修改扩展代码（不推荐）
2. 提交 Issue 请求添加配置选项

---

## 开发扩展

如果你想修改或扩展功能：

1. **打开扩展目录**：
   ```bash
   code sdks/vscode
   ```

2. **安装依赖**：
   ```bash
   cd sdks/vscode
   bun install
   ```

3. **启动调试**：
   - 按 `F5` 启动调试
   - 会打开新的 VSCode 窗口，扩展已加载

4. **测试更改**：
   - 在调试窗口中按 `Cmd+Shift+P`
   - 搜索 "Developer: Reload Window"
   - 重新加载以查看更改

---

## 总结

✅ **OpenCode VSCode 扩展可以感知选中的代码**

主要功能：
1. ✅ 自动检测当前文件和选中代码
2. ✅ 自动发送文件引用到 OpenCode 终端
3. ✅ 支持快捷键快速插入文件引用
4. ✅ 通过 HTTP API 与 OpenCode 服务器通信

使用方式：
- **自动**：打开终端时自动发送文件引用
- **手动**：使用 `Cmd+Option+K` 插入文件引用

文件引用格式：`@<路径>#L<起始行>-<结束行>`
