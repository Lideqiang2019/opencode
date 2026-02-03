# 添加自定义模型指南

本文档介绍如何在 OpenCode 中添加自定义模型和 Provider。

## 目录

- [添加自定义 Provider](#添加自定义-provider)
- [为现有 Provider 添加自定义模型](#为现有-provider-添加自定义模型)
- [配置模型选项](#配置模型选项)
- [使用环境变量和文件引用](#使用环境变量和文件引用)
- [模型变体（Variants）](#模型变体variants)
- [常见问题](#常见问题)

---

## 添加自定义 Provider

如果你需要使用一个**不在 OpenCode 默认列表中的 OpenAI 兼容 Provider**，可以按以下步骤添加：

### 步骤 1: 添加认证信息

运行 `/connect` 命令，选择 **Other**，然后输入：

1. **Provider ID**：一个唯一的标识符（例如：`myprovider`）
2. **API Key**：你的 API 密钥

```bash
$ bun dev
# 然后在 TUI 中输入：
/connect
```

或者使用命令行：

```bash
# 使用 auth set 命令（非交互式）
echo "your-api-key" | bun dev auth set myprovider --stdin
```

### 步骤 2: 配置 Provider

在项目根目录或全局配置目录创建或编辑 `opencode.json` 或 `opencode.jsonc`：

```json title="opencode.json"
{
  "$schema": "https://opencode.ai/config.json",
  "provider": {
    "myprovider": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "我的 AI Provider",
      "options": {
        "baseURL": "https://api.myprovider.com/v1"
      },
      "models": {
        "my-model-name": {
          "name": "我的模型显示名称"
        }
      }
    }
  }
}
```

### 配置选项说明

- **`npm`**：使用的 AI SDK 包
  - `@ai-sdk/openai-compatible`：适用于 OpenAI 兼容的 Provider
  - `@ai-sdk/cerebras`：适用于 Cerebras
  - 其他 AI SDK 包：根据 Provider 选择对应的包
- **`name`**：在 UI 中显示的 Provider 名称
- **`options.baseURL`**：API 端点 URL
- **`options.apiKey`**：可选，如果不想使用 auth 存储，可以直接在这里设置
- **`options.headers`**：可选，自定义请求头
- **`models`**：该 Provider 下可用的模型列表

### 完整示例

```jsonc title="opencode.jsonc"
{
  "$schema": "https://opencode.ai/config.json",
  "provider": {
    "myprovider": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "我的 AI Provider",
      "options": {
        "baseURL": "https://api.myprovider.com/v1",
        "apiKey": "{env:MYPROVIDER_API_KEY}",
        "headers": {
          "X-Custom-Header": "value"
        },
        "timeout": 60000
      },
      "models": {
        "my-model-name": {
          "name": "我的模型显示名称",
          "limit": {
            "context": 200000,
            "output": 65536
          }
        }
      }
    }
  }
}
```

---

## 为现有 Provider 添加自定义模型

如果你使用的是已存在的 Provider（如 `openai`、`anthropic` 等），但想添加新的模型：

```jsonc title="opencode.jsonc"
{
  "$schema": "https://opencode.ai/config.json",
  "provider": {
    "openai": {
      "models": {
        "gpt-5-custom": {
          "name": "GPT-5 Custom",
          "limit": {
            "context": 128000,
            "output": 16384
          }
        }
      }
    }
  }
}
```

---

## 配置模型选项

你可以为模型配置全局选项，这些选项会在调用模型时自动应用：

```jsonc title="opencode.jsonc"
{
  "$schema": "https://opencode.ai/config.json",
  "provider": {
    "openai": {
      "models": {
        "gpt-5": {
          "options": {
            "reasoningEffort": "high",
            "textVerbosity": "low",
            "reasoningSummary": "auto",
            "include": ["reasoning.encrypted_content"]
          }
        }
      }
    },
    "anthropic": {
      "models": {
        "claude-sonnet-4-5-20250929": {
          "options": {
            "thinking": {
              "type": "enabled",
              "budgetTokens": 16000
            }
          }
        }
      }
    }
  }
}
```

### 模型配置字段

- **`limit.context`**：模型接受的最大输入 token 数
- **`limit.output`**：模型可以生成的最大 token 数
- **`options`**：模型特定的选项（如 temperature、reasoningEffort 等）
- **`name`**：模型的显示名称
- **`id`**：可选，如果与模型 key 不同，可以指定实际的 API model ID

---

## 使用环境变量和文件引用

OpenCode 支持在配置中使用环境变量和文件引用：

### 环境变量引用

使用 `{env:VARIABLE_NAME}` 语法：

```jsonc title="opencode.jsonc"
{
  "provider": {
    "myprovider": {
      "options": {
        "apiKey": "{env:MYPROVIDER_API_KEY}",
        "baseURL": "{env:MYPROVIDER_BASE_URL}"
      }
    }
  }
}
```

### 文件引用

使用 `{file:path/to/file}` 语法读取文件内容：

```jsonc title="opencode.jsonc"
{
  "provider": {
    "myprovider": {
      "options": {
        "apiKey": "{file:~/.secrets/myprovider-key.txt}"
      }
    }
  }
}
```

---

## 模型变体（Variants）

变体允许你为同一个模型配置不同的设置，而无需创建重复的模型条目：

```jsonc title="opencode.jsonc"
{
  "$schema": "https://opencode.ai/config.json",
  "provider": {
    "opencode": {
      "models": {
        "gpt-5": {
          "variants": {
            "high": {
              "reasoningEffort": "high",
              "textVerbosity": "low",
              "reasoningSummary": "auto"
            },
            "low": {
              "reasoningEffort": "low",
              "textVerbosity": "low",
              "reasoningSummary": "auto"
            }
          }
        }
      }
    }
  }
}
```

使用变体时，模型 ID 格式为：`provider/model@variant`，例如：`opencode/gpt-5@high`

---

## 配置文件位置

OpenCode 会按以下顺序查找配置文件：

1. **项目配置**：`./opencode.json` 或 `./opencode.jsonc`
2. **全局配置**：`~/.config/opencode/opencode.json` 或 `~/.config/opencode/opencode.jsonc`
3. **命令行参数**：
   - `--config`：指定配置文件路径
   - `--config-dir`：指定额外配置目录
   - `--config-content`：内联 JSON 配置内容

### 使用命令行参数

```bash
# 使用指定配置文件
bun dev --config ./custom-config.jsonc run "test"

# 使用配置目录
bun dev --config-dir ./custom-configs run "test"

# 使用内联配置
bun dev --config-content '{"provider":{"myprovider":{"npm":"@ai-sdk/openai-compatible","options":{"baseURL":"https://api.example.com"}}}}' run "test"
```

---

## 验证配置

配置完成后，运行以下命令验证：

```bash
# 列出所有可用的模型
bun dev models

# 列出所有 Provider
bun dev models --json | jq '.providers[] | select(.id == "myprovider")'

# 测试运行
bun dev run "hello" --model myprovider/my-model-name
```

---

## 常见问题

### 1. 模型不显示在列表中

**检查项：**
- 确认 Provider ID 在配置中正确
- 确认已添加认证信息（`bun dev auth list`）
- 确认 `npm` 字段指向正确的 AI SDK 包
- 确认 `baseURL` 正确

**调试：**
```bash
# 启用调试日志
bun dev --print-logs --log-level DEBUG models
```

### 2. API Key 相关问题

**如果使用环境变量：**
```bash
export MYPROVIDER_API_KEY="your-key"
bun dev run "test"
```

**如果使用 auth 存储：**
```bash
# 查看已存储的认证
bun dev auth list

# 设置认证
bun dev auth set myprovider
```

### 3. 自定义 Provider 需要特定的 AI SDK 包

某些 Provider 需要特定的 AI SDK 包，例如：

- **Cerebras**：`@ai-sdk/cerebras`
- **OpenAI 兼容**：`@ai-sdk/openai-compatible`
- **Anthropic**：`@ai-sdk/anthropic`（通常已内置）

查看 [AI SDK 文档](https://ai-sdk.dev/) 了解支持的 Provider。

### 4. 模型选项不生效

确保：
- 选项格式正确（参考 Provider 的文档）
- 模型 ID 正确（`provider/model` 格式）
- 配置已正确加载（检查日志）

### 5. 超时设置

可以为 Provider 设置超时：

```jsonc title="opencode.jsonc"
{
  "provider": {
    "myprovider": {
      "options": {
        "timeout": 60000  // 60 秒，单位：毫秒
        // 或设置为 false 禁用超时
        // "timeout": false
      }
    }
  }
}
```

---

## 高级配置示例

### 示例 1：使用代理服务

```jsonc title="opencode.jsonc"
{
  "$schema": "https://opencode.ai/config.json",
  "provider": {
    "openai": {
      "options": {
        "baseURL": "https://proxy.example.com/v1",
        "headers": {
          "X-Proxy-Auth": "{env:PROXY_AUTH_TOKEN}"
        }
      }
    }
  }
}
```

### 示例 2：多个自定义模型

```jsonc title="opencode.jsonc"
{
  "$schema": "https://opencode.ai/config.json",
  "provider": {
    "myprovider": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "我的 Provider",
      "options": {
        "baseURL": "https://api.myprovider.com/v1"
      },
      "models": {
        "model-1": {
          "name": "模型 1",
          "limit": {
            "context": 128000,
            "output": 16384
          }
        },
        "model-2": {
          "name": "模型 2",
          "limit": {
            "context": 256000,
            "output": 32768
          },
          "options": {
            "temperature": 0.7
          }
        }
      }
    }
  }
}
```

### 示例 3：使用模型变体

```jsonc title="opencode.jsonc"
{
  "$schema": "https://opencode.ai/config.json",
  "provider": {
    "myprovider": {
      "models": {
        "base-model": {
          "name": "基础模型",
          "variants": {
            "fast": {
              "temperature": 0.3,
              "maxTokens": 1000
            },
            "creative": {
              "temperature": 0.9,
              "maxTokens": 2000
            },
            "precise": {
              "temperature": 0.1,
              "maxTokens": 4000
            }
          }
        }
      }
    }
  }
}
```

使用变体：
```bash
bun dev run "test" --model myprovider/base-model@fast
```

---

## 相关文档

- [Provider 文档](/docs/providers)
- [模型配置文档](/docs/models)
- [配置文件文档](/docs/config)
- [AI SDK 文档](https://ai-sdk.dev/)

---

## 总结

添加自定义模型的步骤：

1. ✅ 添加认证信息（`/connect` 或 `auth set`）
2. ✅ 创建配置文件（`opencode.json` 或 `opencode.jsonc`）
3. ✅ 配置 Provider（`npm`、`baseURL`、`options`）
4. ✅ 定义模型（`models` 字段）
5. ✅ 验证配置（`bun dev models`）
6. ✅ 使用模型（`bun dev run` 或 `/models` 命令）

如有问题，请查看日志或提交 Issue。
