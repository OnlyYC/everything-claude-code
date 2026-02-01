---
name: code-review
description: 对未提交变更进行全面的安全性和质量审查
command: /code-review
---

# 代码审查

对未提交变更进行全面的安全性和质量审查：

1. 获取变更的文件：`git diff --name-only HEAD`

2. 对每个变更的文件，检查：

**安全问题（致命）：**
- 硬编码的数据库密码、API 密钥、AccessKey
- SQL 注入漏洞（MyBatis ${} 拼接、JDBC 字符串拼接）
- XSS 漏洞（未转义的模板输出）
- 缺少入参校验（@Valid、@Validated）
- 不安全的依赖（Log4j2、Fastjson 等已知漏洞）
- 路径遍历风险（用户控制的文件路径）
- CSRF 漏洞（缺少 CSRF Token）
- 权限绕过（缺少 @PreAuthorize）
- 敏感信息日志输出（密码、Token）

**代码质量（高）：**
- 方法超过 50 行
- 类超过 500 行
- 深层嵌套（> 4 层）
- 缺少异常处理（try-catch 空体或吞掉异常）
- System.out.println 或 e.printStackTrace() 语句
- TODO/FIXME 注释
- Controller/Service/Mapper 缺少 Javadoc
- NPE 风险（未做空值判断）
- 资源未关闭（Stream、Connection 未使用 try-with-resources）

**最佳实践（中）：**
- 违反单一职责原则
- 代码中包含表情符号
- 新代码缺少单元测试
- 魔法值（未定义常量的字符串或数字）
- 代码格式不一致
- 过度使用 @Autowired（推荐构造器注入）
- 事务传播行为配置不当
- 缺少日志记录（关键业务流程）

**Spring Boot/MyBatis 规范（中）：**
- Controller 直接返回实体（应使用 DTO/VO）
- 缺少统一异常处理（@RestControllerAdvice）
- 缺少统一响应封装（Result<T>）
- Mapper XML 中使用 ${} 而非 #{}
- 缺少分页参数校验
- 缺少接口版本控制

3. 产生报告，包含：
   - 严重性：致命、高、中、低
   - 文件位置和行号
   - 问题描述
   - 建议修复

4. 如果发现致命或高优先问题则阻拦提交

## 静态分析工具

```bash
# Maven 编译检查（跨平台通用）
mvn clean compile

# Checkstyle 代码风格检查（跨平台通用）
mvn checkstyle:check

# SpotBugs 缺陷检测（跨平台通用）
mvn spotbugs:check

# PMD 代码质量检查（跨平台通用）
mvn pmd:check

# OWASP Dependency Check 依赖漏洞扫描（跨平台通用）
mvn dependency-check:check
```

绝不批准有安全漏洞的代码！

## 相关指令

- `/java-review` - Java 特定代码审查
- `/verify` - 执行完整验证循环
- `/build-fix` - 修复发现的问题
