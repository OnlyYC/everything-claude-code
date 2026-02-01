---
name: plan
description: 重述需求、评估风险并创建分步实现计划。在修改任何代码前等待用户确认
command: /plan
---

# Plan 指令

此指令呼叫 **planner** Agent，在撰写任何代码前建立全面的实现计划。

## 此指令的功能

1. **重述需求** - 澄清需要构建什么
2. **识别风险** - 揭示潜在问题和阻碍
3. **建立步骤计划** - 将实现拆解为阶段
4. **等待确认** - 必须在继续前收到使用者批准

## 何时使用

在以下情况使用 `/plan`：
- 开始新功能
- 进行重大架构变更
- 进行复杂重构
- 多个文件/组件会受影响
- 需求不清楚或模糊

## 运作方式

planner Agent 会：

1. **分析请求**并以清晰的语言重述需求
2. **拆解为阶段**，包含具体、可执行的步骤
3. **识别依赖**组件之间的关系
4. **评估风险**和潜在阻碍
5. **估计复杂度**（高/中/低）
6. **呈现计划**并等待您明确确认

## 使用示例

```
使用者：/plan 我需要在市场解决时新增实时通知

Agent（planner）：
# 实现计划：实时市场解决通知

## 需求重述
- 当使用者关注的市场解决时发送通知
- 支持多个通知管道（站内信、邮件、Webhook）
- 确保通知可靠送达
- 包含市场结果和使用者持仓结果

## 实现阶段

### 阶段 1：数据库表设计
- 新建 notification 表，字段：id、user_id、market_id、type、status、created_at
- 新建 user_notification_preferences 表用于管道偏好
- 在 user_id 和 market_id 上建立索引以优化性能

### 阶段 2：通知服务
- 在 `com.example.service` 建立 NotificationService
- 使用 Spring Event 机制实现异步通知
- 为失败的送达新增重试逻辑
- 建立通知模板

### 阶段 3：整合点
- 连接到市场解决逻辑（当状态变更为 "resolved"）
- 查询所有在该市场有持仓的使用者
- 为每个使用者发布通知事件

### 阶段 4：Controller 层
- 新建 NotificationController
- 提供 GET /api/notifications 获取通知列表
- 提供 PUT /api/notifications/{id}/read 标记已读
- 提供 GET /api/notifications/preferences 获取偏好设置

## 依赖
- Spring Event 机制（ApplicationEventPublisher）
- 邮件服务（JavaMailSender）
- Redis（可选，用于缓存）

## 风险
- 高：邮件送达率（需要配置 SMTP）
- 中：每个市场 1000+ 使用者时的性能
- 中：如果市场频繁解决可能造成通知垃圾信息
- 低：Spring Event 异步处理的开销

## 估计复杂度：中
- 后端：4-6 小时
- 前端：2-3 小时
- 测试：2-3 小时
- 总计：8-12 小时

**等待确认**：继续此计划？（是/否/修改）
```

## 重要提醒

**关键**：planner Agent **不会**撰写任何代码，直到您明确以"是"、"继续"或类似肯定回应确认计划。

如果您想要修改，回应：
- "修改：[您的变更]"
- "不同的方法：[替代方案]"
- "跳过阶段 2，先做阶段 3"

## 与其他指令的整合

计划后：
- 使用 `/tdd` 以测试驱动开发实现
- 如果发生构建错误，使用 `/build-fix` 或 `/java-build`
- 使用 `/code-review` 审查完成的实现

## 相关 Agent

此指令呼叫位于以下位置的 `planner` Agent：
`~/.claude/agents/planner.md`（或 `~/.claude/agents/plan.md`）
