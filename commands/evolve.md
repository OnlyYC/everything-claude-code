---
name: evolve
description: 将关联经验聚合为技能、指令或智能体
command: true
---

# 经验聚合指令

## 实现方式

使用插件根目录运行经验 CLI 工具：

```bash
python3 "${CLAUDE_PLUGIN_ROOT}/skills/continuous-learning-v2/scripts/instinct-cli.py" evolve [--generate]
```

如果 `CLAUDE_PLUGIN_ROOT` 未设置（手动安装），使用：

```bash
python3 ~/.claude/skills/continuous-learning-v2/scripts/instinct-cli.py evolve [--generate]
```

分析经验并将相关的聚合为更高级的结构：
- **指令**：当经验描述用户主动调用的操作时
- **技能**：当经验描述自动触发的行为时
- **智能体**：当经验描述复杂的多步骤流程时

## 使用方法

```
/evolve                    # 分析所有经验并建议聚合方案
/evolve --domain testing   # 仅聚合测试领域的经验
/evolve --dry-run          # 预览将要创建的内容但不实际创建
/evolve --threshold 5      # 至少 5 条相关经验才聚合
```

## 聚合规则

### → 指令（用户主动调用）

当经验描述用户会明确请求的操作时：
- 多条经验描述「当用户要求……」
- 触发条件类似「当创建新的 X 时」
- 遵循可重复的步骤序列

示例：
- `new-table-step1`：「新增数据库表时，创建 Flyway 迁移脚本」
- `new-table-step2`：「新增数据库表时，更新 Entity 实体类」
- `new-table-step3`：「新增数据库表时，重新生成 MyBatis Mapper」

→ 生成：`/new-table` 指令

### → 技能（自动触发）

当经验描述应该自动发生的行为时：
- 模式匹配触发
- 错误处理响应
- 代码风格强制

示例：
- `prefer-service-layer`：「编写业务逻辑时，优先使用 Service 层」
- `dto-vo-separation`：「定义接口时，严格区分 DTO/VO/Entity」
- `transactional-readonly`：「查询方法添加 @Transactional(readOnly=true)」

→ 生成：`spring-boot-patterns` 技能

### → 智能体（需要深度隔离）

当经验描述复杂的多步骤流程，适合隔离处理时：
- 调试工作流
- 重构序列
- 研究任务

示例：
- `code-review-step1`：「代码审查时，先检查安全问题」
- `code-review-step2`：「代码审查时，检查并发问题」
- `code-review-step3`：「代码审查时，检查事务配置」
- `code-review-step4`：「代码审查时，检查异常处理」

→ 生成：`java-reviewer` 智能体

## 执行步骤

1. 从 `~/.claude/homunculus/instincts/` 读取所有经验
2. 按以下维度分组：
   - 领域相似性
   - 触发模式重叠
   - 动作序列关联
3. 对每组 3 条以上相关经验：
   - 确定聚合类型（指令/技能/智能体）
   - 生成对应文件
   - 保存到 `~/.claude/homunculus/evolved/{commands,skills,agents}/`
4. 将聚合结构关联回源经验

## 输出格式

```
🧬 聚合分析
==================

找到 3 组可聚合的经验：

## 聚合组 1：数据库表变更工作流
经验：new-table-migration, update-entity, regenerate-mapper
类型：指令
置信度：85%（基于 12 次观察）

将创建：/new-table 指令
文件：
  - ~/.claude/homunculus/evolved/commands/new-table.md

## 聚合组 2：Spring Boot 编码规范
经验：service-layer-isolation, dto-vo-separation, transactional-readonly, global-exception
类型：技能
置信度：78%（基于 8 次观察）

将创建：spring-boot-patterns 技能
文件：
  - ~/.claude/homunculus/evolved/skills/spring-boot-patterns.md

## 聚合组 3：Java 代码审查流程
经验：review-security, review-concurrency, review-transaction, review-exception
类型：智能体
置信度：72%（基于 6 次观察）

将创建：java-reviewer 智能体
文件：
  - ~/.claude/homunculus/evolved/agents/java-reviewer.md

---
运行 `/evolve --execute` 创建这些文件。
```

## Java 技术栈聚合示例

### 指令聚合示例：新增接口工作流

```
## 聚合组：新增 REST 接口工作流
相关经验：
  - define-dto-vo：「新增接口时，先定义 DTO 和 VO」
  - create-controller：「新增接口时，创建 Controller 类」
  - create-service：「新增接口时，创建 Service 接口和实现类」
  - create-mapper：「新增接口时，创建 MyBatis Mapper 接口」
  - write-test：「新增接口时，编写单元测试」

→ 生成：/new-api 指令
```

### 技能聚合示例：MyBatis 安全规范

```
## 聚合组：MyBatis 安全规范
相关经验：
  - use-hash-binding：「编写 SQL 时使用 #{} 参数绑定」
  - avoid-dollar-binding：「禁止使用 ${} 字符串拼接」
  - input-validation：「Mapper 接口参数必须校验」
  - sql-injection-check：「检查 SQL 注入风险」

→ 生成：mybatis-security 技能
```

### 智能体聚合示例：编译错误修复

```
## 聚合组：Java 编译错误修复
相关经验：
  - analyze-error：「编译失败时，先分析错误类型」
  - fix-import：「找不到符号时，检查 import 语句」
  - fix-dependency：「依赖缺失时，更新 pom.xml」
  - verify-fix：「修复后重新编译验证」

→ 生成：java-build-resolver 智能体
```

## 命令参数

| 参数 | 说明 |
|------|------|
| `--execute` | 实际创建聚合结构（默认为预览） |
| `--dry-run` | 预览但不创建 |
| `--domain <名称>` | 仅聚合指定领域的经验 |
| `--threshold <n>` | 聚合所需的最少经验数量（默认：3） |
| `--type <command\|skill\|agent>` | 仅创建指定类型 |

## 生成文件格式

### 指令
```markdown
---
name: new-table
description: 新增数据库表，包含迁移脚本、实体类和 Mapper 生成
command: /new-table
evolved_from:
  - new-table-migration
  - update-entity
  - regenerate-mapper
---

# 新增数据库表指令

[基于聚合经验生成的内容]

## 步骤
1. 创建 Flyway 迁移脚本
2. 更新 Entity 实体类
3. 生成 MyBatis Mapper 接口
4. 编写单元测试
```

### 技能
```markdown
---
name: spring-boot-patterns
description: 强制执行 Spring Boot 编码规范
evolved_from:
  - service-layer-isolation
  - dto-vo-separation
  - transactional-readonly
---

# Spring Boot 编码规范技能

[基于聚合经验生成的内容]
```

### 智能体
```markdown
---
name: java-reviewer
description: 系统化的 Java 代码审查智能体
model: sonnet
evolved_from:
  - review-security
  - review-concurrency
  - review-transaction
---

# Java 代码审查智能体

[基于聚合经验生成的内容]
```

## 相关指令

- `/instinct-status` - 查看所有经验
- `/instinct-export` - 导出经验文件
- `/skill-create` - 从 Git 历史生成技能文件