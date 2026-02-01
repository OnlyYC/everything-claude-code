---
name: performance
description: 性能优化指南
priority: optional
tags: [performance, context, parallel, model]
---

# 性能优化

## 上下文窗口管理

### 上下文敏感度分类

**高敏感度任务**（避免在高上下文使用）：
- 跨多个文件的代码重构
- 复杂功能的端到端实现
- 多层依赖的调试

**低敏感度任务**（可安全进行）：
- 单文件编辑/修复
- 独立工具开发
- 文档更新
- 简单 Bug 修复

### Token 节省策略

| 策略 | 说明 |
|------|------|
| **使用专用工具** | Glob/Grep/Read 代替 Bash 命令 |
| **并行读取** | 并行读取多个文件，而非串行等待 |
| **使用 Agents** | Explore Agent 多轮探索不污染主上下文 |

## 模型选择策略

| 模型 | 成本 | 延迟 | 推理能力 | 适用场景 |
|------|------|------|----------|----------|
| **haiku** | 低 | 低 | 中等 | 简单搜索、格式化 |
| **sonnet** | 中 | 中 | 高 | 通用代码实现、问题诊断（默认） |
| **opus** | 高 | 高 | 最高 | 复杂架构设计、深层推理 |

### 决策树

```
需要复杂推理/架构设计？
├── 是 → opus
└── 否
    ├── 简单任务（模式匹配）？
    │   ├── 是 → haiku
    │   └── 否 → sonnet
```

## 并行工具调用

### 规则

**始终对独立操作使用并行工具调用**

```
# 推荐：并行读取多个文件
单个消息中包含：
- Read UserService.java
- Read OrderService.java
- Read PaymentService.java

# 推荐：并行执行多个 Agents
单个消息中包含：
- Agent 1：分析模块 A
- Agent 2：审查模块 B
- Agent 3：检查模块 C

# 不推荐：串行等待
先 Read UserService.java → 等待 → 再 Read OrderService.java...
```

## 增量式工作方法

### 大型任务分解

对于复杂/大型任务：

```
1. 拆分为小的、独立的步骤
2. 每步完成后验证
3. 避免一次性处理太多文件
4. 使用 Task 工具追踪进度
```

### 增量开发示例

```markdown
# 任务：实现用户认证功能

# 错误方式：一次性完成
一次性创建：AuthService, AuthController, JWT工具类, 测试类...

# 正确方式：增量完成
步骤 1：创建 Entity 和 Mapper（验证数据库）
步骤 2：实现 Service 核心逻辑（单元测试）
步骤 3：实现 Controller 接口（集成测试）
步骤 4：添加 JWT 工具类（端到端测试）
步骤 5：补充异常处理和日志
```

## 代码读取优化

### 优先使用专用工具

| 操作 | 专用工具 | 避免 |
|------|----------|------|
| 查找文件 | `Glob` | `Bash find` / `Bash ls` |
| 搜索内容 | `Grep` | `Bash grep` / `Bash rg` |
| 读取文件 | `Read` | `Bash cat` / `Bash head` |

### 探索代码库策略

| 需求 | 工具/Agent |
|------|------------|
| 单次查找文件 | 直接使用 Glob |
| 单次搜索关键词 | 直接使用 Grep |
| 多轮探索/理解结构 | 使用 Explore Agent |
| 分析代码质量 | 使用 java-reviewer Agent |
| 制定实现计划 | 使用 Plan Agent |

## 数据库查询优化

### N+1 查询问题

**问题识别：**

```java
// ❌ N+1 查询问题
List<User> users = userMapper.selectList(null);  // 1 次查询
for (User user : users) {
    List<Order> orders = orderMapper.selectByUserId(user.getId());  // N 次查询
}
```

**解决方案：**

```java
// ✓ 批量查询 + 内存组装
List<User> users = userMapper.selectList(null);
Set<Long> userIds = users.stream().map(User::getId).collect(Collectors.toSet());
Map<Long, List<Order>> orderMap = orderMapper.selectBatchIds(userIds)
    .stream()
    .collect(Collectors.groupingBy(Order::getUserId));
users.forEach(u -> u.setOrders(orderMap.getOrDefault(u.getId(), Collections.emptyList())));
```

## 代码缓存策略

### 文件内容缓存

```
# 规则：同一会话中，Read 工具读取的文件会被缓存

# 最佳实践：
- 重复读取同一文件时，Claude 会使用缓存
- 避免在多次调用中重复读取相同文件
- 首次读取完整内容，后续在内存中操作
```

## 构建故障排查

### 增量修复策略

如果构建失败：

```
1. 分析所有错误（收集全部）
2. 按依赖关系排序错误
3. 每次修复 1-3 个相关错误
4. 验证构建状态
5. 重复直到通过
```

### Java 构建修复

```bash
# 使用 java-build Agent 自动修复
# Agent 会：
1. 分析编译错误
2. 识别根本原因
3. 增量修复（每次少量）
4. 验证构建状态
```

## Agent 选择优化

### 何时使用 Agent

| 场景 | 使用 Agent | 直接使用工具 |
|------|-----------|-------------|
| 查找特定文件 | 否 → Glob | ✓ |
| 单个关键词搜索 | 否 → Grep | ✓ |
| 多轮代码探索 | ✓ Explore Agent | - |
| 制定实现计划 | ✓ Plan Agent | - |
| 架构决策 | ✓ architect Agent | - |
| Java 代码审查 | ✓ java-reviewer Agent | - |
| 运行命令/测试 | - | ✓ Bash |
| 读取/编辑文件 | - | ✓ Read/Edit |

## 性能优化清单

在执行任务前，确认：

- [ ] 独立操作使用并行工具调用
- [ ] 优先使用专用工具（Glob/Grep/Read）
- [ ] 多轮探索使用 Explore Agent
- [ ] 根据任务复杂度选择合适模型
- [ ] 大任务拆分为小步骤增量完成
- [ ] 避免重复读取相同文件
- [ ] 构建失败时增量修复
