---
name: continuous-learning-v2
description: 基于本能的学习系统，通过 hooks 观察会话，创建带信心评分的原子本能，并演化它们为技能/指令/agent。
version: 2.1.0
tech_stack: [Python, Node.js, Bash, PowerShell]
platforms: [Windows, macOS, Linux, WSL]
tools: [Read, Write, Edit, Bash, Grep, Glob]
related_skills: [continuous-learning, strategic-compact]
---

# 持续学习 v2 - 基于本能的架构

基于本能的学习系统，通过 hooks 观察会话，创建带信心评分的原子本能，并演化它们为技能/指令/agent。

## 与 v1 的主要区别

| 功能 | v1 | v2 |
|------|----|----|
| 观察 | Stop hook（会话结束） | PreToolUse/PostToolUse（100% 可靠） |
| 分析 | 主要上下文 | 背景 agent（Haiku） |
| 粒度 | 完整技能 | 原子"本能" |
| 信心 | 无 | 0.3-0.9 加权 |
| 演化 | 直接到技能 | 本能 → 聚类 → 技能/指令/agent |
| 分享 | 无 | 导出/导入本能 |
| 跨平台 | Unix shell only | Windows/macOS/Linux/WSL |

## 本能模型

本能是一个小型学习行为：

```yaml
---
id: prefer-functional-style
trigger: "when writing new functions"
confidence: 0.7
domain: "code-style"
source: "session-observation"
---

# 偏好函数风格

## 动作
适当时间使用函数模式而非类。

## 证据
- 观察到 5 次函数模式偏好
- 使用者在 2025-01-15 将基于类的方法修正为函数
```

**属性：**
- **原子性** — 一个触发器，一个动作
- **信心加权** — 0.3 = 试探性，0.9 = 近乎确定
- **领域标记** — code-style、testing、git、debugging、workflow 等
- **证据支持** — 追踪建立它的观察

## 运作方式

```
会话活动
      │
      │ Hooks 捕获提示 + 工具使用（100% 可靠）
      ▼
┌─────────────────────────────────────────┐
│         observations.jsonl              │
│   （提示、工具呼叫、结果）               │
└─────────────────────────────────────────┘
      │
      │ Observer agent 读取（背景、Haiku）
      ▼
┌─────────────────────────────────────────┐
│          模式检测                        │
│   • 使用者修正 → 本能                   │
│   • 错误解决 → 本能                     │
│   • 重复工作流程 → 本能                 │
└─────────────────────────────────────────┘
      │
      │ 建立/更新
      ▼
┌─────────────────────────────────────────┐
│         instincts/personal/             │
│   • prefer-functional.md (0.7)          │
│   • always-test-first.md (0.9)          │
│   • use-zod-validation.md (0.6)         │
└─────────────────────────────────────────┘
      │
      │ /evolve 聚类
      ▼
┌─────────────────────────────────────────┐
│              evolved/                   │
│   • commands/new-feature.md             │
│   • skills/testing-workflow.md          │
│   • agents/refactor-specialist.md       │
└─────────────────────────────────────────┘
```

## 快速开始

### 1. 启用观察 Hooks

#### Windows PowerShell 配置

新增到你的 `%USERPROFILE%\.claude\settings.json`：

```json
{
  "hooks": {
    "PreToolUse": [{
      "matcher": "*",
      "hooks": [{
        "type": "command",
        "command": "powershell.exe -ExecutionPolicy Bypass -File \"%USERPROFILE%\\.claude\\skills\\continuous-learning-v2\\hooks\\observe.ps1\" pre"
      }]
    }],
    "PostToolUse": [{
      "matcher": "*",
      "hooks": [{
        "type": "command",
        "command": "powershell.exe -ExecutionPolicy Bypass -File \"%USERPROFILE%\\.claude\\skills\\continuous-learning-v2\\hooks\\observe.ps1\" post"
      }]
    }]
  }
}
```

#### macOS/Linux 配置

新增到你的 `~/.claude/settings.json`：

```json
{
  "hooks": {
    "PreToolUse": [{
      "matcher": "*",
      "hooks": [{
        "type": "command",
        "command": "~/.claude/skills/continuous-learning-v2/hooks/observe.sh pre"
      }]
    }],
    "PostToolUse": [{
      "matcher": "*",
      "hooks": [{
        "type": "command",
        "command": "~/.claude/skills/continuous-learning-v2/hooks/observe.sh post"
      }]
    }]
  }
}
```

#### WSL 配置

如果你使用 WSL，创建批处理包装器 `observe.bat`：

```batch
@echo off
wsl bash ~/.claude/skills/continuous-learning-v2/hooks/observe.sh %1
```

然后在 settings.json 中：
```json
{
  "hooks": {
    "PreToolUse": [{
      "matcher": "*",
      "hooks": [{
        "type": "command",
        "command": "%USERPROFILE%\\.claude\\skills\\continuous-learning-v2\\hooks\\observe.bat pre"
      }]
    }],
    "PostToolUse": [{
      "matcher": "*",
      "hooks": [{
        "type": "command",
        "command": "%USERPROFILE%\\.claude\\skills\\continuous-learning-v2\\hooks\\observe.bat post"
      }]
    }]
  }
}
```

### 2. 初始化目录结构

#### Windows PowerShell

```powershell
# 创建目录结构
New-Item -ItemType Directory -Force -Path "$env:USERPROFILE\.claude\homunculus\instincts\personal"
New-Item -ItemType Directory -Force -Path "$env:USERPROFILE\.claude\homunculus\instincts\inherited"
New-Item -ItemType Directory -Force -Path "$env:USERPROFILE\.claude\homunculus\evolved\agents"
New-Item -ItemType Directory -Force -Path "$env:USERPROFILE\.claude\homunculus\evolved\skills"
New-Item -ItemType Directory -Force -Path "$env:USERPROFILE\.claude\homunculus\evolved\commands"
New-Item -ItemType File -Force -Path "$env:USERPROFILE\.claude\homunculus\observations.jsonl"
```

#### macOS/Linux/WSL

```bash
# 创建目录结构
mkdir -p ~/.claude/homunculus/{instincts/{personal,inherited},evolved/{agents,skills,commands}}
touch ~/.claude/homunculus/observations.jsonl
```

### 3. 执行 Observer Agent（可选）

观察者可以在背景执行并分析观察：

**Windows PowerShell:**
```powershell
# 启动背景观察者
Start-Process -FilePath "pwsh" -ArgumentList "-File", "$env:USERPROFILE\.claude\skills\continuous-learning-v2\agents\start-observer.ps1" -WindowStyle Hidden
```

**macOS/Linux/WSL:**
```bash
# 启动背景观察者
nohup ~/.claude/skills/continuous-learning-v2/agents/start-observer.sh > /dev/null 2>&1 &
```

## 指令

| 指令 | 描述 |
|------|------|
| `/instinct-status` | 显示所有学习本能及其信心 |
| `/evolve` | 将相关本能聚类为技能/指令 |
| `/instinct-export` | 导出本能以分享 |
| `/instinct-import <file>` | 从他人导入本能 |

## 配置

编辑配置文件：

### 配置文件路径

| 平台 | 配置文件路径 |
|------|-------------|
| Windows | `%USERPROFILE%\.claude\homunculus\config.json` |
| macOS | `~/.claude/homunculus/config.json` |
| Linux | `~/.claude/homunculus/config.json` |
| WSL | `~/.claude/homunculus/config.json` |

**注意**：Windows 路径使用反斜杠 `\` 或正斜杠 `/` 均可。PowerShell 和大多数现代工具能正确处理两种格式。

```json
{
  "version": "2.0",
  "platform": "auto",
  "observation": {
    "enabled": true,
    "store_path": "observations.jsonl",
    "max_file_size_mb": 10,
    "archive_after_days": 7
  },
  "instincts": {
    "personal_path": "instincts/personal/",
    "inherited_path": "instincts/inherited/",
    "min_confidence": 0.3,
    "auto_approve_threshold": 0.7,
    "confidence_decay_rate": 0.05
  },
  "observer": {
    "enabled": true,
    "model": "haiku",
    "run_interval_minutes": 5,
    "patterns_to_detect": [
      "user_corrections",
      "error_resolutions",
      "repeated_workflows",
      "tool_preferences"
    ]
  },
  "evolution": {
    "cluster_threshold": 3,
    "evolved_path": "evolved/"
  }
}
```

## 文件结构

### Windows 路径

```
%USERPROFILE%\.claude\homunculus\
├── identity.json           # 你的个人资料、技术水平
├── observations.jsonl      # 当时会话观察
├── observations.archive\   # 已处理观察
├── instincts\
│   ├── personal\           # 自动学习本能
│   └── inherited\          # 从他人导入
└── evolved\
    ├── agents\             # 产生的专业 agents
    ├── skills\             # 产生的技能
    └── commands\           # 产生的指令
```

### macOS/Linux/WSL 路径

```
~/.claude/homunculus/
├── identity.json           # 你的个人资料、技术水平
├── observations.jsonl      # 当时会话观察
├── observations.archive/   # 已处理观察
├── instincts/
│   ├── personal/           # 自动学习本能
│   └── inherited/          # 从他人导入
└── evolved/
    ├── agents/             # 产生的专业 agents
    ├── skills/             # 产生的技能
    └── commands/           # 产生的指令
```

## 与 Skill Creator 整合

当你使用 [Skill Creator GitHub App](https://skill-creator.app) 时，它现在产生**两者**：
- 传统 SKILL.md 文件（用于向后相容）
- 本能集合（用于 v2 学习系统）

从仓库分析的本能有 `source: "repo-analysis"` 并包含来源仓库 URL。

## 验证 Hook 配置

### 验证 Hook 是否生效

执行任意工具调用后，检查观察文件是否被创建：

```bash
# Windows PowerShell
Get-ChildItem "$env:USERPROFILE\.claude\homunculus\observations.jsonl"

# macOS/Linux/WSL
ls -la ~/.claude/homunculus/observations.jsonl
```

### 测试 Hook 执行

创建测试脚本验证 hook 是否被正确调用：

**Windows (test-hook.ps1):**
```powershell
$env:TOOL_CALL_COUNT = 60
powershell.exe -ExecutionPolicy Bypass -File "$env:USERPROFILE\.claude\skills\continuous-learning-v2\hooks\observe.ps1" post
# 应该显示策略性压缩提醒
```

**macOS/Linux/WSL (test-hook.sh):**
```bash
export TOOL_CALL_COUNT=60
~/.claude/skills/continuous-learning-v2/hooks/observe.sh post
# 应该显示策略性压缩提醒
```

### 调试 Hook 问题

如果 hook 没有执行，检查：

1. **settings.json 语法是否正确**
```bash
# Windows PowerShell
Get-Content "$env:USERPROFILE\.claude\settings.json" | ConvertFrom-Json

# macOS/Linux
cat ~/.claude/settings.json | jq .
```

2. **脚本路径是否正确**
```bash
# Windows
Test-Path "$env:USERPROFILE\.claude\skills\continuous-learning-v2\hooks\observe.ps1"

# macOS/Linux
test -f ~/.claude/skills/continuous-learning-v2/hooks/observe.sh
```

3. **脚本是否有执行权限 (macOS/Linux)**
```bash
chmod +x ~/.claude/skills/continuous-learning-v2/hooks/observe.sh
```

## 信心评分

信心随时间演化：

| 分数 | 意义 | 行为 |
|------|------|------|
| 0.3 | 试探性 | 建议但不强制 |
| 0.5 | 中等 | 相关时应用 |
| 0.7 | 强烈 | 自动批准应用 |
| 0.9 | 近乎确定 | 核心行为 |

**信心增加**当：
- 重复观察到模式
- 使用者不修正建议行为
- 来自其他来源的类似本能同意

**信心减少**当：
- 使用者明确修正行为
- 长期未观察到模式
- 出现矛盾证据

## 为何 Hooks vs Skills 用于观察？

> "v1 依赖技能进行观察。技能是概率性的——它们根据 Claude 的判断触发约 50-80% 的时间。"

Hooks **100% 的时间**确定性触发。这意味着：
- 每个工具呼叫都被观察
- 无模式被遗漏
- 学习是全面的

## 向后相容性

v2 完全相容 v1：
- 现有 `~/.claude/skills/learned/` 技能仍可运作
- Stop hook 仍执行（但现在也馈入 v2）
- 渐进迁移路径：两者并行执行

## 隐私与安全

- 观察数据保持在你的机器**本地**
- 只有**本能**（抽象模式）可被导出，不包含具体代码或对话内容
- 不会分享实际代码或对话内容
- 你完全控制导出内容
- 导出的本能文件经过净化，不包含敏感信息

### 数据存储位置

| 平台 | 存储路径 |
|------|----------|
| Windows | `%USERPROFILE%\.claude\homunculus\` |
| macOS | `~/.claude/homunculus/` |
| Linux | `~/.claude/homunculus/` |
| WSL | `~/.claude/homunculus/` |

## 相关技能

- `continuous-learning` - v1 持续学习系统（基于 Stop hook）
- [Skill Creator](https://skill-creator.app) - 从仓库历史产生本能
- [Homunculus](https://github.com/humanplane/homunculus) - v2 架构灵感
- [Longform Guide](https://x.com/affaanmustafa/status/2014040193557471352) - 持续学习章节

---

*基于本能的学习：一次一个观察，教导 Claude 你的模式。*

## 跨平台支持说明

v2 现已完全支持跨平台使用：

### Windows 原生支持
- 使用 PowerShell 脚本
- 配置路径使用 `%USERPROFILE%`
- 支持 Windows 路径分隔符（`\`）

### WSL 支持
- 可以继续使用 bash 脚本
- 通过批处理文件作为桥梁
- 配置路径自动映射

### macOS/Linux 原生支持
- 使用 bash/shell 脚本
- 配置路径使用 `~`
- 标准 Unix 路径分隔符（`/`）
