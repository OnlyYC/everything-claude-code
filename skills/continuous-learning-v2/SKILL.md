---
name: continuous-learning-v2
description: Instinct-based learning system that observes sessions via hooks, creates atomic instincts with confidence scoring, and evolves them into skills/commands/agents.
version: 2.0.0
---

# 持续学习 v2 - 基于本能的架构

进阶学习系统，通过原子"本能"（带信心评分的小型学习行为）将您的 Claude Code 会话转化为可重用知识。

## v2 的新功能

| 功能 | v1 | v2 |
|------|----|----|
| 观察 | Stop hook（会话结束） | PreToolUse/PostToolUse（100% 可靠） |
| 分析 | 主要上下文 | 背景 agent（Haiku） |
| 粒度 | 完整技能 | 原子"本能" |
| 信心 | 无 | 0.3-0.9 加权 |
| 演化 | 直接到技能 | 本能 → 聚类 → 技能/指令/agent |
| 分享 | 无 | 导出/导入本能 |

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

### 2. 初始化目录结构

```bash
mkdir -p ~/.claude/homunculus/{instincts/{personal,inherited},evolved/{agents,skills,commands}}
touch ~/.claude/homunculus/observations.jsonl
```

### 3. 执行 Observer Agent（可选）

观察者可以在背景执行并分析观察：

```bash
# 启动背景观察者
~/.claude/skills/continuous-learning-v2/agents/start-observer.sh
```

## 指令

| 指令 | 描述 |
|------|------|
| `/instinct-status` | 显示所有学习本能及其信心 |
| `/evolve` | 将相关本能聚类为技能/指令 |
| `/instinct-export` | 导出本能以分享 |
| `/instinct-import <file>` | 从他人导入本能 |

## 设定

编辑 `config.json`：

```json
{
  "version": "2.0",
  "observation": {
    "enabled": true,
    "store_path": "~/.claude/homunculus/observations.jsonl",
    "max_file_size_mb": 10,
    "archive_after_days": 7
  },
  "instincts": {
    "personal_path": "~/.claude/homunculus/instincts/personal/",
    "inherited_path": "~/.claude/homunculus/instincts/inherited/",
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
    "evolved_path": "~/.claude/homunculus/evolved/"
  }
}
```

## 文件结构

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

## 隐私

- 观察保持在你的机器**本地**
- 只有**本能**（模式）可被导出
- 不会分享实际代码或对话内容
- 你控制导出内容

## 相关

- [Skill Creator](https://skill-creator.app) - 从仓库历史产生本能
- [Homunculus](https://github.com/humanplane/homunculus) - v2 架构灵感
- [Longform Guide](https://x.com/affaanmustafa/status/2014040193557471352) - 持续学习章节

---

*基于本能的学习：一次一个观察，教导 Claude 你的模式。*
