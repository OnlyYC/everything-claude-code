---
name: continuous-learning
description: Automatically extract reusable patterns from Claude Code sessions and save them as learned skills for future use.
---

# 持续学习技能

自动评估 Claude Code 会话结束时内容，提取可重用模式并储存为学习技能。

## 运作方式

此技能作为 **Stop hook** 在每个会话结束时执行：

1. **会话评估**：检查会话是否有足够讯息（默认：10+ 则）
2. **模式检测**：从会话识别可提取的模式
3. **技能提取**：将有用模式储存到 `~/.claude/skills/learned/`

## 设定

编辑 `config.json` 以自订：

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

## Hook 设定

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

## 为什么用 Stop Hook？

- **轻量**：会话结束时只执行一次
- **非阻塞**：不会为每则讯息增加延迟
- **完整上下文**：可存取完整会话记录

## 相关

- [Longform Guide](https://x.com/affaanmustafa/status/2014040193557471352) - 持续学习章节
- `/learn` 指令 - 会话中手动提取模式

---

## 比较笔记（研究：2025 年 1 月）

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

参见：`/Users/affoon/Documents/tasks/12-continuous-learning-v2.md` 完整规格。
