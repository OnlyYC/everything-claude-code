---
name: agents
description: Agent 协作策略与使用指南
priority: must
tags: [agents, collaboration, task]
---

# Agent 协作

## 可用 Agents

| Agent | 用途 | 触发条件 | 推荐模型 |
|-------|------|----------|----------|
| **Explore** | 代码库探索 | 3 轮以上搜索/查找 | haiku |
| **Plan** | 实现规划 | 3+ 文件变更或架构决策 | sonnet/opus |
| **architect** | 系统设计 | 技术选型、架构权衡 | opus |
| **java-reviewer** | Java 代码审查 | 编写 Java 代码后立即触发 | sonnet |
| **java-build** | 构建修复 | Java 编译错误 | haiku |
| **java-test** | TDD 强制执行 | 实现 Java 功能时 | sonnet |
| **e2e** | 集成测试生成 | 验证完整 API 流程 | sonnet |
| **general-purpose** | 通用研究 | 跨多个来源综合信息 | sonnet |

## 何时使用 Agent vs 直接使用工具

### 使用专用工具（单次操作）

| 操作 | 使用工具 | 示例 |
|------|----------|------|
| 查找特定文件 | `Glob` | 查找 `src/main/java/**/*.java` |
| 搜索单个关键词 | `Grep` | 搜索特定类名或方法名 |
| 读取单个文件 | `Read` | 读取已知路径的文件 |
| 运行命令/测试 | `Bash` | 执行 git、npm、mvn 等命令 |
| 编辑文件 | `Edit`/`Write` | 修改或创建文件 |

### 使用 Agent（多步/复杂任务）

| 场景 | 使用 Agent | 理由 |
|------|-----------|------|
| 多轮代码探索 | **Explore** | 避免主上下文污染 |
| 需要实现计划 | **Plan** | 需要架构分析和规划 |
| 架构决策 | **architect** | 需要深度推理 |
| 编写 Java 代码后 | **java-reviewer** | 主动审查代码质量 |
| Java 构建失败 | **java-build** | 自动修复编译错误 |
| 实现 Java 功能 | **java-test** | 强制 TDD 流程 |

## 主动触发规则

### Java 项目（强制执行）

| 时机 | Agent | 说明 |
|------|-------|------|
| 编写 Java 代码后 | **java-reviewer** | 立即审查代码质量、安全性 |
| Java 构建失败 | **java-build** | 自动修复编译错误 |
| 实现 Java 功能 | **java-test** | 强制 TDD：先测试后实现 |
| 需要 API 集成测试 | **e2e** | 生成完整测试套件 |

### 通用项目（条件触发）

| 条件 | Agent | 触发标准 |
|------|-------|----------|
| 复杂功能实现 | **Plan** | 涉及 3+ 文件或架构变更 |
| 代码库理解 | **Explore** | 需要 3+ 轮搜索/查找 |
| 技术方案评估 | **architect** | 需要比较多种方案 |

## 并行执行策略

### 原则

**对独立操作始终使用并行 Task 执行**

### 并行模式

```
# 模式 1：并行读取文件
单个消息：Read File1 + Read File2 + Read File3

# 模式 2：并行启动 Agents
单个消息：Explore Agent + Grep FileA + Read FileB

# 模式 3：并行多视角分析
单个消息：architect Agent + java-reviewer Agent + Explore Agent
```

### 串行模式（仅在依赖时使用）

```
操作 B 依赖操作 A 的结果时，才串行执行
```

## Agent 参数配置

| 参数 | 说明 | 推荐值 |
|------|------|--------|
| `subagent_type` | Agent 类型 | 必需 |
| `prompt` | 任务描述 | 必需，具体明确 |
| `model` | 模型选择 | 可选（见下表） |
| `max_turns` | 最大轮次 | 可选，默认 20 |

### 模型选择指南

| Agent | 推荐模型 | 理由 |
|-------|----------|------|
| Explore | haiku | 模式匹配任务，快速响应 |
| Plan | sonnet/opus | 需要推理和规划能力 |
| architect | opus | 复杂系统设计需要深度推理 |
| java-reviewer | sonnet | 平衡速度和审查质量 |
| java-build | haiku | 机械性修复任务 |
| e2e | sonnet | API 测试需要理解业务逻辑 |
| general-purpose | sonnet | 通用任务默认选择 |

## 典型工作流

### 探索 → 规划 → 实现 → 审查

```
Explore Agent（理解代码库）
    ↓
Plan Agent（制定实现计划）
    ↓
java-test Agent（TDD：生成测试）
    ↓
实现代码（使用 Edit/Write）
    ↓
java-reviewer Agent（主动审查）
```

### 并行多视角分析

对复杂问题并行启动：
- architect Agent：架构分析
- Explore Agent：代码探索
- java-reviewer Agent：质量审查

### TDD 循环（Java）

```
RED   → java-test Agent（生成失败的测试）
GREEN → 实现代码（最小实现）
REFACTOR → 重构优化
VERIFY → 验证 80%+ 覆盖率
```

## Agent vs 工具决策树

```
需要操作代码库？
  ├─ 是
  │   ├─ 仅需 1 次查找/搜索？
  │   │   └─ 是 → 直接使用 Glob/Grep
  │   └─ 需要 3+ 轮探索？
  │       └─ 是 → Explore Agent
  │
  ├─ 需要实现功能？
  │   ├─ 涉及 3+ 文件或架构决策？
  │   │   └─ 是 → Plan Agent
  │   └─ Java 代码？
  │       └─ 是 → java-test Agent（强制 TDD）
  │
  └─ 需要审查代码？
      └─ Java 代码？
          └─ 是 → java-reviewer Agent（主动）
```

## 最佳实践

| 原则 | 说明 |
|------|------|
| **简单操作用工具** | 单次查找/搜索/读取直接使用 Glob/Grep/Read |
| **复杂任务用 Agent** | 多轮探索、规划、审查使用 Agent |
| **并行执行独立操作** | 在单个消息中同时启动多个 Agents |
| **明确 Prompt** | 具体描述任务目标，避免模糊指令 |
| **合理选择模型** | 根据任务复杂度选择 haiku/sonnet/opus |
| **Java 代码主动审查** | 编写后立即使用 java-reviewer Agent |
| **Java 功能强制 TDD** | 使用 java-test Agent 确保先测试后实现 |
