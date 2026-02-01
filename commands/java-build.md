---
description: Fix Java compilation errors, build issues, and dependency problems incrementally. Invokes java-build-resolver agent for minimal, surgical fixes.
---

# Java 编译与修复

此指令调用 **java-build-resolver** Agent，以最小变更增量修复 Java 编译错误。

## 此指令的功能

1. **执行诊断**：运行 `mvn clean compile`、`mvn dependency:tree`
2. **解析错误**：按文件分组并按严重性排序
3. **增量修复**：一次一个错误
4. **验证每次修复**：每次变更后重新编译
5. **报告摘要**：显示已修复和剩余问题

## 何时使用

在以下情况使用 `/java-build`：
- `mvn clean compile` 失败并出现编译错误
- `mvn dependency:tree` 显示依赖冲突
- `mvn test` 因编译问题失败
- 拉取破坏构建的变更后

## 执行的诊断命令

```bash
# 主要编译检查
mvn clean compile

# 依赖分析
mvn dependency:tree
mvn dependency:analyze

# 跳过测试编译
mvn clean compile -DskipTests

# 下载依赖
mvn dependency:resolve
```

## 常见修复的错误

| 错误 | 典型修复 |
|------|----------|
| `cannot find symbol` | 新增 import 或修正类名拼写 |
| `package X does not exist` | 新增 Maven 依赖到 pom.xml |
| `incompatible types` | 类型转换或修正泛型声明 |
| `missing return statement` | 新增 return 语句 |
| `method X in class Y cannot be applied` | 修正方法参数类型或数量 |
| `cyclic inheritance` | 重构类继承关系 |
| `IOException, FileNotFoundException` | 新增 throws 声明或 try-catch |
| `Diamond operator` | 修正泛型实例化 |
| `variable X might not have been initialized` | 初始化变量 |

## 依赖问题修复

| 错误 | 典型修复 |
|------|----------|
| `Missing artifact` | 检查仓库配置或更新版本 |
| `Dependency convergence` | 统一依赖版本 |
| `Circular dependency` | 重构模块结构 |
| `Snapshot not found` | 检查私服仓库或清理本地缓存 |

## 修复策略

1. **编译错误优先** - 代码必须编译
2. **依赖问题其次** - 修复 Maven 依赖
3. **警告第三** - 风格和最佳实践
4. **一次一个修复** - 验证每次变更

## 停止条件

Agent 会在以下情况停止并报告：
- 3 次尝试后同样错误仍存在
- 修复引入更多错误
- 需要超出范围的架构变更
- 需要手动安装的缺少外部依赖

## 输出格式

每次修复尝试后：

```text
[已修复] src/main/java/com/example/UserService.java:42
错误：cannot find symbol: UserService
修复：新增 import com.example.service.UserService
```

最终摘要：

```text
构建状态：成功/失败
已修复编译错误：N
已修复依赖问题：N
已修改文件：列表
剩余问题：列表（如果有）
```

## Maven 常用命令

```bash
# 清理并编译
mvn clean compile

# 查看依赖树
mvn dependency:tree

# 分析未使用的依赖
mvn dependency:analyze

# 强制更新快照
mvn clean install -U

# 跳过测试
mvn clean package -DskipTests

# 查看有效 POM
mvn help:effective-pom
```

## 相关指令

- `/tdd` - 编译成功后编写测试
- `/code-review` - 审查代码质量
- `/verify` - 执行完整验证循环

## 相关

- Agent：`agents/java-build-resolver.md`
- 技能：`skills/java-patterns/`
