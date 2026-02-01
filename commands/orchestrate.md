---
name: orchestrate
description: 复杂任务的循序 Agent 工作流程
command: /orchestrate [feature|bugfix|refactor|security|custom] [task-description]
---

# Orchestrate 指令

复杂任务的循序 Agent 工作流程。

## 使用方式

`/orchestrate [workflow-type] [task-description]`

## 工作流程类型

### feature
完整的功能实现工作流程：
```
planner -> java-tdd-guide -> code-reviewer -> java-reviewer
```

### bugfix
Bug 调查和修复工作流程：
```
explorer -> java-tdd-guide -> code-reviewer
```

### refactor
安全重构工作流程：
```
architect -> code-reviewer -> java-tdd-guide
```

### security
以安全性为焦点的审查：
```
java-reviewer -> code-reviewer -> architect
```

## 执行模式

对工作流程中的每个 Agent：

1. **呼叫 Agent**，带入前一个 Agent 的上下文
2. **收集输出**作为结构化交接文档
3. **传递给下一个 Agent**
4. **汇总结果**为最终报告

## 交接文档格式

Agent 之间，建立交接文档：

```markdown
## 交接：[前一个 Agent] -> [下一个 Agent]

### 上下文
[完成事项的摘要]

### 发现
[关键发现或决策]

### 修改的文件
[触及的文件列表]

### 开放问题
[下一个 Agent 的未解决项目]

### 建议
[建议的后续步骤]
```

## 最终报告格式

```
协调报告
====================
工作流程：feature
任务：新增用户认证模块
Agents：planner -> java-tdd-guide -> code-reviewer -> java-reviewer

摘要
-------
[一段摘要]

AGENT 输出
-------------
Planner：[摘要]
Java TDD Guide：[摘要]
Code Reviewer：[摘要]
Java Reviewer：[摘要]

变更的文件
-------------
[列出所有修改的文件]

测试结果
------------
[测试通过/失败摘要]

安全性状态
---------------
[安全性发现]

建议
--------------
[发布 / 需要改进 / 阻挡]
```

## 平行执行

对于独立的检查，平行执行 Agents：

```markdown
### 平行阶段
同时执行：
- code-reviewer（品质）
- java-reviewer（Java 特定问题）
- architect（设计）

### 合并结果
将输出合并为单一报告
```

## 参数

| 参数 | 说明 |
|------|------|
| `feature <description>` | 完整功能工作流程 |
| `bugfix <description>` | Bug 修复工作流程 |
| `refactor <description>` | 重构工作流程 |
| `security <description>` | 安全性审查工作流程 |
| `custom <agents> <description>` | 自定义 Agent 序列 |

## 自定义工作流程示例

```
/orchestrate custom "architect,java-tdd-guide,code-reviewer" "重新设计缓存层"
```

## 提示

1. **复杂功能从 `/plan` 开始**
2. **合并前总是包含 code-reviewer**
3. **对验证/支付/PII 使用 java-reviewer**
4. **保持交接简洁** - 专注于下一个 Agent 需要的内容
5. **如有需要，在 Agents 之间执行 verification**

## 相关指令

- `/plan` - 创建实现计划
- `/tdd` - 测试驱动开发
- `/code-review` - 代码审查
- `/java-review` - Java 特定审查
- `/verify` - 完整验证循环
