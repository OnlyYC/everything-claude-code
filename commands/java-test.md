---
name: java-test
description: 强制执行 Java TDD 工作流。先编写 JUnit 5 测试，再实现。确保 80%+ 覆盖率
command: /java-test
---

# Java 单元测试指令

此指令强制执行 Java 代码的测试驱动开发（TDD）方法论，使用 JUnit 5 + Mockito 最佳实践。

## 此指令的功能

1. **定义接口/方法签名骨架**：先创建方法签名
2. **先编写测试**：编写失败的 JUnit 5 测试（RED）
3. **执行测试**：验证测试因正确的原因失败
4. **实现最小代码**：编写刚好足以通过的代码（GREEN）
5. **重构**：在测试保持绿色的同时改进代码
6. **检查覆盖率**：确保 80% 以上覆盖率

## 何时使用

在以下情况使用 `/java-test`：
- 实现新的 Java 方法
- 为现有代码新增测试覆盖率
- 修复 Bug（先编写重现 bug 的测试）
- 构建关键业务逻辑
- 学习 Java 中的 TDD 工作流程

## TDD 循环

```
RED     → 先写失败的测试
GREEN   → 实现最小代码使其通过
REFACTOR → 改进代码，保持测试通过
REPEAT   → 下一功能/场景
```

## 测试模式

### 标准 JUnit 5 测试

```java
@ExtendWith(MockitoExtension.class)
class UserServiceTest {

    @Mock
    private UserMapper userMapper;

    @InjectMocks
    private UserServiceImpl userService;

    @Test
    @DisplayName("根据 ID 查询用户 - 成功")
    void getUserById_Success() {
        // Arrange
        Long userId = 1L;
        UserDO userDO = UserDO.builder()
                .id(userId)
                .username("testuser")
                .build();
        when(userMapper.selectById(userId)).thenReturn(userDO);

        // Act
        UserVO result = userService.getUserById(userId);

        // Assert
        assertThat(result).isNotNull();
        assertThat(result.getId()).isEqualTo(userId);
        assertThat(result.getUsername()).isEqualTo("testuser");
        verify(userMapper).selectById(userId);
    }

    @Test
    @DisplayName("根据 ID 查询用户 - 不存在抛出异常")
    void getUserById_NotExists_ThrowsException() {
        // Arrange
        Long userId = 999L;
        when(userMapper.selectById(userId)).thenReturn(null);

        // Act & Assert
        assertThatThrownBy(() -> userService.getUserById(userId))
                .isInstanceOf(BusinessException.class)
                .hasMessage("用户不存在");
    }
}
```

### 参数化测试

```java
@ParameterizedTest
@DisplayName("密码强度校验 - 参数化测试")
@CsvSource({
    "Password123!, true",
    "pass, false",
    "12345678, false",
    "Password, false",
    ", false"
})
void validatePassword_Valid(String password, boolean expected) {
    boolean result = passwordValidator.validate(password);
    assertThat(result).isEqualTo(expected);
}
```

### 并行测试

```java
@TestMethodOrder(MethodOrderer.OrderAnnotation.class)
class UserServiceTest {

    @Test
    @Order(1)
    @DisplayName("创建用户")
    void createUser() {
        // ...
    }

    @Test
    @Order(2)
    @DisplayName("查询用户")
    void getUser() {
        // ...
    }
}
```

## 基准测试

**Windows (PowerShell):**
```powershell
# 覆盖率检查
mvn test jacoco:report

# 覆盖率 Profile
mvn test jacoco:report -Djacoco.destFile=target/jacoco.exec

# 按方法显示覆盖率
mvn test jacoco:report; Get-Content target/site/jacoco/jacoco.csv

# 带覆盖率执行特定测试
mvn test -Dtest=UserServiceTest jacoco:report
```

**macOS/Linux:**
```bash
# 覆盖率检查
mvn test jacoco:report

# 覆盖率 Profile
mvn test jacoco:report -Djacoco.destFile=target/jacoco.exec

# 按方法显示覆盖率
mvn test jacoco:report && cat target/site/jacoco/jacoco.csv

# 带覆盖率执行特定测试
mvn test -Dtest=UserServiceTest jacoco:report
```

## 覆盖率目标

| 代码类型 | 目标 |
|-----------|------|
| 关键业务逻辑 | 100% |
| Service 层 | 90%+ |
| Controller 层 | 80%+ |
| 一般代码 | 80%+ |
| 实体类（Lombok） | 排除 |
| 配置类 | 排除 |

## 最佳实践

**应该做：**
- 在任何实现前先编写测试
- 每次变更后执行测试
- 使用 AAA 模式组织测试代码
- 测试行为，而非测试实现细节
- 包含边界情况（null、空集合、边界值）
- 使用 @DisplayName 描述测试意图
- 使用 AssertJ 断言库

**不应该做：**
- 在测试之前编写实现
- 跳过 RED 阶段
- 直接测试私有方法（通过公共接口测试）
- 在测试中使用 Thread.sleep()
- 忽略不稳定的测试
- Mock 所有东西（优先使用真实对象）

## 测试策略

```bash
# 执行带覆盖率的测试
mvn test jacoco:report

# 执行特定测试类
mvn test -Dtest=UserServiceTest

# 只测试特定方法
mvn test -Dtest=UserServiceTest#getUserById_Success

# 跳过集成测试
mvn test -DskipITs

# 执行所有测试（单元 + 集成）
mvn verify
```

## Spring Boot 测试注解

| 注解 | 用途 |
|------|------|
| @SpringBootTest | 完整应用上下文集成测试 |
| @WebMvcTest | Controller 层测试 |
| @DataJpaTest | JPA/Repository 层测试 |
| @MockBean | Mock Spring Bean |
| @TestConfiguration | 测试专用配置 |
| @Transactional | 测试后回滚数据 |

## 相关指令

- `/java-build` - 修复构建错误
- `/java-review` - 审查实现后的代码
- `/verify` - 执行完整验证循环
- `/build-fix` - 增量修复编译错误

## 相关文件

- Agent：`~/.claude/agents/java-tdd-guide.md`
- 技能：`~/.claude/skills/java-testing/`
- 技能：`~/.claude/skills/tdd-workflow/`
