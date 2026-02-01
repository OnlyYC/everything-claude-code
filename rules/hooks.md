# Hook 系统

## Hook 执行流程

```
用户输入 → user-prompt-submit-hook → PreToolUse → 工具执行 → PostToolUse → ... → Stop Hook
                                    ↓ (每个工具调用前)
                                    ↓ 验证/修改参数
                                                         ↓ (每个工具调用后)
                                                         ↓ 自动格式化/检查
```

## Hook 类型

Claude Code 支持在 `~/.claude/settings.json` 中配置 hooks：

- **PreToolUse**：工具执行前（验证、参数修改）
- **PostToolUse**：工具执行后（自动格式化、检查）
- **user-prompt-submit-hook**：用户提交 prompt 后执行
- **Stop**：会话结束时（最终验证）

## JSON 配置转义规则

在 JSON 中配置 hooks 时，请注意以下转义规则：

| 场景 | 正确写法 | 错误写法 |
|------|----------|----------|
| 单行命令 | `"command": "echo hello"` | - |
| 多条命令 | `"command": "cmd1 && cmd2"` | `"command": "cmd1\ncmd2"` |
| 双引号嵌套 | `"command": "echo \"hello\""` | `"command": "echo 'hello'"` |
| 反斜杠 | `"command": "grep \\.java"` | `"command": "grep .java"` |
| 管道符 | `"command": "cmd1 \| cmd2"` | `"command": "cmd1 | cmd2"` |

**重要**：
- JSON 字符串中不能包含实际的换行符（必须用 `\n` 或 `&&`/`;` 连接命令）
- JSON 中的 `\` 需要转义为 `\\`
- JSON 中的 `"` 需要转义为 `\"`

## 跨平台兼容性

**重要**：Hooks 在不同 shell 环境中的兼容性差异

| Shell | 支持状态 | 说明 |
|-------|---------|------|
| **bash** | 完全支持 | 推荐使用，支持 `[[ ]]` 和 `=~` |
| **zsh** | 完全支持 | 大部分兼容 bash 语法 |
| **fish** | 部分支持 | 需要修改语法（不兼容 `[[ ]]`） |
| **cmd/PowerShell** | 不支持 | Windows 下建议使用 WSL 或 Git Bash |

**检测当前 shell 的兼容脚本：**

```json
{
  "PreToolUse": [
    {
      "pattern": "bash",
      "command": "if [ -n \"$BASH_VERSION\" ]; then echo \"Bash 环境，hooks 完全支持\"; else echo \"警告：当前非 Bash 环境，部分 hooks 可能失效\"; fi",
      "description": "Shell 兼容性检测"
    }
  ]
}
```

**JSON 转义说明**：
- 在 JSON 中，多行 bash 脚本必须使用 `\n` 转义符表示换行
- 或使用 `;` 或 `&&` 连接多个命令为单行
- JSON 字符串内不能包含实际的换行符（会破坏 JSON 结构）

## Hook 配置示例

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "pattern": "bash",
        "command": "echo \"$CLI_ARGS\" | grep -qE '^(npm|pnpm|yarn|cargo)( install| build| test)' && echo '建议使用 tmux 运行此长时间命令' || true",
        "description": "长时间命令提醒"
      },
      {
        "pattern": "bash",
        "command": "echo \"$CLI_ARGS\" | grep -qE '^git.*push.*--force' && echo '警告：强制推送可能覆盖远程历史' || true",
        "description": "危险操作警告"
      }
    ],
    "PostToolUse": [
      {
        "pattern": "Edit",
        "command": "echo \"$FILE_PATH\" | grep -qE '\\.(java|kt)$' && mvn spotless:apply -q 2>/dev/null || true",
        "description": "Java 代码自动格式化"
      },
      {
        "pattern": "Edit",
        "command": "echo \"$FILE_PATH\" | grep -qE '\\.(ts|tsx|js|jsx)$' && npx eslint --fix \"$FILE_PATH\" 2>/dev/null || true",
        "description": "TypeScript/JavaScript 自动格式化"
      },
      {
        "pattern": "Edit",
        "command": "grep -q 'console\\.log' \"$FILE_PATH\" 2>/dev/null && echo '警告：文件包含 console.log' || true",
        "description": "检测调试代码"
      }
    ],
    "user-prompt-submit-hook": [
      {
        "command": "echo \"$USER_PROMPT\" | grep -qE '^commit$' && git status --short || true",
        "description": "Commit 前显示状态"
      }
    ],
    "Stop": [
      {
        "command": "git diff --stat 2>/dev/null || true",
        "description": "显示变更统计"
      }
    ]
  }
}
```

## 实用 Hook 模板

### PreToolUse Hooks

```json
{
  "PreToolUse": [
    {
      "pattern": "bash",
      "command": "echo \"$CLI_ARGS\" | grep -qE '(rm.*-rf|del.*(/s|/q))' && echo '警告：递归删除操作' || true",
      "description": "危险删除操作警告"
    },
    {
      "pattern": "bash",
      "command": "CURRENT_BRANCH=$(git branch --show-current 2>/dev/null); echo \"$CLI_ARGS\" | grep -qE '^git.*push' && echo \"$CURRENT_BRANCH\" | grep -qE '^(main|master)$' && echo '警告：直接推送到主分支' || true",
      "description": "主分支推送保护"
    },
    {
      "pattern": "bash",
      "command": "echo \"$CLI_ARGS\" | grep -qE '^npm.*(install|add)' && echo '提示：考虑使用 pnpm 以节省磁盘空间' || true",
      "description": "包管理器建议"
    }
  ]
}
```

### PostToolUse Hooks

```json
{
  "PostToolUse": [
    {
      "pattern": "Write|Edit",
      "command": "echo \"$FILE_PATH\" | grep -qE '\\.py$' && python -m py_compile \"$FILE_PATH\" 2>&1 | head -1 || true",
      "description": "Python 语法检查"
    },
    {
      "pattern": "Write|Edit",
      "command": "echo \"$FILE_PATH\" | grep -qE '\\.go$' && go fmt \"$FILE_PATH\" 2>/dev/null || true",
      "description": "Go 代码格式化"
    },
    {
      "pattern": "Write|Edit",
      "command": "echo \"$FILE_PATH\" | grep -qE '\\.(rs|toml)$' && cargo fmt --quiet 2>/dev/null || true",
      "description": "Rust 代码格式化"
    },
    {
      "pattern": "Write|Edit",
      "command": "echo \"$FILE_PATH\" | grep -qE '\\.(sh|bash)$' && shellcheck \"$FILE_PATH\" 2>&1 | head -5 || true",
      "description": "Shell 脚本检查"
    }
  ]
}
```

### Stop Hooks

```json
{
  "Stop": [
    {
      "command": "git diff --stat 2>/dev/null || echo '非 Git 仓库或无变更'",
      "description": "显示变更统计"
    },
    {
      "command": "grep -rE 'TODO|FIXME|XXX|HACK' --include='*.java' --include='*.kt' --include='*.ts' . 2>/dev/null | wc -l | xargs -I {} echo '未完成 TODO: {}'",
      "description": "统计未完成 TODO"
    },
    {
      "command": "git status --short 2>/dev/null | wc -l | xargs -I {} echo '未暂存文件数: {}'",
      "description": "统计未暂存文件"
    }
  ]
}
```

## Hook 失败处理

### 设计原则

1. **永不阻塞**：使用 `|| true` 确保 hook 失败不影响主流程
2. **静默失败**：使用 `2>/dev/null` 抑制错误输出
3. **快速执行**：单个 hook 执行时间应 < 1 秒
4. **幂等性**：多次执行结果一致

### 错误处理模式

| 模式 | 用途 | 示例 |
|------|------|------|
| **静默失败** | 完全忽略错误 | `command_here 2>/dev/null \|\| true` |
| **单行错误** | 仅显示首次错误 | `command_here 2>&1 \| head -1 \|\| true` |
| **条件警告** | 特定条件失败时提示 | `grep -qE '\.java$' <<< \"$FILE_PATH\" && command_here 2>/dev/null \|\| echo '格式化失败'` |

**JSON 配置示例**：

```json
{
  "PostToolUse": [
    {
      "pattern": "Edit",
      "command": "command_here 2>/dev/null || true",
      "description": "静默失败模式"
    },
    {
      "pattern": "Edit",
      "command": "command_here 2>&1 | head -1 || true",
      "description": "单行错误模式"
    },
    {
      "pattern": "Edit",
      "command": "grep -qE '\\.java$' <<< \"$FILE_PATH\" && command_here 2>/dev/null || echo '格式化失败'",
      "description": "条件警告模式"
    }
  ]
}
```

## 调试 Hooks

### 查看当前配置

```bash
# 查看 Claude Code settings
cat ~/.claude/settings.json | jq '.hooks'
```

### 测试单个 Hook

```bash
# 模拟环境变量测试
export CLI_ARGS="git push --force"
export FILE_PATH="src/main/java/UserService.java"
export USER_PROMPT="commit"

# 运行 hook 命令
your_hook_command_here
```

### 启用调试输出

```json
{
  "PostToolUse": [
    {
      "pattern": "Edit",
      "command": "echo \"[Hook Debug] File: $FILE_PATH\" >&2; command_here 2>/dev/null || true",
      "description": "带调试输出的 hook"
    }
  ]
}
```

## 自动接受权限

谨慎使用 `allowedPrompts`（在计划模式中使用）：

- **适用场景**：结构清晰、风险可控的代码变更
- **不适用**：探索性工作、破坏性操作、敏感数据处理
- **禁止使用**：`dangerouslySkipPermissions` 参数

## 任务跟踪工具

使用任务工具追踪多步骤工作：

| 工具 | 用途 |
|------|------|
| **TaskCreate** | 创建新任务 |
| **TaskUpdate** | 更新任务状态（pending/in_progress/completed） |
| **TaskGet** | 获取任务详情 |
| **TaskList** | 列出所有任务 |

**使用场景：**
- 追踪多步骤任务的进度
- 验证对需求的理解
- 支持即时调整
- 显示细粒度实现步骤

**任务依赖：**
- `addBlockedBy`：设置前置依赖（必须等待的任务）
- `addBlocks`：设置后续依赖（等待当前任务的任务）

**最佳实践：**
1. 开始任务时立即设为 `in_progress`
2. 完成后立即设为 `completed`
3. 失败时保持 `in_progress` 并创建修复任务
4. 取消时设为 `deleted`

## 最佳实践总结

| 原则 | 说明 |
|------|------|
| **轻量执行** | 单个 hook 执行时间应 < 1 秒 |
| **永不阻塞** | 始终使用 `|| true` 或 `2>/dev/null` |
| **跨平台兼容** | 优先使用 POSIX 兼容语法 |
| **幂等设计** | 多次执行结果一致 |
| **明确输出** | 错误信息输出到 stderr (`>&2`) |
| **任务粒度适中** | 单个任务应在 30 分钟内可完成 |
| **依赖关系清晰** | 使用 `addBlockedBy` 确保正确执行顺序 |
