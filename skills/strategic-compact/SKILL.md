---
name: strategic-compact
description: Suggests manual context compaction at logical intervals to preserve context through task phases rather than arbitrary auto-compaction.
---

# 策略性压缩技能

在工作流程的策略点建议手动 `/compact`，而非依赖任意的自动压缩。

## 为什么需要策略性压缩？

自动压缩在任意点触发：
- 经常在任务中途，丢失重要上下文
- 不知道逻辑任务边界
- 可能中断复杂的多步骤操作

逻辑边界的策略性压缩：
- **探索后、执行前** - 压缩研究上下文，保留实现计划
- **完成后里程碑** - 为下一阶段重新开始
- **主要上下文转换前** - 在不同任务前清除探索上下文

## 运作方式

`suggest-compact.sh` 脚本在 PreToolUse（Edit/Write）执行并：

1. **跟踪工具调用** - 计算会话中的工具调用次数
2. **门槛检测** - 在可配置门槛建议（默认：50 次调用）
3. **定期提醒** - 门槛后每 25 次调用提醒一次

## Hook 配置

新增到你的 `~/.claude/settings.json`：

```json
{
  "hooks": {
    "PreToolUse": [{
      "matcher": "tool == \"Edit\" || tool == \"Write\"",
      "hooks": [{
        "type": "command",
        "command": "~/.claude/skills/strategic-compact/suggest-compact.sh"
      }]
    }]
  }
}
```

## 配置

环境变量：
- `COMPACT_THRESHOLD` - 第一次建议前的工具调用次数（默认：50）

## 最佳实践

1. **规划后压缩** - 计划确定后，压缩以重新开始
2. **调试后压缩** - 继续前清除错误解决上下文
3. **不要在实现中途压缩** - 为相关变更保留上下文
4. **阅读建议** - Hook 告诉你*何时*，你决定*是否*

## 相关

- [Longform Guide](https://x.com/affaanmustafa/status/2014040193557471352) - Token 优化章节
- 记忆持久性 hooks - 用于压缩后存活的状态
