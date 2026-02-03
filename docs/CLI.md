
# 核心目标
提取CLI能力，我想要本地建一个工程

# 任务
1. 你需要理清楚当前项目中哪些是CLI的代码，列出一个文档，然后创建一个新的分支，将CLI代码提取出来。
2. 然后清楚地标记出来模型配置和调用的地方，我需要进一步可以让用户可以自定义模型和APIKey，像claude code proxy开源配置一样。

# 注意
1. 未来当前opencode仓库迭代了，我该怎么升级。

# 我已经做了什么（在分支 cli-extract）

1. **新分支**：已创建 `cli-extract`（基于 `dev`）。
2. **CLI 代码清单与配置/调用点文档**：见 `CLI-CODEMAP.md`。
3. **更容易做“可配置化”的增强**：
   - 将 `OPENCODE_CONFIG` / `OPENCODE_CONFIG_CONTENT` 改为动态读取（避免进程启动时冻结）。
   - 增加全局 CLI 参数：`--config` / `--config-dir` / `--config-content`（等价设置对应 env）。
   - 增加非交互式写入 key 的命令：`opencode auth set <provider> [key]`（也支持 stdin）。

# 未来怎么升级（建议）

优先保持本分支改动为“增量补丁”（新增文档 + 小范围 CLI 参数/命令增强），然后定期把 `dev` rebase/merge 过来：

- `git fetch origin`
- `git rebase origin/dev`（或 `git merge origin/dev`）

# 本地“抽出一个 CLI 工程”（可执行）

本分支提供一个脚本，把 CLI 相关 workspace 抽到一个新目录，形成可独立 `bun install` 的最小工程：

- 生成到默认目录：
  - `bun run ./script/extract-cli.ts`
- 指定输出目录（覆盖输出用 `--force`）：
  - `bun run ./script/extract-cli.ts --out /path/to/opencode-cli --force`