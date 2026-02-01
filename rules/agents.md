# Agent 协同

## 可用 Agents

| Agent | 用途 | 触发场景 |
|-------|------|----------|
| **Explore** | 代码库探索：文件查找、代码搜索、结构理解 | 需要多轮探索代码库时 |
| **Plan** | 实现规划：制定步骤、识别风险 | 复杂功能实现前 |
| **architect** | 系统设计：架构决策、技术选型 | 需要架构分析时 |
| **java-reviewer** | Java 代码审查 | 编写 Java 代码后主动审查 |
| **general-purpose** | 通用研究 | 复杂搜索和多步骤任务 |
| **claude-code-guide** | Claude Code 帮助 | 关于 Claude Code 功能问题 |
| **java-build** | 构建修复 | Java 编译错误、依赖问题 |
| **java-test** | TDD 强制执行 | 强制先测试后实现 |
| **e2e** | 集成测试生成 | 生成和运行完整的集成测试套件 |
| **statusline-setup** | 状态栏配置 | 配置 Claude Code 状态栏显示 |

## 主动触发场景

无需用户提示，在以下场景主动使用：

- 复杂功能请求 → Plan Agent（制定计划）
- 编写 Java 代码后 → java-reviewer Agent（主动审查）
- Java 构建失败 → java-build Agent（自动修复）
- 编写 Java 功能 → java-test Agent（强制 TDD）
- 需要集成测试 → e2e Agent（生成测试套件）

## 并行执行策略

对独立操作始终使用并行 Task 执行，在单个消息中同时启动多个 agents。

## Agent 选择策略

**选择原则**：简单操作直接使用工具，复杂任务使用 Agent

- Glob/Grep/Read：单次查找/搜索/读取
- Explore Agent：多轮代码探索
- Plan Agent：实现规划
- architect Agent：架构决策
- java-reviewer Agent：Java 代码审查
- java-build Agent：Java 构建修复
- java-test/e2e Agent：测试相关

## 典型工作流

**探索 → 规划 → 实现 → 审查**：Explore Agent → Plan Agent → java-test Agent → java-reviewer Agent

**并行多视角分析**：对复杂问题并行启动 architect + java-reviewer + Explore

**TDD 循环**：RED（生成测试）→ GREEN（实现代码）→ REFACTOR（重构）→ VERIFY（验证覆盖率）

## Agent vs 工具

**关键区别**：Agent 可自主决策和多步操作，工具直接执行单次操作

**使用原则**：简单操作用工具，复杂任务用 Agent

## Agent 调用参数

- `subagent_type`：Agent 类型（Explore、Plan、java-reviewer 等）
- `prompt`：任务描述
- `model`：可选模型（haiku/sonnet/opus）
- `max_turns`：可选最大轮次限制

**模型建议**：

| Agent | 推荐模型 | 理由 |
|-------|----------|------|
| Explore | haiku | 简单模式匹配，快速响应 |
| Plan | sonnet/opus | 需要架构推理能力 |
| architect | opus | 复杂系统设计需要深度推理 |
| java-reviewer | sonnet | 平衡速度和审查质量 |
| java-build | haiku | 机械性修复任务 |
| general-purpose | sonnet | 通用任务的默认选择 |

## 最佳实践

**选择合适 Agent**：简单任务不用 Agent，复杂任务选专业 Agent

**并行执行**：独立任务必须并行，避免串行等待

**明确 Prompt**：具体描述任务目标，避免模糊指令

**合理轮次**：设置 max_turns 避免无限循环，但不要过少限制
