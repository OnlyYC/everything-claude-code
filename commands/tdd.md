---
name: tdd
description: 强制执行测试驱动开发工作流。先生成 JUnit 5 测试，再实现最小代码。确保 80%+ 覆盖率
command: /tdd
---

# TDD 指令

此指令调用 **java-tdd-guide** Agent 来强制执行 Java 测试驱动开发方法论。

## 此指令的功能

1. **定义接口骨架** - 先定义方法签名
2. **先生成测试** - 编写失败的 JUnit 5 测试（RED）
3. **实现最小代码** - 编写刚好足以通过的代码（GREEN）
4. **重构** - 在测试保持绿色的同时改进代码（REFACTOR）
5. **验证覆盖率** - 确保 80% 以上测试覆盖率

## 何时使用

在以下情况使用 `/tdd`：
- 实现新功能
- 新增新方法/组件
- 修复 Bug（先编写重现 bug 的测试）
- 重构现有代码
- 构建关键业务逻辑

## 运作方式

java-tdd-guide Agent 会：

1. **定义接口**用于输入/输出
2. **编写会失败的测试**（因为代码还不存在）
3. **执行测试**并验证它们因正确的原因失败
4. **编写最小实现**使测试通过
5. **执行测试**并验证它们通过
6. **重构**代码，同时保持测试通过
7. **检查覆盖率**，如果低于 80% 则新增更多测试

## TDD 循环

```
RED -> GREEN -> REFACTOR -> REPEAT

RED:      编写失败的测试
GREEN:    编写最小代码使其通过
REFACTOR: 改进代码，保持测试通过
REPEAT:   下一个功能/场景
```

## 测试模式

### JUnit 5 标准测试

```java
@ExtendWith(MockitoExtension.class)
class UserServiceTest {

    @Mock
    private UserMapper userMapper;

    @InjectMocks
    private UserServiceImpl userService;

    @Test
    @DisplayName("根据 ID 查询用户 - 存在")
    void getUserById_Exists() {
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
    @DisplayName("根据 ID 查询用户 - 不存在")
    void getUserById_NotExists() {
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
@DisplayName("密码校验 - 参数化测试")
@CsvSource({
    "Password123!, true",
    "pass, false",
    "12345678, false",
    "Password, false"
})
void validatePassword_Valid(String password, boolean expected) {
    boolean result = passwordValidator.validate(password);
    assertThat(result).isEqualTo(expected);
}
```

### Mock 测试

```java
@Test
@DisplayName("创建用户 - 用户名已存在")
void createUser_UsernameExists() {
    // Arrange
    CreateUserRequest request = CreateUserRequest.builder()
            .username("existinguser")
            .build();
    when(userMapper.selectByUsername("existinguser"))
            .thenReturn(Optional.of(new UserDO()));

    // Act & Assert
    assertThatThrownBy(() -> userService.createUser(request))
            .isInstanceOf(BusinessException.class)
            .hasMessage("用户名已存在");
}
```

## 覆盖率要求

- **所有代码至少 80%**
- **以下类型需要 100%：**
  - 财务计算
  - 权限校验逻辑
  - 安全关键代码
  - 核心业务逻辑

## TDD 最佳实践

**应该做：**
- 在任何实现前先编写测试
- 在实现前执行测试并验证它们失败
- 编写最小代码使测试通过
- 只在测试通过后才重构
- 新增边界情况和错误场景
- 使用 @DisplayName 描述测试意图
- 使用 AssertJ 或 Hamcrest 断言库

**不应该做：**
- 在测试之前编写实现
- 跳过每次变更后执行测试
- 一次编写太多代码
- 忽略失败的测试
- 测试实现细节（测试行为）
- Mock 所有东西（优先使用真实对象）

## 测试命令

**Windows (PowerShell):**
```powershell
# 执行所有测试
mvn test

# 执行特定测试类
mvn test -Dtest=UserServiceTest

# 执行特定测试方法
mvn test -Dtest=UserServiceTest#getUserById_Exists

# 生成覆盖率报告
mvn test jacoco:report

# 查看覆盖率
mvn test jacoco:report; start target/site/jacoco/index.html
```

**macOS/Linux:**
```bash
# 执行所有测试
mvn test

# 执行特定测试类
mvn test -Dtest=UserServiceTest

# 执行特定测试方法
mvn test -Dtest=UserServiceTest#getUserById_Exists

# 生成覆盖率报告
mvn test jacoco:report

# 查看覆盖率
mvn test jacoco:report && open target/site/jacoco/index.html
```

## 依赖配置

```xml
<!-- JUnit 5 -->
<dependency>
    <groupId>org.junit.jupiter</groupId>
    <artifactId>junit-jupiter</artifactId>
    <scope>test</scope>
</dependency>

<!-- Mockito -->
<dependency>
    <groupId>org.mockito</groupId>
    <artifactId>mockito-core</artifactId>
    <scope>test</scope>
</dependency>

<!-- AssertJ -->
<dependency>
    <groupId>org.assertj</groupId>
    <artifactId>assertj-core</artifactId>
    <scope>test</scope>
</dependency>

<!-- Spring Boot Test -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
</dependency>
```

## 重要提醒

**强制要求**：测试必须在实现之前编写。TDD 循环是：

1. **RED** - 编写失败的测试
2. **GREEN** - 实现使其通过
3. **REFACTOR** - 改进代码

绝不跳过 RED 阶段。绝不不在测试之前编写代码。

## 与其他指令的集成

- 先使用 `/plan` 理解要构建什么
- 使用 `/tdd` 带着测试实现
- 如果发生构建错误，使用 `/build-fix` 或 `/java-build`
- 使用 `/code-review` 审查实现
- 使用 `/test-coverage` 验证覆盖率

## 相关 Agent

此指令调用位于以下位置的 **java-tdd-guide** Agent：
`~/.claude/agents/java-tdd-guide.md`

并可参考位于以下位置的 `java-tdd-workflow` 技能：
`~/.claude/skills/java-tdd-workflow/`
