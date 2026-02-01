# Agent 协同

## 可用 Agents

位于 `~/.claude/agents/`：

| Agent | 用途 | 何时使用 |
|-------|------|----------|
| planner | 实现规划 | 复杂功能、重构 |
| architect | 系统设计 | 架构决策 |
| tdd-guide | 测试驱动开发 | 新功能、Bug 修复 |
| code-reviewer | 代码评审 | 编写代码后 |
| security-reviewer | 安全分析 | 提交前 |
| build-error-resolver | 修复构建错误 | 构建失败时 |
| e2e-runner | E2E 测试 | 关键用户流程 |
| refactor-cleaner | 无用代码清理 | 代码维护 |
| doc-updater | 文档 | 更新文档 |

## 主动使用 Agent

无需用户提示：
1. 复杂功能请求 - 使用 **planner** Agent
2. 刚编写/修改代码 - 使用 **code-reviewer** Agent
3. Bug 修复或新功能 - 使用 **tdd-guide** Agent
4. 架构决策 - 使用 **architect** Agent

## 并行任务执行

对独立操作始终使用并行 Task 执行：

```markdown
# 推荐：并行执行
并行启动 3 个 agents：
1. Agent 1：AuthService 的安全分析
2. Agent 2：缓存系统的性能审查
3. Agent 3：StringUtils 的类型检查

# 不推荐：不必要的串行
先 agent 1，然后 agent 2，然后 agent 3
```

## 多视角分析

对于复杂问题，使用分角色子 agents：
- 事实审核员
- 高级工程师
- 安全专家
- 一致性审核员
- 冗余检查员
