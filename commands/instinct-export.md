---
name: instinct-export
description: 导出经验文件，与团队成员共享或迁移到其他项目
command: /instinct-export [--domain <名称>] [--min-confidence <值>] [--output <文件>] [--format <yaml|json|md>] [--include-evidence]
---

# 经验导出指令

将已学习的经验导出为可共享格式。适用于：
- 与团队成员共享编码规范
- 迁移到新机器
- 贡献到项目规范仓库

## 使用方法

```
/instinct-export                           # 导出所有个人经验
/instinct-export --domain testing          # 仅导出测试相关经验
/instinct-export --min-confidence 0.7      # 仅导出高置信度经验
/instinct-export --output team-instincts.yaml
```

## 执行步骤

1. 从 `~/.claude/homunculus/instincts/personal/` 读取经验文件
2. 根据命令行参数过滤
3. 清理敏感信息：
   - 移除会话 ID
   - 移除文件路径（仅保留模式）
   - 移除早于「上周」的时间戳
4. 生成导出文件

## 输出格式

生成 YAML 格式文件：

```yaml
# 经验导出文件
# 生成时间：2024-01-15  # 使用实际导出日期
# 来源：personal
# 数量：12 条经验

version: "2.0"
exported_by: "continuous-learning-v2"
export_date: "2025-01-22T10:30:00Z"

instincts:
  - id: prefer-service-layer
    trigger: "编写业务逻辑时"
    action: "优先使用 Service 层封装，避免 Controller 直接处理业务"
    confidence: 0.9
    domain: spring-boot
    observations: 15

  - id: test-first-workflow
    trigger: "新增功能时"
    action: "遵循 TDD 原则：先写测试，再实现功能"
    confidence: 0.9
    domain: testing
    observations: 12

  - id: mybatis-param-safe
    trigger: "编写 MyBatis SQL 时"
    action: "使用 #{} 参数绑定，避免 ${} SQL 注入风险"
    confidence: 0.95
    domain: security
    observations: 20

  - id: dto-vo-separation
    trigger: "定义接口参数时"
    action: "严格区分 DTO/VO/Entity，禁止直接返回 Entity"
    confidence: 0.85
    domain: spring-boot
    observations: 10

  - id: grep-before-edit
    trigger: "修改代码时"
    action: "先用 Grep 搜索确认影响范围，再用 Read 读取，最后 Edit 修改"
    confidence: 0.8
    domain: workflow
    observations: 8
```

## 隐私保护

导出内容包括：
- ✅ 触发模式
- ✅ 动作描述
- ✅ 置信度分数
- ✅ 领域分类
- ✅ 观察次数

导出内容不包含：
- ❌ 实际代码片段
- ❌ 文件路径
- ❌ 会话记录
- ❌ 个人标识信息

## 命令参数

| 参数 | 说明 |
|------|------|
| `--domain <名称>` | 仅导出指定领域的经验 |
| `--min-confidence <值>` | 最小置信度阈值（默认：0.3） |
| `--output <文件>` | 输出文件路径（默认：instincts-export-YYYYMMDD.yaml） |
| `--format <yaml\|json\|md>` | 输出格式（默认：yaml） |
| `--include-evidence` | 包含依据文本（默认：不包含） |

## 常见领域分类

| 领域 | 说明 | 示例经验 |
|------|------|----------|
| `spring-boot` | Spring Boot 框架规范 | Service 层封装、DTO/VO 分离 |
| `mybatis` | MyBatis/MyBatis-Plus 规范 | #{} 参数绑定、Mapper 接口命名 |
| `security` | 安全最佳实践 | SQL 注入防护、XSS 防护 |
| `testing` | 测试规范 | TDD 工作流、JUnit 5 + Mockito |
| `git` | Git 工作流 | 约定式提交、分支规范 |
| `code-style` | 代码风格 | 命名规范、注释规范 |
| `workflow` | 工作流程 | 编辑代码前的搜索确认 |

## 导出场景示例

### 团队共享编码规范

```bash
# 导出高置信度的 Spring Boot 规范
/instinct-export --domain spring-boot --min-confidence 0.8 --output team-spring-boot-rules.yaml
```

### 项目交接

```bash
# 导出所有经验用于交接（使用当前日期）
/instinct-export --output project-handover-$(date +%Y%m%d).yaml
```

### 贡献到团队技能库

```bash
# 导出测试相关经验，转换为 MD 格式
/instinct-export --domain testing --format md --output testing-patterns.md
```

## 相关指令

- `/instinct-import` - 导入经验文件
- `/instinct-status` - 查看已学习的经验
- `/skill-create` - 从 Git 历史生成技能文件
