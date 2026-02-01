# 性能优化

## 上下文窗口管理

上下文窗口是有限资源。Claude Code 的上下文管理：

> **注意**：以下数值为示例参考，实际容量根据所选模型版本可能有所不同。具体值请参考相应模型的官方文档。

| 指标 | 相对范围 | 说明 |
|------|---------|------|
| **上下文容量** | 模型上限 | 不同模型有不同的上下文窗口大小 |
| **典型使用** | 20-50% | 大多数任务的有效范围 |
| **警告阈值** | 70-85% | 接近上限时性能下降 |

### 上下文敏感度分类

**高敏感度任务（避免在高上下文时进行）：**
- 跨多个文件的代码重构
- 复杂功能的端到端实现
- 多层依赖的调试
- 架构重新设计

**低敏感度任务（可安全进行）：**
- 单文件编辑/修复
- 独立工具开发
- 文档更新
- 简单 Bug 修复
- 代码格式化

### Token 节省策略

```markdown
# 策略 1：使用专用工具代替 Bash
- 文件搜索 → 使用 Glob（而非 find/ls）
- 内容搜索 → 使用 Grep（而非 grep/rg）
- 文件读取 → 使用 Read（而非 cat/head/tail）

# 策略 2：并行读取而非串行
# 推荐：并行读取 3 个文件
同时读取：File1.js, File2.js, File3.js

# 不推荐：串行等待
先读 File1.js → 等待 → 读 File2.js...

# 策略 3：使用 Agents 减少上下文污染
- Explore Agent：多轮探索不污染主上下文
- Plan Agent：规划在子进程完成
```

## 模型选择策略

Claude Code 支持多种模型，按需选择：

| 模型 | 成本 | 延迟 | 推理能力 | 适用场景 |
|------|------|------|----------|----------|
| **haiku** | 低 | 低 | 中等 | 简单搜索、格式化、小文件编辑 |
| **sonnet** | 中 | 中 | 高 | 通用代码实现、问题诊断（默认） |
| **opus** | 高 | 高 | 最高 | 复杂架构设计、深层推理 |

### 使用场景决策树

```
需要复杂推理/架构设计？
├── 是 → opus
└── 否
    ├── 简单任务（<5分钟）？
    │   ├── 是 → haiku（节省成本）
    │   └── 否 → sonnet（平衡）
```

### 典型场景示例

| 任务 | 推荐模型 | 理由 |
|------|----------|------|
| 查找特定文件 | haiku | 简单模式匹配 |
| 代码格式化 | haiku | 机械操作 |
| 实现新功能 | sonnet | 需要理解和推理 |
| 修复 Bug | sonnet | 需要分析代码逻辑 |
| 系统架构设计 | opus | 需要深度推理 |
| 重构大型模块 | opus | 复杂依赖分析 |

## 并行工具调用

### 规则

**始终对独立操作使用并行工具调用**

```markdown
# 推荐：并行读取多个文件
单个消息中包含：
- Read UserService.java
- Read OrderService.java
- Read PaymentService.java

# 推荐：并行执行多个 Agents
单个消息中包含：
- Agent 1：AuthService 的安全分析
- Agent 2：缓存系统的性能审查
- Agent 3：StringUtils 的类型检查

# 不推荐：串行等待
先 Read UserService.java → 等待 → 再 Read OrderService.java...
```

### 并行执行模式

```markdown
# 模式 1：并行读取（最常见）
当需要读取多个不相关的文件时，始终并行

# 模式 2：并行搜索
当需要搜索多个不同模式时，并行执行 Grep

# 模式 3：并行 Agents
当需要多个不同视角的分析时，并行启动 Agents

# 模式 4：串行执行（仅在依赖时）
操作 B 依赖操作 A 的结果时，才串行执行
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
| 读取文件 | `Read` | `Bash cat` / `Bash head` / `Bash tail` |

### 探索代码库策略

| 需求 | 工具/Agent |
|------|------------|
| 单次查找文件 | 直接使用 Glob |
| 单次搜索关键词 | 直接使用 Grep |
| 多轮探索/理解结构 | 使用 Explore Agent |
| 分析代码质量 | 使用 java-reviewer Agent |
| 制定实现计划 | 使用 Plan Agent |

## 性能监控指标

### 可观察性指标

在实际工作中，可以关注以下指标：

| 指标 | 监控方式 | 目标 |
|------|----------|------|
| **响应时间** | 感知 | 快速响应，通常几秒到十几秒 |
| **迭代次数** | 观察 | 简单任务尽量少轮次完成 |
| **工具并行度** | 观察 | 独立操作应并行 |
| **上下文效率** | 观察 | 避免重复读取相同文件 |

### 性能问题模式

```markdown
# 模式 1：串行读取（低效）
❌ 串行读取 10 个文件

# 模式 2：重复读取（浪费）
❌ 多次读取同一文件未使用缓存

# 模式 3：大文件全量读取（低效）
❌ 读取整个文件，实际只需部分内容
✅ 使用 Read 工具的 offset/limit 参数

# 模式 4：不必要的 Agent 调用（延迟）
❌ 简单任务也启动 Agent
✅ 直接使用工具完成
```

## 数据库查询优化

### N+1 查询问题

**问题识别：**

```java
// ❌ N+1 查询问题
List<User> users = userMapper.selectList(null);  // 1 次查询
for (User user : users) {
    List<Order> orders = orderMapper.selectByUserId(user.getId());  // N 次查询
}
// 总计：1 + N 次查询
```

**解决方案：**

```java
// ✓ 方案 1：使用 MyBatis-Plus 关联查询
@TableName("t_user")
public class User {
    @TableField(select = false)  // 查询时不自动加载
    private List<Order> orders;
}

// ✓ 方案 2：使用 Join 查询
@Select("SELECT u.*, o.* FROM t_user u " +
        "LEFT JOIN t_order o ON u.id = o.user_id " +
        "WHERE u.deleted = 0")
List<UserWithOrders> selectUsersWithOrders();

// ✓ 方案 3：批量查询 + 内存组装
List<User> users = userMapper.selectList(null);
Set<Long> userIds = users.stream().map(User::getId).collect(Collectors.toSet());
Map<Long, List<Order>> orderMap = orderMapper.selectBatchIds(userIds)
    .stream()
    .collect(Collectors.groupingBy(Order::getUserId));
users.forEach(u -> u.setOrders(orderMap.getOrDefault(u.getId(), Collections.emptyList())));
```

**检测方法：**

```bash
# 启用 MyBatis SQL 日志
logging:
  level:
    com.example.mapper: DEBUG

# 检查日志中是否有重复的 SELECT 语句
```

## 代码缓存策略

### 文件内容缓存

```markdown
# 规则：同一会话中，Read 工具读取的文件会被缓存

# 最佳实践：
- 重复读取同一文件时，Claude 会使用缓存
- 避免在多次调用中重复读取相同文件
- 首次读取完整内容，后续在内存中操作

# 注意：
- 文件在外部被修改后，需要重新读取
- Edit/Write 工具会自动更新缓存
```

### 搜索结果缓存

```markdown
# Grep 搜索结果可被重用
- 首次搜索后，结果在上下文中可用
- 后续分析可直接引用，无需重复搜索
```

## 构建故障排查

### 增量修复策略

如果构建失败：

```markdown
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

### 通用构建修复

```markdown
# 模式 1：依赖缺失
→ 安装缺失的依赖

# 模式 2：语法错误
→ 修复语法问题

# 模式 3：类型不匹配
→ 修正类型定义

# 模式 4：配置问题
→ 检查配置文件
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

### Agent 并行执行

```markdown
# 推荐：并行启动多个独立 Agents
单个消息中：
- Agent 1：分析模块 A 的性能
- Agent 2：审查模块 B 的安全性
- Agent 3：检查模块 C 的代码质量

# 不推荐：串行执行
先 Agent 1 → 等待 → 再 Agent 2...
```

## Token 预算分配

> **注意**：Token 消耗量级会根据模型版本、代码复杂度和任务类型有显著差异。以下仅为相对参考值，实际使用时应根据具体情况调整。

### 典型任务相对量级

| 任务类型 | 相对消耗 | 说明 |
|----------|---------|------|
| 读取单个文件 | 低 | 取决于文件大小 |
| 简单编辑 | 低-中 | 简单位移的修改 |
| 复杂重构 | 中-高 | 跨多个文件的更改 |
| 架构设计 | 高 | 需要大量上下文和推理 |
| 完整功能实现 | 很高 | 可能接近上下文上限 |

### 预算管理原则

```markdown
1. 优先预留 20-30% buffer（根据任务不确定性调整）
2. 单次任务避免超过上下文容量的 50-70%
3. 超过阈值时考虑分阶段处理
4. 使用 Agents 减少主上下文污染
5. 根据模型版本动态调整预算策略
```

## 性能优化清单

在执行任务前，确认：

- [ ] 独立操作使用并行工具调用
- [ ] 优先使用专用工具（Glob/Grep/Read）
- [ ] 多轮探索使用 Explore Agent
- [ ] 根据任务复杂度选择合适模型
- [ ] 大任务拆分为小步骤增量完成
- [ ] 避免重复读取相同文件
- [ ] 构建失败时增量修复
