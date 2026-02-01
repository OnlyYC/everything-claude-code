---
name: strategic-compact
description: 在逻辑任务边界建议手动上下文压缩，而非依赖任意自动压缩。确保在任务阶段完成后压缩以保留关键上下文。
version: 1.1.0
---

# 策略性压缩技能

在工作流程的策略点建议手动 `/compact`，而非依赖任意的自动压缩。

## 为什么需要策略性压缩？

### 自动压缩的问题

自动压缩在任意点触发时存在以下问题：
- 经常在任务中途丢失重要上下文
- 不知道逻辑任务边界
- 可能中断复杂的多步骤操作
- 无法保留关键的决策和推理过程

### 策略性压缩的优势

逻辑边界的策略性压缩：
- **探索后、执行前** - 压缩研究上下文，保留实现计划
- **完成后里程碑** - 为下一阶段重新开始
- **主要上下文转换前** - 在不同任务前清除探索上下文

## 运作方式

此技能通过跟踪工具调用来识别最佳压缩时机：

1. **跟踪工具调用** - 计算会话中的工具调用次数
2. **门槛检测** - 在可配置门槛建议（默认：50 次调用）
3. **定期提醒** - 门槛后每 25 次调用提醒一次

### 触发条件

| 条件 | 行动 | 说明 |
|------|------|------|
| 工具调用 > 50 次 | 建议压缩 | 达到初始门槛 |
| 之后每 +25 次调用 | 再次提醒 | 持续提醒直到压缩 |
| 探索阶段结束 | 强烈建议压缩 | 保留发现，清除过程 |
| 功能模块完成 | 强烈建议压缩 | 为下一模块准备 |

## 实际操作示例

### 示例 1：探索后压缩

**场景**：理解项目认证流程

```
=== 探索阶段 ===
用户：帮我理解这个项目的认证流程
Claude：[分析代码，解释 JWT、Filter、SecurityConfig...]
用户：这个过滤器什么时候执行？
Claude：[更多代码分析...]
用户：明白了，继续实现功能

=== 此时应该压缩 ===
/compact

=== 压缩提示 ===
已完成项目认证流程的探索：
- 使用 JWT + Spring Security
- JwtAuthenticationFilter 在 UsernamePasswordAuthenticationFilter 之前执行
- SecurityConfig 配置了无状态会话管理
- 密码使用 BCrypt 加密存储

下一步：实现新的 API 端点，需确保 JWT 认证正常工作
```

### 示例 2：功能完成后压缩

**场景**：创建用户管理 API

```
=== 实现阶段 ===
用户：创建用户管理 API
Claude：[创建 Controller、Service、Mapper...]
用户：添加分页功能
Claude：[添加分页支持...]
用户：添加参数校验
Claude：[添加 @Valid 注解和错误处理...]

=== 功能完成，应该压缩 ===
/compact

=== 压缩提示 ===
已完成用户管理 API：
- UserController：增删改查完整接口
- UserService：业务逻辑 + 事务管理
- UserMapper：MyBatis-Plus 基础 CRUD
- 分页：使用 Page<PageRequest>
- 校验：@Valid + 全局异常处理
- 测试：单元测试覆盖率 85%

下一步：实现订单管理功能
```

### 示例 3：调试后压缩

**场景**：解决数据库连接问题

```
=== 调试阶段 ===
用户：数据库连接失败
Claude：[检查配置...]
用户：提示 access denied
Claude：[检查用户权限...]
用户：修改密码后还有问题
Claude：[检查 SSL 配置...]
用户：终于解决了

=== 问题解决，应该压缩 ===
/compact

=== 压缩提示 ===
已解决数据库连接问题：
- 原因：MySQL 8.0 默认需要 SSL 连接
- 解决方案：添加 useSSL=false 参数
- 配置修改：url 中添加 allowPublicKeyRetrieval=true
- 测试验证：连接成功

下一步：继续实现功能
```

### 示例 4：重构后压缩

**场景**：代码重构完成

```
=== 重构阶段 ===
用户：重构用户服务，提取公共接口
Claude：[创建抽象服务类...]
用户：修改订单服务
Claude：[调整依赖关系...]
用户：更新测试用例
Claude：[修改测试代码...]

=== 重构完成，应该压缩 ===
/compact

=== 压缩提示 ===
已完成用户服务重构：
- 创建 AbstractService 基类
- UserService 和 OrderService 继承基类
- 提取公共方法：validate、logOperation
- 测试更新：所有测试通过
- 覆盖率提升：从 75% 提升到 88%

下一步：实现支付服务
```

## 配置说明

### Hook 配置（可选）

如果需要自动检测和提醒，可以配置相应的 hook 脚本：

#### Windows PowerShell Hook

将以下内容保存到 `%USERPROFILE%\.claude\hooks\check-compact.ps1`：

```powershell
# 检查工具调用次数，在适当时机建议压缩
[CmdletBinding()]
param()

$ErrorActionPreference = "Stop"
$callCount = [int](${env:TOOL_CALL_COUNT} -as "0")
$threshold = 50

if ($callCount -gt $threshold) {
    $remainder = $callCount % 25
    if ($remainder -eq 0 -or $callCount -eq $threshold) {
        Write-Host "=== 策略性压缩提醒 ===" -ForegroundColor Yellow
        Write-Host "已进行 $callCount 次工具调用" -ForegroundColor Yellow
        Write-Host "建议在当前任务完成后执行 /compact 命令" -ForegroundColor Yellow
    }
}
```

#### macOS/Linux/WSL Hook

将以下内容保存到 `~/.claude/hooks/check-compact.sh`：

```bash
#!/bin/bash
# 检查工具调用次数，在适当时机建议压缩
set -euo pipefail

call_count=${TOOL_CALL_COUNT:-0}
threshold=50

if [ "$call_count" -gt "$threshold" ]; then
    remainder=$((call_count % 25))
    if [ "$remainder" -eq 0 ] || [ "$call_count" -eq "$threshold" ]; then
        echo "=== 策略性压缩提醒 ==="
        echo "已进行 $call_count 次工具调用"
        echo "建议在当前任务完成后执行 /compact 命令"
    fi
fi
```

### 手动使用原则

如果没有配置 hook，可以手动遵循以下原则：

- **规划后压缩** - 计划确定后，压缩以重新开始
- **调试后压缩** - 继续前清除错误解决上下文
- **不要在实现中途压缩** - 为相关变更保留上下文

## 压缩时机指南

| 时机 | 动作 | 原因 |
|------|------|------|
| 探索阶段完成后 | 立即压缩 | 保留发现，清除探索过程 |
| 实现计划确定后 | 立即压缩 | 保留计划，清除讨论过程 |
| 功能模块完成后 | 立即压缩 | 为下一模块准备 |
| Bug 修复后 | 立即压缩 | 清除调试过程 |
| 实现代码期间 | 不压缩 | 保持上下文连续性 |

## 最佳实践

1. **使用 `/compact` 命令** - 手动触发压缩
2. **描述当前状态** - 压缩时说明已完成和待办事项
3. **保留关键决策** - 记录重要的技术选择和理由
4. **记录下一步** - 明确接下来要做什么

## 压缩提示模板

```
当前会话已完成：
- [任务1] 完成内容...
- [任务2] 完成内容...

下一步计划：
- [任务3] 待办内容...
- [任务4] 待办内容...

关键决策：
- 决策1: 原因...
- 决策2: 原因...
```

## 相关技能

- `continuous-learning` - 持续学习技能
- `continuous-learning-v2` - 基于本能的学习系统
- [Longform Guide](https://x.com/affaanmustafa/status/2014040193557471352) - Token 优化章节

## 最佳实践总结

1. **压缩时机选择**
   - 探索完成后立即压缩
   - 功能模块完成后立即压缩
   - Bug 修复完成后立即压缩
   - 避免在实现代码期间压缩

2. **压缩内容建议**
   - 保留：关键决策、技术选型、架构设计
   - 保留：已完成的功能列表
   - 保留：下一步计划
   - 清除：探索过程中的尝试和错误
   - 清除：详细的调试过程
   - 清除：已经解决的问题详情

3. **压缩命令格式**

使用 `/compact` 命令时，建议按以下格式描述：

```
当前会话已完成：
- [功能/任务1] 完成内容...
- [功能/任务2] 完成内容...

关键决策：
- 决策1：原因...
- 决策2：原因...

下一步计划：
- [任务3] 待办内容...
- [任务4] 待办内容...
```

---

**记住**：策略性压缩可以保持对话上下文清晰，提高后续交互效率。在合适的时机压缩是提升 Claude Code 体验的关键技巧。
