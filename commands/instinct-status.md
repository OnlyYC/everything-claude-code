---
name: instinct-status
description: 显示所有已学习的经验及其置信度
command: /instinct-status [--domain <名称>] [--low-confidence] [--high-confidence] [--source <类型>] [--json]
---

# 经验状态指令

按领域分组显示所有已学习的经验及其置信度分数。

## 前置条件

需要安装 **continuous-learning-v2** 技能，确保以下 CLI 工具可用：

**Windows (PowerShell):**
```powershell
# 检查工具是否安装
python3 "$env:CLAUDE_PLUGIN_ROOT\skills\continuous-learning-v2\scripts\instinct-cli.py" --help

# 或使用默认路径
python3 ~/.claude/skills/continuous-learning-v2/scripts/instinct-cli.py --help
```

**macOS/Linux:**
```bash
# 检查工具是否安装
python3 "${CLAUDE_PLUGIN_ROOT}/skills/continuous-learning-v2/scripts/instinct-cli.py" --help

# 或使用默认路径
python3 ~/.claude/skills/continuous-learning-v2/scripts/instinct-cli.py --help
```

如果未安装，请先安装 continuous-learning-v2 技能。

## 使用方法

```
/instinct-status
/instinct-status --domain spring-boot
/instinct-status --low-confidence
```

## 执行步骤

1. 从 `~/.claude/homunculus/instincts/personal/` 读取个人经验
2. 从 `~/.claude/homunculus/instincts/inherited/` 读取继承经验
3. 按领域分组显示，并附置信度条形图

## 输出格式

```
📊 经验状态
==================

## Spring Boot 规范（4 条经验）

### service-layer-isolation
触发时机：编写业务逻辑时
动作：业务逻辑必须通过 Service 层，Controller 不可直接调用 Mapper
置信度：████████░░ 80%
来源：session-observation | 最后更新：2024-01-15  # 示例日期

### dto-vo-separation
触发时机：定义接口参数时
动作：严格区分 DTO/VO/Entity，禁止直接返回 Entity
置信度：██████░░░░ 60%
来源：repo-analysis (github.com/acme/webapp)

## MyBatis 规范（2 条经验）

### mybatis-param-binding
触发时机：编写 MyBatis SQL 时
动作：使用 #{} 参数绑定，避免 ${} SQL 注入风险
置信度：█████████░ 90%
来源：session-observation

### mapper-interface-naming
触发时机：创建 MyBatis Mapper 时
动作：Mapper 接口命名以 *Mapper 结尾，XML 文件名与接口一致
置信度：███████░░░ 70%
来源：repo-analysis

## 测试规范（3 条经验）

### test-first-workflow
触发时机：新增功能时
动作：遵循 TDD 原则：先写测试，再实现功能
置信度：█████████░ 90%
来源：session-observation

### junit5-mockito
触发时机：编写单元测试时
动作：使用 JUnit 5 + Mockito，避免 JUnit 4
置信度：████████░░ 80%
来源：session-observation

---
总计：9 条经验（4 条个人，5 条继承）
观察器：运行中（上次分析：5 分钟前）
```

## Java 技术栈经验领域

### Spring Boot 规范

| 经验 ID | 触发时机 | 动作 |
|---------|----------|------|
| `service-layer-isolation` | 编写业务逻辑 | Controller 不可直接调用 Mapper |
| `dto-vo-separation` | 定义接口参数 | 禁止直接返回 Entity |
| `transactional-readonly` | 编写查询方法 | 添加 `@Transactional(readOnly = true)` |
| `global-exception-handler` | 处理异常 | 统一使用 `@RestControllerAdvice` |
| `validation-first` | 接收参数 | 使用 `@Valid` 或 `@Validated` 校验 |

### MyBatis 规范

| 经验 ID | 触发时机 | 动作 |
|---------|----------|------|
| `mybatis-param-binding` | 编写 SQL | 使用 `#{}` 参数绑定 |
| `mapper-xml-location` | 创建 Mapper.xml | 放在 `resources/mapper/` 目录 |
| `mapper-interface-naming` | 创建 Mapper 接口 | 以 `*Mapper` 结尾 |

### 安全规范

| 经验 ID | 触发时机 | 动作 |
|---------|----------|------|
| `sql-injection-prevention` | 编写 SQL | 禁止 `${}` 字符串拼接 |
| `sensitive-data-logging` | 打印日志 | 禁止输出密码、Token |
| `password-encryption` | 存储密码 | 使用 BCrypt 加密 |

### 测试规范

| 经验 ID | 触发时机 | 动作 |
|---------|----------|------|
| `test-first-workflow` | 新增功能 | 遵循 TDD 原则 |
| `junit5-mockito` | 编写单元测试 | 使用 JUnit 5 + Mockito |
| `coverage-target` | 完成功能 | 确保覆盖率 80%+ |

## 命令参数

| 参数 | 说明 |
|------|------|
| `--domain <名称>` | 按领域过滤（spring-boot、mybatis、testing 等） |
| `--low-confidence` | 仅显示置信度 < 0.5 的经验 |
| `--high-confidence` | 仅显示置信度 >= 0.7 的经验 |
| `--source <类型>` | 按来源过滤（session-observation、repo-analysis、inherited） |
| `--json` | 以 JSON 格式输出，便于程序处理 |

## 输出等级说明

| 置信度 | 等级 | 说明 |
|--------|------|------|
| 90%+ | ██████████ | 高度可信，强烈建议遵循 |
| 70-89% | ███████░░░ | 较为可信，建议遵循 |
| 50-69% | █████░░░░░ | 中等可信，可选择性遵循 |
| < 50% | ███░░░░░░░ | 低可信，需谨慎评估 |

## 相关指令

- `/instinct-export` - 导出经验文件
- `/instinct-import` - 导入经验文件
- `/skill-create` - 从 Git 历史生成技能文件