# 验证指令

对当前代码库状态执行全面验证。

## 说明

按此确切顺序执行验证：

1. **编译检查**
   - 执行 Maven 编译：`mvn clean compile`
   - 如果失败，报告错误并停止

2. **单元测试**
   - 执行所有单元测试：`mvn test`
   - 报告通过/失败数量
   - 报告覆盖率百分比

3. **集成测试**
   - 执行集成测试：`mvn verify`
   - 报告集成测试结果

4. **代码质量检查**
   - 执行 Checkstyle：`mvn checkstyle:check`
   - 执行 SpotBugs：`mvn spotbugs:check`
   - 报告警告和错误

5. **依赖安全扫描**
   - 执行 OWASP 依赖检查：`mvn dependency-check:check`
   - 报告发现的漏洞

6. **日志审计**
   - 在源文件中搜索 System.out.println
   - 在源文件中搜索 e.printStackTrace()
   - 报告位置

7. **敏感信息扫描**
   - 搜索硬编码的密码、密钥
   - 搜索 API Key、AccessKey
   - 报告位置

8. **Git 状态**
   - 显示未提交的变更
   - 显示上次提交后修改的文件

## 输出

生成简洁的验证报告：

```
验证：[通过/失败]

编译：    [OK/失败]
单元测试：  [X/Y 通过，Z% 覆盖率]
集成测试：  [X/Y 通过]
代码质量：  [OK/X 个问题]
依赖安全：  [OK/X 个漏洞]
日志审计：  [OK/X 个 System.out]
敏感信息：  [OK/X 个疑似]

准备好创建 PR：[是/否]
```

如果有任何关键问题，列出它们并提供修复建议。

## 参数

$ARGUMENTS 可以是：
- `quick` - 只检查编译 + 单元测试
- `full` - 所有检查（默认）
- `pre-commit` - 与提交相关的检查
- `pre-pr` - 完整检查加上安全扫描
- `ci` - CI 环境完整检查（包含依赖扫描）

## Maven 完整验证命令

```bash
# 快速验证（本地开发）
mvn clean compile test

# 完整验证（提交前）
mvn clean verify checkstyle:check spotbugs:check

# CI 完整验证
mvn clean verify dependency-check:check
```

## 常见问题修复

| 问题 | 修复命令 |
|------|---------|
| 编译失败 | `mvn clean compile` 查看详细错误 |
| 测试失败 | `mvn test -Dtest=类名#方法名` 定位问题 |
| 覆盖率不足 | 运行 `/test-coverage` 增加测试 |
| Checkstyle 警告 | `mvn checkstyle:checkstyle` 自动修复部分 |
| 依赖漏洞 | 更新依赖版本或添加忽略规则 |
