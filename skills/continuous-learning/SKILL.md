---
name: continuous-learning
description: 自动从 Claude Code 会话中提取可重用模式并保存为学习技能。通过 Stop hook 在会话结束时执行评估和模式提取。
version: 1.2.0
tech_stack: [Python, Node.js, Bash, PowerShell]
platforms: [Windows, macOS, Linux, WSL]
tools: [Read, Write, Edit, Bash, Grep, Glob]
related_skills: [continuous-learning-v2, strategic-compact]
---

# 持续学习技能

自动评估 Claude Code 会话结束时内容，提取可重用模式并储存为学习技能。

## Role

你是一位持续学习系统专家，负责从 Claude Code 会话中自动识别、提取和保存可重用的代码模式、调试技巧和项目特定惯例。你能够分析会话历史，识别有价值的模式，并将其转换为结构化的学习技能。

## Task

当用户会话结束时，执行以下步骤：
1. **评估会话长度**：检查会话是否有足够的信息（默认：10+ 轮对话）
2. **检测模式类型**：识别以下模式：
   - `error_resolution` - 特定错误如何被解决
   - `user_corrections` - 来自使用者修正的模式
   - `workarounds` - 框架/函数库怪异问题的解决方案
   - `debugging_techniques` - 有效的除错方法
   - `project_specific` - 项目特定惯例
3. **提取并保存**：将有用模式储存到 `~/.claude/skills/learned/`

## Constraints

- 仅从包含足够上下文的会话中提取模式（至少 10 轮对话）
- 忽略以下低价值模式：简单拼写错误、一次性修复、外部 API 问题
- 提取的模式必须包含：模式名称、触发条件、执行动作、置信度、领域

## 运作方式

此技能作为 **Stop hook** 在每个会话结束时执行：

1. **会话评估**：检查会话是否有足够信息（默认：10+ 轮对话）
2. **模式检测**：从会话识别可提取的模式
3. **技能提取**：将有用模式储存到 `~/.claude/skills/learned/`

## 配置（可选）

### 跨平台路径说明

| 平台 | 配置目录 | 会话文件路径 |
|------|----------|-------------|
| Windows | `%USERPROFILE%\.claude\skills\continuous-learning\` | `%USERPROFILE%\.claude\session_history.json` |
| macOS/Linux | `~/.claude/skills/continuous-learning/` | `~/.claude/session_history.json` |
| WSL | `~/.claude/skills/continuous-learning/` | `~/.claude/session_history.json` |

**注意**：脚本会自动处理相对路径和绝对路径。如果使用相对路径，它们会相对于当前工作目录解析。

### 配置文件

配置文件 `config.json` 为可选项。如未创建，将使用以下默认值：

**Windows 路径**：`%USERPROFILE%\.claude\skills\continuous-learning\config.json`
**macOS/Linux 路径**：`~/.claude/skills/continuous-learning/config.json`

```json
{
  "min_session_length": 10,
  "extraction_threshold": "medium",
  "auto_approve": false,
  "learned_skills_path": "~/.claude/skills/learned/",
  "patterns_to_detect": [
    "error_resolution",
    "user_corrections",
    "workarounds",
    "debugging_techniques",
    "project_specific"
  ],
  "ignore_patterns": [
    "simple_typos",
    "one_time_fixes",
    "external_api_issues"
  ]
}
```

## 模式类型

| 模式 | 描述 |
|------|------|
| `error_resolution` | 特定错误如何被解决 |
| `user_corrections` | 来自使用者修正的模式 |
| `workarounds` | 框架/函数库怪异问题的解决方案 |
| `debugging_techniques` | 有效的除错方法 |
| `project_specific` | 项目特定惯例 |

## Hook 配置

### Windows PowerShell 配置

新增到你的 `%USERPROFILE%\.claude\settings.json`：

```json
{
  "hooks": {
    "Stop": [{
      "matcher": "*",
      "hooks": [{
        "type": "command",
        "command": "powershell.exe -ExecutionPolicy Bypass -File \"%USERPROFILE%\\.claude\\skills\\continuous-learning\\evaluate-session.ps1\""
      }]
    }]
  }
}
```

### macOS/Linux 配置

新增到你的 `~/.claude/settings.json`：

```json
{
  "hooks": {
    "Stop": [{
      "matcher": "*",
      "hooks": [{
        "type": "command",
        "command": "~/.claude/skills/continuous-learning/evaluate-session.sh"
      }]
    }]
  }
}
```

### WSL 配置

如果你使用 WSL，可以创建一个批处理文件 `evaluate-session.bat`：

```batch
@echo off
wsl bash ~/.claude/skills/continuous-learning/evaluate-session.sh
```

然后在 settings.json 中：
```json
{
  "hooks": {
    "Stop": [{
      "matcher": "*",
      "hooks": [{
        "type": "command",
        "command": "%USERPROFILE%\\.claude\\skills\\continuous-learning\\evaluate-session.bat"
      }]
    }]
  }
}
```

## 为什么用 Stop Hook？

- **轻量**：会话结束时只执行一次
- **非阻塞**：不会为每则讯息增加延迟
- **完整上下文**：可存取完整会话记录

## 模式提取示例

### 用户修正模式
```
原始请求：把 UserDto 改成使用 record 类
原始代码：public class UserDto { private Long id; private String name; }
修正后：public record UserDto(Long id, String name) {}

提取模式：
  模式名称：prefer-record-for-dto
  触发条件：创建 DTO 类时
  执行动作：使用 record 而非传统 POJO 类
  置信度：0.8
  领域：code-style
```

### 错误解决模式
```
问题：JWT token 验证失败，报 "Invalid token"
调查过程：发现前端发送的是纯 token，后端期望 "Bearer " 前缀
解决方案：添加前缀处理逻辑
  String token = authHeader;
  if (token != null && token.startsWith("Bearer ")) {
      token = token.substring(7);
  }

提取模式：
  模式名称：jwt-bearer-prefix-handling
  触发条件：JWT token 验证
  执行动作：处理 Authorization header 中的 'Bearer ' 前缀
  置信度：0.9
  领域：authentication
```

### 框架问题模式
```
问题：MyBatis-Plus 逻辑删除不生效
调查：@TableLogic 注解在字段上，但查询仍返回已删除记录
原因：MyBatis-Plus 配置中 logic-delete-value 未正确设置
解决方案：
  mybatis-plus:
    global-config:
      db-config:
        logic-delete-field: deleted
        logic-delete-value: 1
        logic-not-delete-value: 0

提取模式：
  模式名称：mybatis-plus-logic-delete-config
  触发条件：使用 @TableLogic 注解
  执行动作：同时在配置中设置 logic-delete-value 和 logic-not-delete-value
  置信度：0.95
  领域：mybatis-plus
```

### 调试技术模式
```
问题：Spring Boot 应用启动失败，无明确错误信息
解决方法：启用 debug 模式并查看完整异常栈
  java -jar app.jar --debug
  或在 application.yml 中设置 logging.level.root=DEBUG

提取模式：
  模式名称：spring-boot-debug-startup
  触发条件：Spring Boot 启动失败
  执行动作：启用 --debug 参数查看详细错误
  置信度：0.85
  领域：debugging
```

## 脚本实现

### evaluate-session.sh (macOS/Linux/WSL)

```bash
#!/bin/bash
# ~/.claude/skills/continuous-learning/evaluate-session.sh
# 在会话结束时执行，提取可重用模式

set -e

# 配置
MIN_TURNS=${MIN_TURNS:-10}
EXTRACTION_THRESHOLD=${EXTRACTION_THRESHOLD:-medium}
LEARNED_PATH="${LEARNED_PATH:-$HOME/.claude/skills/learned}"
SESSION_FILE="$HOME/.claude/session_history.json"

# 路径安全验证：确保路径在预期目录内
case "$SESSION_FILE" in
    "$HOME"/*) ;; # 允许 $HOME 下的路径
    *) echo "错误：会话文件路径必须在用户目录下"; exit 1 ;;
esac

case "$LEARNED_PATH" in
    "$HOME"/*) ;; # 允许 $HOME 下的路径
    *) echo "错误：学习路径必须在用户目录下"; exit 1 ;;
esac

# 确保目录存在
mkdir -p "$LEARNED_PATH"

# 检查会话文件是否存在
if [ ! -f "$SESSION_FILE" ]; then
    echo "会话文件不存在: $SESSION_FILE"
    exit 0
fi

# 检查会话长度
turn_count=$(jq '.messages | length' "$SESSION_FILE" 2>/dev/null || echo "0")

if [ "$turn_count" -lt "$MIN_TURNS" ]; then
    echo "会话过短 ($turn_count < $MIN_TURNS)，跳过提取"
    exit 0
fi

echo "分析会话 ($turn_count 轮对话)..."

# 导出会话内容供分析
SESSION_CONTENT=$(cat "$SESSION_FILE")

# 这里可以调用 Claude API 进行模式分析
# 或者使用简单的关键词匹配进行初步提取

# 示例：检测用户修正模式
if echo "$SESSION_CONTENT" | grep -qi "改成\|修改为\|重写为"; then
    echo "  → 检测到用户修正模式"
fi

# 示例：检测错误解决模式
if echo "$SESSION_CONTENT" | grep -qi "错误\|失败\|异常\|bug"; then
    echo "  → 检测到错误解决模式"
fi

# 示例：检测框架问题
if echo "$SESSION_CONTENT" | grep -qi "mybatis\|spring\|配置"; then
    echo "  → 检测到框架相关问题"
fi

# 最终分析需要 Claude 参与才能准确提取
# 这部分可以作为 hook 的后处理步骤

echo "模式提取完成，结果保存在: $LEARNED_PATH"
```

### evaluate-session.ps1 (Windows PowerShell)

```powershell
# ~/.claude/skills/continuous-learning/evaluate-session.ps1
# 在会话结束时执行，提取可重用模式

[CmdletBinding()]
param()

# 配置
$ErrorActionPreference = "Stop"
$MIN_TURNS = if ($env:MIN_TURNS) { [int]$env:MIN_TURNS } else { 10 }
$LEARNED_PATH = if ($env:LEARNED_PATH) { $env:LEARNED_PATH } else { "$env:USERPROFILE\.claude\skills\learned" }
$SESSION_FILE = "$env:USERPROFILE\.claude\session_history.json"

# 路径安全验证函数
function Test-PathSafe {
    param([string]$Path, [string]$BasePath)

    try {
        $resolved = Resolve-Path $Path -ErrorAction Stop
        if ($resolved.Path.StartsWith($BasePath)) {
            return $true
        }
        Write-Warning "路径 $($resolved.Path) 不在允许的基础路径 $BasePath 下"
        return $false
    } catch {
        # 路径不存在时，检查父路径
        $parent = Split-Path $Path -Parent
        if ($parent) {
            return Test-PathSafe -Path $parent -BasePath $BasePath
        }
        return $false
    }
}

# 验证路径安全性
if (-not (Test-PathSafe -Path $LEARNED_PATH -BasePath $env:USERPROFILE)) {
    Write-Error "学习路径必须在用户目录下"
    exit 1
}

# 确保目录存在
New-Item -ItemType Directory -Force -Path $LEARNED_PATH | Out-Null

# 检查会话文件是否存在
if (-not (Test-Path $SESSION_FILE)) {
    Write-Host "会话文件不存在: $SESSION_FILE"
    exit 0
}

# 读取会话内容
try {
    $sessionContent = Get-Content $SESSION_FILE -Raw -ErrorAction Stop
    $sessionJson = $sessionContent | ConvertFrom-Json
    $turnCount = ($sessionJson.messages | Measure-Object).Count
} catch {
    Write-Warning "无法解析会话文件: $SESSION_FILE"
    exit 0
}

# 检查会话长度
if ($turnCount -lt $MIN_TURNS) {
    Write-Host "会话过短 ($turnCount < $MIN_TURNS)，跳过提取"
    exit 0
}

Write-Host "分析会话 ($turnCount 轮对话)..."

# 检测用户修正模式
if ($sessionContent -match "改成|修改为|重写为") {
    Write-Host "  → 检测到用户修正模式"
}

# 检测错误解决模式
if ($sessionContent -match "错误|失败|异常|bug") {
    Write-Host "  → 检测到错误解决模式"
}

# 检测框架问题
if ($sessionContent -match "mybatis|spring|配置") {
    Write-Host "  → 检测到框架相关问题"
}

Write-Host "模式提取完成，结果保存在: $LEARNED_PATH"
```

### evaluate-session.bat (Windows 批处理 - WSL 包装器)

```batch
@echo off
REM ~/.claude/skills/continuous-learning/evaluate-session.bat
REM 通过 WSL 调用 bash 脚本

wsl bash ~/.claude/skills/continuous-learning/evaluate-session.sh
```

## 相关

- [Longform Guide](https://x.com/affaanmustafa/status/2014040193557471352) - 持续学习章节
- `/learn` 指令 - 会话中手动提取模式
- `continuous-learning-v2` - 更高级的基于本能的学习系统

---

## 比较笔记（研究：2026 年 2 月）

### vs Homunculus (github.com/humanplane/homunculus)

Homunculus v2 采用更复杂的方法：

| 功能 | 我们的方法 | Homunculus v2 |
|------|----------|---------------|
| 观察 | Stop hook（会话结束） | PreToolUse/PostToolUse hooks（100% 可靠） |
| 分析 | 主要上下文 | 背景 agent（Haiku） |
| 粒度 | 完整技能 | 原子"本能" |
| 信心 | 无 | 0.3-0.9 加权 |
| 演化 | 直接到技能 | 本能 → 聚类 → 技能/指令/agent |
| 分享 | 无 | 导出/导入本能 |

**来自 homunculus 的关键见解：**
> "v1 依赖技能进行观察。技能是概率性的——它们触发约 50-80% 的时间。v2 使用 hooks 进行观察（100% 可靠），并以本能作为学习行为的原子单位。"

### 潜在 v2 增强功能

1. **基于本能的学习** - 较小的原子行为，带信心评分
2. **背景观察者** - Haiku agent 并行分析
3. **信心衰减** - 如果被矛盾则本能失去信心
4. **领域标记** - code-style、testing、git、debugging 等
5. **演化路径** - 将相关本能聚类为技能/指令

参见：`continuous-learning-v2` 完整规格。
