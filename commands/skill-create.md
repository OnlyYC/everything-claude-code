---
name: skill-create
description: 分析本地 Git 提交历史，提取代码模式并生成 SKILL.md 技能文件。Skill Creator GitHub 应用的本地版本。
allowed_tools: ["Bash", "Read", "Write", "Grep", "Glob"]
---

# /skill-create - 本地技能生成

分析项目的 Git 提交历史，提取编码规范与模式，生成可供 Claude 学习的 SKILL.md 技能文件。

## 使用方法

```bash
/skill-create                    # 分析当前仓库
/skill-create --commits 100      # 分析最近 100 次提交
/skill-create --output ./skills  # 指定输出目录
/skill-create --instincts        # 同时生成 continuous-learning-v2 的经验文件
```

## 功能说明

1. **解析 Git 历史** - 分析提交记录、文件变更和代码模式
2. **识别规范模式** - 识别重复的工作流和编码约定
3. **生成 SKILL.md** - 创建符合 Claude Code 规范的技能文件
4. **可选生成经验文件** - 用于 continuous-learning-v2 系统

## 分析步骤

### 第一步：收集 Git 数据

```bash
# 获取最近的提交及文件变更
git log --oneline -n ${COMMITS:-200} --name-only --pretty=format:"%H|%s|%ad" --date=short

# 按文件统计提交频率
git log --oneline -n 200 --name-only | grep -v "^$" | grep -v "^[a-f0-9]" | sort | uniq -c | sort -rn | head -20

# 获取提交信息模式
git log --oneline -n 200 | cut -d' ' -f2- | head -50
```

### 第二步：识别模式

查找以下模式类型：

| 模式 | 检测方法 |
|------|---------|
| **提交规范** | 正则匹配提交信息（feat:、fix:、chore:） |
| **文件联动变更** | 总是一起修改的文件 |
| **工作流序列** | 重复的文件变更模式 |
| **架构风格** | 目录结构和命名约定 |
| **测试规范** | 测试文件位置、命名、覆盖率 |

### 第三步：生成 SKILL.md

输出格式：

```markdown
---
name: {仓库名}-patterns
description: 从 {仓库名} 提取的编码模式
version: 1.0.0
source: local-git-analysis
analyzed_commits: {数量}
---

# {仓库名} 编码规范

## 提交规范
{检测到的提交信息模式}

## 代码架构
{检测到的目录结构和组织方式}

## 工作流程
{检测到的重复文件变更模式}

## 测试规范
{检测到的测试约定}
```

### 第四步：生成经验文件（如果指定 --instincts）

用于 continuous-learning-v2 集成：

```yaml
---
id: {仓库}-commit-convention
trigger: "编写提交信息时"
confidence: 0.8
domain: git
source: local-repo-analysis
---

# 遵循约定式提交规范

## 操作
提交信息前缀使用：feat:、fix:、chore:、docs:、test:、refactor:

## 依据
- 分析了 {n} 次提交
- {百分比}% 遵循约定式提交格式
```

## 输出示例

在 Java + Spring Boot 项目上运行 `/skill-create` 可能生成：

```markdown
---
name: my-app-patterns
description: 从 my-app 仓库提取的编码模式
version: 1.0.0
source: local-git-analysis
analyzed_commits: 150
---

# My App 编码规范

## 提交规范

本项目遵循 **约定式提交** 规范：
- `feat:` - 新功能
- `fix:` - 缺陷修复
- `chore:` - 构建/工具链维护
- `docs:` - 文档更新
- `refactor:` - 代码重构（不改变功能）
- `test:` - 测试相关

## 代码架构

```
src/main/java/com/example/
├── controller/      # 控制器层（*Controller.java）
├── service/         # 业务逻辑层（*Service.java、*ServiceImpl.java）
├── mapper/          # MyBatis Mapper 接口（*Mapper.java）
├── model/           # 数据模型
│   ├── entity/      # 数据库实体（*DO.java、*PO.java）
│   ├── dto/         # 数据传输对象（*DTO.java、*Req.java、*Resp.java）
│   └── vo/          # 视图对象（*VO.java）
├── config/          # 配置类（*Config.java、*Properties.java）
├── common/          # 公共模块
│   ├── constant/    # 常量定义
│   ├── enums/       # 枚举类
│   └── exception/   # 自定义异常
└── util/            # 工具类

src/main/resources/
├── mapper/          # MyBatis XML 映射文件（*Mapper.xml）
├── application.yml  # 主配置文件
└── application-dev.yml # 开发环境配置
```

## 工作流程

### 新增接口流程
1. 在 `model/vo` 下定义请求/响应 VO
2. 在 `controller` 下创建或修改控制器
3. 在 `service` 下编写业务接口和实现
4. 在 `mapper` 下编写 MyBatis-Plus Mapper 接口
5. 可选：在 `resources/mapper` 下编写自定义 XML
6. 编写单元测试

### 数据库表变更流程
1. 修改 SQL 脚本（`db/migration/` 或 `db/schema.sql`）
2. 更新对应的 Entity 实体类
3. 同步更新 Mapper.xml
4. 执行数据库迁移：`mvn mybatis-flyway:migrate` 或手动执行脚本
5. 编写集成测试验证

## 测试规范

- 测试文件位置：`src/test/java`，与主代码包结构一致
- 命名约定：`{类名}Test.java`
- 覆盖率目标：80%+
- 单元测试框架：JUnit 5 + Mockito
- 集成测试框架：Spring Boot Test + @SpringBootTest

## 依赖管理

- 父 POM：`spring-boot-starter-parent`
- Java 版本：21
- Spring Boot 版本：3.x
- MyBatis-Plus：3.5.x
- 数据库驱动：mysql-connector-j
```

## GitHub 应用集成

如需更高级功能（1万+ 提交分析、团队共享、自动 PR），请使用 [Skill Creator GitHub 应用](https://github.com/apps/skill-creator)：

- 安装地址：[github.com/apps/skill-creator](https://github.com/apps/skill-creator)
- 在任意 Issue 下评论 `/skill-creator analyze`
- 接收包含生成技能文件的 PR

## 相关指令

- `/instinct-import` - 导入生成的经验文件
- `/instinct-status` - 查看已学习的经验
- `/evolve` - 将经验聚合为技能/智能体

---

*本指令属于 [Everything Claude Code](https://github.com/affaan-m/everything-claude-code) 项目*
