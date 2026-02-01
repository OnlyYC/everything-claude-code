---
name: eval-harness
description: Eval 驱动开发（EDD）框架：适配 Java 21 + Spring Boot 3 + Spring MVC + MyBatis-Plus + Maven + MySQL 技术栈的评估框架
version: 1.1.0
tech_stack: [Java 21, Spring Boot 3, MyBatis-Plus, MySQL, Maven]
related_skills: [tdd-workflow, springboot-tdd, java-testing, verification-loop]
tools: Read, Write, Edit, Bash, Grep, Glob
---

# Eval Harness 技能

Eval 驱动开发（EDD）框架，适配 Java 21 + Spring Boot 3 + Spring MVC + MyBatis-Plus + Maven + MySQL 技术栈。

## 理念

Eval 驱动开发将 evals 视为"AI 开发的单元测试"：
- 在实现前定义预期行为
- 开发期间持续执行 evals
- 每次变更追踪回归
- 使用 pass@k 指标进行可靠性测量

## Eval 类型

### 能力 Evals

测试 Claude 是否能做到以前做不到的事：

```markdown
[CAPABILITY EVAL: feature-name]
任务：功能描述
成功标准：
  - [ ] 标准 1
  - [ ] 标准 2
  - [ ] 标准 3
预期输出：预期结果描述
```

### 回归 Evals

确保变更不会破坏现有功能：

```markdown
[REGRESSION EVAL: feature-name]
基准：Git SHA 或检查点名称
测试：
  - UserServiceTest#testCreateUser: PASS/FAIL
  - OrderControllerTest#testCreateOrder: PASS/FAIL
  - UserMapperTest#testSelectByCondition: PASS/FAIL
结果：X/Y 通过（先前为 Y/Y）
```

## 评分器类型

### 1. 基于代码的评分器（自动验证）

使用代码的确定性检查：

```bash
# 检查文件是否包含预期模式
grep -q "public class UserService" src/main/java/com/example/service/UserService.java && echo "PASS" || echo "FAIL"

# 检查测试是否通过
mvn test -Dtest=UserServiceTest && echo "PASS" || echo "FAIL"

# 检查构建是否成功
mvn clean compile && echo "PASS" || echo "FAIL"
```

### 2. 基于模型的评分器（AI 评估）

使用 Claude 评估开放式输出：

```markdown
[MODEL GRADER PROMPT]
评估以下 Java 代码变更：
1. 是否符合 Spring Boot 3 最佳实践？
2. 是否正确使用 MyBatis-Plus？
3. 事务注解 @Transactional 使用是否正确？
4. 异常处理是否恰当？
5. 是否符合 Java 21 语法特性？

分数：1-5（1=差，5=优秀）
理由：[解释]
```

### 3. 人工评分器（手动审查）

标记为手动审查：

```markdown
[HUMAN REVIEW REQUIRED]
变更：变更内容描述
理由：为何需要人工审查
风险等级：LOW/MEDIUM/HIGH
```

## 指标

### pass@k

"k 次尝试中至少一次成功"
- pass@1：第一次尝试成功率
- pass@3：3 次尝试内成功
- 典型目标：pass@3 > 90%

### pass^k

"所有 k 次试验都成功"
- 更高的可靠性标准
- pass^3：连续 3 次成功
- 用于关键路径

## Eval 工作流程

### 1. 定义（编码前）

```markdown
## EVAL 定义：user-management

### 能力 Evals
1. 可以创建新用户账户
2. 可以验证邮箱格式
3. 可以安全地哈希密码
4. 可以查询用户列表
5. 可以更新用户信息

### 回归 Evals
1. 现有登录仍可运作
2. 会话管理未变更
3. 其他业务模块未受影响

### 成功指标
- 能力 evals 的 pass@3 > 90%
- 回归 evals 的 pass^3 = 100%
- 测试覆盖率 > 80%
```

### 2. 实现

编写代码以通过定义的 evals。

### 3. 评估

```bash
# 执行能力 evals
mvn test -Dtest=*UserTest

# 执行回归 evals
mvn test -Dtest=*IntegrationTest

# 生成覆盖率报告
mvn jacoco:report
```

### 4. 报告

```markdown
EVAL 报告：user-management
========================

能力 Evals：
  create-user:     PASS (pass@1)
  validate-email:  PASS (pass@2)
  hash-password:   PASS (pass@1)
  list-users:      PASS (pass@1)
  update-user:     PASS (pass@3)
  整体：           5/5 通过

回归 Evals：
  login-flow:      PASS
  session-mgmt:    PASS
  other-modules:   PASS
  整体：           3/3 通过

指标：
  pass@1: 80% (4/5)
  pass@3: 100% (5/5)
  覆盖率： 85%

状态：准备审查
```

## 整合模式

### 实现前

```
/eval define feature-name
```

在 `.claude/evals/feature-name.md` 创建 eval 定义文件。

### 实现期间

```
/eval check feature-name
```

执行当前 evals 并报告状态。

### 实现后

```
/eval report feature-name
```

生成完整 eval 报告。

## Eval 储存

在项目中储存 evals：

```
.claude/
  evals/
    user-management.md      # Eval 定义
    user-management.log     # Eval 执行历史
    baseline.json           # 回归基准
```

## 最佳实践

1. **编码前定义 evals** - 强制清楚思考成功标准
2. **频繁执行 evals** - 及早捕捉回归
3. **随时间追踪 pass@k** - 监控可靠性趋势
4. **优先使用代码评分器** - 确定性 > 概率性
5. **安全性需人工审查** - 永远不要完全自动化安全检查
6. **保持 evals 快速** - 慢 evals 不会被执行
7. **与代码一起版本化 evals** - Evals 是一等工件

## 示例：用户管理模块

```markdown
## EVAL：user-management

### 阶段 1：定义

能力 Evals：
- [ ] 用户可以用邮箱/密码注册
- [ ] 邮箱格式验证正确
- [ ] 密码使用 BCrypt 哈希
- [ ] 用户信息正确保存到数据库
- [ ] 可以分页查询用户列表

回归 Evals：
- [ ] 现有 API 端点正常工作
- [ ] 数据库 schema 兼容
- [ ] 其他业务模块不受影响

### 阶段 2：实现

创建以下文件：
- Entity: UserEntity.java
- Mapper: UserMapper.java (MyBatis-Plus)
- Service: UserService.java
- Controller: UserController.java
- DTO: CreateUserDTO.java, UserVO.java
- Tests: UserServiceTest.java, UserControllerTest.java

### 阶段 3：评估

执行：mvn clean test

### 阶段 4：报告

EVAL 报告：user-management
==========================

能力：5/5 通过（pass@3：100%）
回归：3/3 通过（pass^3：100%）
覆盖率：88%

状态：准备合并
```

## 快速命令参考

### Maven 测试命令

```bash
# 运行所有测试（Windows/macOS/Linux 通用）
mvn test

# 运行特定测试类
mvn test -Dtest=UserServiceTest

# 运行特定测试方法
mvn test -Dtest=UserServiceTest#shouldCreateUser

# 跳过测试
mvn -DskipTests

# 生成测试报告
mvn surefire-report:report

# 生成覆盖率报告
mvn jacoco:report

# 查看测试结果摘要（Unix-like systems only）
mvn test | grep -E "(Tests run:|BUILD SUCCESS|BUILD FAILURE)"

# 查看测试结果摘要（Windows PowerShell）
mvn test | Select-String -Pattern "Tests run:|BUILD"
```

### 代码检查命令

```bash
# 编译检查
mvn compile

# 静态分析
mvn spotbugs:check
mvn checkstyle:check
mvn pmd:check

# 依赖检查
mvn org.owasp:dependency-check-maven:check

# 完整验证
mvn clean verify
```

## 项目结构参考

```
src/
├── main/
│   ├── java/
│   │   └── com/
│   │       └── example/
│   │           ├── controller/    # Controller 层
│   │           ├── service/       # Service 层
│   │           ├── mapper/        # MyBatis Mapper
│   │           ├── entity/        # 实体类
│   │           ├── dto/           # 数据传输对象
│   │           ├── vo/            # 视图对象
│   │           ├── config/        # 配置类
│   │           └── Application.java
│   └── resources/
│       ├── mapper/                # MyBatis XML
│       ├── application.yml
│       └── application-test.yml
└── test/
    └── java/
        └── com/
            └── example/
                ├── controller/
                ├── service/
                └── mapper/
```

**记住**：快速反馈胜过后期意外。在生产系统中将警告视为缺陷。保持高标准的代码质量和测试覆盖率。

## 评分示例

### 能力 Eval 评分示例

```markdown
## EVAL：用户注册功能

### 能力 Evals
1. 邮箱格式验证正确性 - 权重 20%
2. 密码强度检查 - 权重 20%
3. 数据持久化成功 - 权重 30%
4. 返回正确的用户数据 - 权重 20%
5. 错误处理完善 - 权重 10%

评分标准：
- 9-10分：所有功能正确实现，错误处理完善
- 7-8分：核心功能正确，部分边界情况处理不足
- 5-6分：基本功能可用，存在明显缺陷
- 0-4分：功能不完整或无法正常工作
```

### 回归 Eval 评分示例

```markdown
## EVAL：订单支付功能回归测试

### 回归 Evals
- OrderServiceTest#testCreatePayment: PASS
- OrderControllerTest#testPaymentAPI: PASS
- PaymentIntegrationTest#testPaymentFlow: PASS
- OrderMapperTest#testInsertOrder: PASS

评分项：
1. 所有测试通过 - 权重 40%
2. 无性能回归 - 权重 20%
3. 代码覆盖率未下降 - 权重 20%
4. 无新增安全漏洞 - 权重 20%

评分标准：
- 9-10分：所有测试通过，无回归问题
- 7-8分：核心功能通过，有轻微问题
- 5-6分：部分测试失败，需要修复
- 0-4分：严重回归，需要立即修复
```

## 相关技能

- `tdd-workflow` - 测试驱动开发流程
- `java-testing` - JUnit 5 + Mockito 测试指南
- `springboot-tdd` - Spring Boot TDD 方法论
- `verification-loop` - 完整项目验证流程
