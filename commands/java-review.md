---
description: Comprehensive Java code review for Spring Boot patterns, concurrency safety, error handling, and security. Invokes java-reviewer agent.
---

# Java 代码审查

此指令调用 **java-reviewer** Agent 进行全面的 Java 特定代码审查。

## 此指令的功能

1. **识别 Java 变更**：通过 `git diff` 找出修改的 `.java` 文件
2. **执行静态分析**：运行 Checkstyle、SpotBugs 和 PMD
3. **安全性扫描**：检查 SQL 注入、XSS、权限绕过
4. **并发审查**：分析线程安全性、synchronized 使用、并发集合
5. **生成报告**：按严重性分类问题

## 何时使用

在以下情况使用 `/java-review`：
- 撰写或修改 Java 代码后
- 提交 Java 变更前
- 审查包含 Java 代码的 PR
- 加入新的 Java 代码仓库时
- 学习 Java 最佳实践

## 审查类别

### 致命（必须修复）

- **SQL 注入**：MyBatis ${} 拼接、JDBC 字符串拼接
  ```java
  // 错误
  @Select("SELECT * FROM users WHERE id = ${userId}")
  User getUserById(String userId);

  // 正确
  @Select("SELECT * FROM users WHERE id = #{userId}")
  User getUserById(String userId);
  ```

- **XSS 漏洞**：未转义的用户输入
- **硬编码密钥**：密码、API Key 直接写在代码中
- **权限绕过**：缺少 @PreAuthorize 验证
- **命令注入**：Runtime.exec() 使用用户输入

### 高（应该修复）

- **缺少异常处理**：吞掉异常或空 catch 块
  ```java
  // 错误
  try {
      // ...
  } catch (Exception e) {
      // 什么也不做
  }

  // 正确
  try {
      // ...
  } catch (Exception e) {
      log.error("操作失败", e);
      throw new BusinessException("操作失败", e);
  }
  ```

- **NPE 风险**：未做空值判断
- **资源泄漏**：Stream、Connection 未使用 try-with-resources
- **事务配置错误**：@Transactional 使用不当
- **日志输出敏感信息**：密码、Token 记录到日志

### 中（考虑）

- **违反单一职责原则**：类或方法职责过多
- **缺少 Javadoc**：公共 API 缺少文档注释
- **魔法值**：未定义常量的字符串或数字
- **过度使用 @Autowired**：推荐构造器注入
- **方法过长**：方法超过 50 行
- **深层嵌套**：嵌套层级超过 4 层
- **代码格式不一致**：缩进、命名风格

## 执行的自动化检查

```bash
# 编译检查
mvn clean compile

# 静态分析
mvn checkstyle:check
mvn spotbugs:check
mvn pmd:check

# 依赖安全扫描
mvn dependency-check:check
```

## Spring Boot 特定检查

- Controller 直接返回实体（应使用 DTO/VO）
- 缺少统一异常处理（@RestControllerAdvice）
- 缺少统一响应封装（Result<T>）
- @Transactional 修饰 private 方法
- 缺少入参校验（@Valid、@Validated）
- 循环依赖问题

## 并发安全检查

- 实例变量使用不当（线程安全）
- synchronized 使用不当
- 并发集合使用错误
- 线程池配置不当
- @Async 使用不当

## 批准标准

| 状态 | 条件 |
|------|------|
| ✅ 批准 | 没有致命或高优先问题 |
| ⚠️ 警告 | 仅中优先问题（可谨慎合并） |
| ❌ 阻挡 | 发现致命或高优先问题 |

## 相关指令

- 先使用 `/tdd` 确保测试通过
- 如果发生构建错误，使用 `/java-build`
- 提交前使用 `/java-review`
- 对通用问题使用 `/code-review`

## 相关

- Agent：`agents/java-reviewer.md`
- 技能：`skills/java-patterns/`、`skills/spring-boot-patterns/`
