# Git 工作流

## Commit 消息格式

```
<type>: <description>

<optional body>
```

类型：feat、fix、refactor、docs、test、chore、perf、ci

注意：署名通过 ~/.claude/settings.json 全局禁用。

## Pull Request 工作流

创建 PR 时：
1. 分析完整 commit 历史（不仅是最新的 commit）
2. 使用 `git diff [base-branch]...HEAD` 查看所有变更
3. 编写全面的 PR 摘要
4. 包含带 TODO 的测试计划
5. 如果是新分支，使用 `-u` 参数推送

## 功能实现工作流

1. **先规划**
   - 使用 **planner** Agent 制定实现计划
   - 识别依赖项和风险
   - 拆分为多个阶段

2. **TDD 方法**
   - 使用 **tdd-guide** Agent
   - 先编写测试（RED）
   - 实现代码使测试通过（GREEN）
   - 重构优化（IMPROVE）
   - 验证 80%+ 覆盖率

3. **代码评审**
   - 编写代码后立即使用 **code-reviewer** Agent
   - 处理关键和高优先级问题
   - 尽可能修复中优先级问题

4. **提交与推送**
   - 详细的 commit 消息
   - 遵循 Conventional Commits 格式
