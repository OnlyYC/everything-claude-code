---
name: refactor-clean
description: 通过测试验证安全地识别和移除无用代码
command: /refactor-clean
---

# 重构清理

通过测试验证安全地识别和移除无用代码：

1. 执行无用代码分析工具：
   - **SpotBugs**：检测未使用的类、方法和字段
   - **PMD**：代码质量检查，找出未使用的代码
   - **JaCoCo**：基于覆盖率报告找出未覆盖代码
   - **SonarQube**：全面的技术债务分析（可选）

2. 在 `.reports/dead-code-analysis.md` 生成完整报告

3. 按风险分类发现：
   - **安全**：测试类、废弃的工具类
   - **注意**：Controller、Service、Mapper
   - **危险**：配置类、主入口类、被反射调用的类

4. 只建议安全的删除

5. 每次删除前：
   - 执行完整测试套件
   - 验证测试通过
   - 应用更改
   - 重新执行测试
   - 如果测试失败则回滚

6. 显示已清理项目的摘要

## 无用代码检测工具

```bash
# Maven 清理检查（跨平台通用）
mvn clean dependency:analyze

# 查找未使用的依赖
mvn dependency:analyze

# SpotBugs 死代码检测
mvn spotbugs:check

# PMD 代码质量检查
mvn pmd:check

# 基于覆盖率查找未使用代码
mvn test jacoco:report
# 查看报告，找出 0% 覆盖的代码
```

## 常见可清理的项目

| 类型 | 说明 | 风险 |
|------|------|------|
| 未使用的 import | 无效导入语句 | 低 |
| 未使用的私有方法 | 类内私有但未被调用 | 低 |
| 未使用的私有字段 | 类内私有但未被引用 | 低 |
| 未使用的类 | 整个类未被引用 | 中 |
| 未使用的依赖 | pom.xml 中声明但未使用 | 中 |
| 废弃的方法 | 标注 @Deprecated 的方法 | 中 |
| 整个废弃模块 | 功能已被替代 | 高 |

## 清理顺序建议

1. 先清理未使用的 import（安全）
2. 再清理未使用的私有方法和字段（相对安全）
3. 然后清理未使用的类（需要仔细检查）
4. 最后清理未使用的依赖（需要测试验证）

## 自动清理

```bash
# Java 自动格式化和优化（跨平台通用）
mvn fmt:format
```

### IDE 自动优化功能
- **IntelliJ IDEA**: Code → Optimize Imports
- **Eclipse**: Source → Organize Imports
- **VS Code**: 右键 → Format Document / Organize Imports

在执行测试前绝不删除代码！

## 相关指令

- `/test-coverage` - 分析测试覆盖率
- `/code-review` - 审查代码质量
- `/verify` - 执行完整验证循环
