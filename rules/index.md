---
name: index
description: Claude Code 规则集索引
version: 1.0.0
lastUpdated: 2026-02-01
---

# Claude Code Rules 索引

> 本目录包含 Claude Code 的完整规则集，适用于 **Java / Spring Boot / MyBatis-Plus** 项目。

## 规则优先级

| 优先级 | 标记 | 说明 |
|--------|------|------|
| **强制** | **必须** | 违反将导致代码质量或安全问题 |
| **推荐** | **应该** | 遵循可提升代码质量和开发效率 |
| **可选** | **可以** | 根据项目情况灵活选择 |

## 规则文件导航

### 核心规则（必读）

| 文件 | 描述 | 优先级 | 版本 |
|------|------|--------|------|
| [agents.md](agents.md) | Agent 协作策略：何时使用 Agent vs 直接使用工具 | **必须** | 1.0.0 |
| [security.md](security.md) | 安全指南：密钥管理、SQL 注入防护、XSS/CSRF | **必须** | 1.0.0 |
| [testing.md](testing.md) | 测试规范：TDD、80% 覆盖率、JUnit 5 最佳实践 | **必须** | 1.0.0 |

### 代码风格

| 文件 | 描述 | 优先级 | 版本 |
|------|------|--------|------|
| [coding-style.md](coding-style.md) | Java 代码风格：不可变性、分层架构、命名规范 | **应该** | 1.0.0 |
| [patterns.md](patterns.md) | 常见模式：统一响应格式、DTO/VO 分层、对象转换 | **应该** | 1.0.0 |

### 工作流程

| 文件 | 描述 | 优先级 | 版本 |
|------|------|--------|------|
| [git-workflow.md](git-workflow.md) | Git 工作流：分支命名、Commit 消息、PR 流程 | **应该** | 1.0.0 |
| [hooks.md](hooks.md) | Hook 系统：PreToolUse/PostToolUse 配置 | **可选** | 1.0.0 |
| [performance.md](performance.md) | 性能优化：上下文管理、模型选择、并行执行 | **可选** | 1.0.0 |

## 快速参考

### Java 项目开发流程

```
1. 规划 → Plan Agent（复杂功能）
2. TDD → java-test Agent（强制先测试）
3. 实现 → 遵循 coding-style.md 和 patterns.md
4. 审查 → java-reviewer Agent（主动审查）
5. 提交 → 遵循 git-workflow.md
```

### 通用规则（所有项目）

- **安全第一**：始终遵循 security.md
- **测试先行**：使用 TDD 方法（testing.md）
- **性能优先**：独立操作使用并行工具调用（performance.md）

## 常见问题

### Q: 何时使用 Agent？

**简单操作（直接使用工具）**：
- 查找已知路径文件 → Glob
- 搜索单个关键词 → Grep
- 读取单个文件 → Read
- 运行简单命令 → Bash

**复杂任务（使用 Agent）**：
- 3 轮以上代码探索 → Explore Agent
- 涉及 3+ 文件的功能实现 → Plan Agent
- 需要架构决策 → architect Agent
- Java 代码编写后 → java-reviewer Agent（主动）

详见 [agents.md](agents.md)

### Q: Java 项目必须遵循哪些规则？

1. **强制 TDD**：使用 java-test Agent，先编写测试后实现
2. **主动审查**：编写 Java 代码后立即使用 java-reviewer Agent
3. **安全检查**：提交前通过 security.md 的安全检查清单
4. **测试覆盖率**：达到 80% 以上（testing.md）

### Q: 如何提高性能？

1. **并行工具调用**：在单个消息中同时读取多个文件
2. **专用工具优先**：使用 Glob/Grep/Read 而非 Bash 命令
3. **合理选择模型**：简单任务用 haiku，复杂任务用 sonnet/opus
4. **使用 Agent**：多轮探索使用 Explore Agent 减少上下文污染

详见 [performance.md](performance.md)

## 版本历史

| 版本 | 日期 | 变更 |
|------|------|------|
| 1.0.0 | 2026-02-01 | 初始版本：9 个规则文件 + 索引，添加 YAML 元数据 |
