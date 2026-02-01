---
name: instinct-import
description: 从团队成员、技能生成器或其他来源导入经验文件
command: /instinct-import [文件路径或URL] [--dry-run] [--force] [--merge-strategy <higher|local|import>]
---

# 经验导入指令

从团队成员、技能生成器或其他来源导入经验文件。

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

## 可导入来源
- 团队成员导出的经验文件
- 技能生成器（仓库分析）
- 社区经验合集
- 旧机器备份

## 使用方法

```
/instinct-import team-instincts.yaml
/instinct-import https://github.com/org/repo/instincts.yaml
/instinct-import --from-skill-creator acme/webapp
```

## 执行步骤

1. 获取经验文件（本地路径或 URL）
2. 解析并验证格式
3. 检查与现有经验的重复情况
4. 合并或新增经验
5. 保存到 `~/.claude/homunculus/instincts/inherited/`

## 导入流程

```
📥 正在从 team-instincts.yaml 导入经验
================================================

找到 12 条待导入经验。

分析冲突中...

## 新增经验（8 条）
以下经验将被添加：
  ✓ use-zod-validation（置信度：0.7）
  ✓ prefer-named-exports（置信度：0.65）
  ✓ test-async-functions（置信度：0.8）
  ...

## 重复经验（3 条）
已存在相似经验：
  ⚠️ prefer-functional-style
     本地：置信度 0.8，观察 12 次
     导入：置信度 0.7
     → 保留本地（置信度更高）

  ⚠️ test-first-workflow
     本地：置信度 0.75
     导入：置信度 0.9
     → 更新为导入版本（置信度更高）

## 冲突经验（1 条）
与本地经验存在冲突：
  ❌ use-classes-for-services
     冲突于：avoid-classes
     → 跳过（需要手动处理）

---
导入 8 条新经验，更新 1 条，跳过 3 条？
```

## 合并策略

### 重复经验处理

导入与现有经验匹配的经验时：
- **高置信度优先**：保留置信度更高的版本
- **合并依据**：累加观察次数
- **更新时间戳**：标记为最近验证过

### 冲突经验处理

导入与现有经验相矛盾的经验时：
- **默认跳过**：不导入冲突经验
- **标记待审查**：将双方标记为需要关注
- **手动决策**：由用户决定保留哪个

## 来源追踪

导入的经验会附带以下标记：

```yaml
source: "inherited"
imported_from: "team-instincts.yaml"
imported_at: "2025-01-22T10:30:00Z"  # 使用实际导入时间戳
original_source: "session-observation"  # 或 "repo-analysis"
```

## 技能生成器集成

从技能生成器导入时：

```
/instinct-import --from-skill-creator acme/webapp
```

这将获取从仓库分析生成的经验：
- 来源：`repo-analysis`
- 初始置信度较高（0.7+）
- 关联到源仓库

## Java 技术栈经验示例

导入 Spring Boot + MyBatis-Plus 项目经验时（日期示例）：

```
## 新增经验（5 条）
  ✓ service-layer-isolation（置信度：0.9）
     "业务逻辑必须通过 Service 层，Controller 不可直接调用 Mapper"

  ✓ mybatis-param-binding（置信度：0.95）
     "MyBatis SQL 必须使用 #{} 参数绑定，禁止 ${}"

  ✓ dto-vo-separation（置信度：0.85）
     "接口返回使用 VO，参数接收使用 DTO，禁止直接返回 Entity"

  ✓ transactional-readonly（置信度：0.8）
     "查询方法必须添加 @Transactional(readOnly = true)"

  ✓ global-exception-handler（置信度：0.9）
     "异常统一由 @RestControllerAdvice 处理，禁止 try-catch 吞异常"
```

## 命令参数

| 参数 | 说明 |
|------|------|
| `--dry-run` | 预览但不实际导入 |
| `--force` | 即使存在冲突也强制导入 |
| `--merge-strategy <higher\|local\|import>` | 重复经验处理策略 |
| `--from-skill-creator <所有者/仓库>` | 从技能生成器分析结果导入 |
| `--min-confidence <值>` | 仅导入高于指定置信度的经验 |

## 输出结果

导入完成后：

```
✅ 导入完成！

新增：8 条经验
更新：1 条经验
跳过：3 条经验（2 条重复，1 条冲突）

新经验已保存至：~/.claude/homunculus/instincts/inherited/

运行 /instinct-status 查看所有经验。
```

## 相关指令

- `/instinct-export` - 导出经验文件
- `/instinct-status` - 查看已学习的经验
- `/skill-create` - 从 Git 历史生成技能文件