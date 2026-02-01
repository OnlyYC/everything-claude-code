---
name: hooks
description: Hook 系统配置指南
priority: optional
tags: [hooks, pretooluse, posttooluse, automation]
---

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

| Hook 类型 | 触发时机 | 用途 |
|-----------|----------|------|
| **PreToolUse** | 工具执行前 | 验证、参数修改、危险操作警告 |
| **PostToolUse** | 工具执行后 | 自动格式化、检查 |
| **user-prompt-submit-hook** | 用户提交 prompt 后 | 预处理、上下文准备 |
| **Stop** | 会话结束时 | 最终验证、清理 |

## JSON 配置要点

### 基本格式

```json
{
  "hooks": {
    "PreToolUse": [
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
      }
    ]
  }
}
```

### JSON 转义规则

| 场景 | 正确写法 | 错误写法 |
|------|----------|----------|
| 单行命令 | `"command": "echo hello"` | - |
| 多条命令 | `"command": "cmd1 && cmd2"` | `"command": "cmd1\ncmd2"` |
| 双引号嵌套 | `"command": "echo \"hello\""` | `"command": "echo 'hello'"` |
| 反斜杠 | `"command": "grep \\.java"` | `"command": "grep .java"` |
| 管道符 | `"command": "cmd1 \\| cmd2"` | `"command": "cmd1 | cmd2"` |

**重要**：
- JSON 字符串中不能包含实际的换行符（必须用 `\n` 或 `&&`/`;` 连接命令）
- JSON 中的 `\` 需要转义为 `\\`
- JSON 中的 `"` 需要转义为 `\"`

## 跨平台兼容性

| Shell | 支持状态 | 说明 |
|-------|---------|------|
| **bash** | 完全支持 | 推荐使用 |
| **zsh** | 完全支持 | 大部分兼容 bash 语法 |
| **fish** | 部分支持 | 需要修改语法 |
| **cmd/PowerShell** | 不支持 | Windows 下建议使用 WSL 或 Git Bash |

## Hook 配置示例

### PreToolUse Hooks

```json
{
  "PreToolUse": [
    {
      "pattern": "bash",
      "command": "echo \"$CLI_ARGS\" | grep -qE '^(npm|pnpm|yarn|cargo)( install|build|test)' && echo '建议使用 tmux 运行此长时间命令' || true",
      "description": "长时间命令提醒"
    },
    {
      "pattern": "bash",
      "command": "CURRENT_BRANCH=$(git branch --show-current 2>/dev/null); echo \"$CLI_ARGS\" | grep -qE '^git.*push' && echo \"$CURRENT_BRANCH\" | grep -qE '^(main|master)$' && echo '警告：直接推送到主分支' || true",
      "description": "主分支推送保护"
    }
  ]
}
```

### PostToolUse Hooks

```json
{
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
  ]
}
```

### Stop Hooks

```json
{
  "Stop": [
    {
      "command": "git diff --stat 2>/dev/null || true",
      "description": "显示变更统计"
    },
    {
      "command": "grep -rE 'TODO|FIXME|XXX|HACK' --include='*.java' . 2>/dev/null | wc -l | xargs -I {} echo '未完成 TODO: {}'",
      "description": "统计未完成 TODO"
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
| **静默失败** | 完全忽略错误 | `command 2>/dev/null \\|\\| true` |
| **单行错误** | 仅显示首次错误 | `command 2>&1 \\| head -1 \\|\\| true` |
| **条件警告** | 特定条件失败时提示 | 条件失败时显示警告 |

## 实用 Hook 模板

### PreToolUse：危险操作保护

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
      "command": "echo \"$CLI_ARGS\" | grep -qE '^npm.*(install|add)' && echo '提示：考虑使用 pnpm 以节省磁盘空间' || true",
      "description": "包管理器建议"
    }
  ]
}
```

### PostToolUse：代码格式化

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
    }
  ]
}
```

## 调试 Hooks

### 查看当前配置

```bash
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

## 最佳实践总结

| 原则 | 说明 |
|------|------|
| **轻量执行** | 单个 hook 执行时间应 < 1 秒 |
| **永不阻塞** | 始终使用 `|| true` 或 `2>/dev/null` |
| **跨平台兼容** | 优先使用 POSIX 兼容语法 |
| **幂等设计** | 多次执行结果一致 |
| **明确输出** | 错误信息输出到 stderr (`>&2`) |
