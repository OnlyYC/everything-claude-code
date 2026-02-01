---
name: testing
description: Java/Spring Boot 测试规范与 TDD 工作流
priority: must
tags: [java, spring-boot, testing, junit, tdd]
tech-stack: [JUnit 5, Mockito, TestContainers, RestAssured]
---

# 测试规范

> **适用范围**：以下测试规范适用于 **Java / Spring Boot** 项目
> **最低覆盖率**：80%

## 测试类型

| 测试类型 | 覆盖范围 | 运行频率 | 工具 |
|----------|----------|----------|------|
| **单元测试** | 单个方法、Service 层 | 每次构建 | JUnit 5 + Mockito |
| **集成测试** | API 接口、数据库 | 每次构建 | @SpringBootTest + MockMvc |
| **E2E 测试** | 关键用户流程 | 每日/发布前 | RestAssured + TestContainers |

## 测试依赖

```xml
<dependencies>
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

    <!-- Spring Boot Test -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>

    <!-- TestContainers -->
    <dependency>
        <groupId>org.testcontainers</groupId>
        <artifactId>testcontainers</artifactId>
        <scope>test</scope>
    </dependency>

    <!-- RestAssured -->
    <dependency>
        <groupId>io.rest-assured</groupId>
        <artifactId>rest-assured</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>
```

## 测试命名

| 类型 | 命名格式 | 示例 |
|------|----------|------|
| **测试类** | `{ClassName}Test` | `UserServiceTest` |
| **测试方法** | `{methodName}_{scenario}_{expectedResult}` | `getUserById_WhenUserExists_ShouldReturnUser` |

```java
@Test
void getUserById_WhenUserExists_ShouldReturnUser() {}

@Test
void getUserById_WhenUserNotFound_ShouldThrowException() {}

@Test
void createUser_WithValidData_ShouldReturnUserId() {}
```

## 单元测试（JUnit 5 + Mockito）

```java
@ExtendWith(MockitoExtension.class)
@DisplayName("UserService 单元测试")
class UserServiceTest {

    @Mock
    private UserMapper userMapper;

    @InjectMocks
    private UserServiceImpl userService;

    @Test
    @DisplayName("根据ID查询用户 - 成功")
    void getUserById_Success() {
        // Given
        Long userId = 1L;
        User user = User.builder().id(userId).username("张三").build();
        when(userMapper.selectById(userId)).thenReturn(user);

        // When
        User result = userService.getById(userId);

        // Then
        assertThat(result).isNotNull();
        assertThat(result.getId()).isEqualTo(userId);
        verify(userMapper, times(1)).selectById(userId);
    }

    @Test
    @DisplayName("根据ID查询用户 - 不存在抛出异常")
    void getUserById_NotFound_ThrowsException() {
        // Given
        Long userId = 999L;
        when(userMapper.selectById(userId)).thenReturn(null);

        // When & Then
        assertThatThrownBy(() -> userService.getById(userId))
                .isInstanceOf(BusinessException.class)
                .hasMessageContaining("用户不存在");
    }
}
```

## 集成测试（@SpringBootTest）

```java
@SpringBootTest
@AutoConfigureMockMvc
@Transactional
@DisplayName("UserController 集成测试")
class UserControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @Autowired
    private ObjectMapper objectMapper;

    @Test
    @DisplayName("创建用户 - 成功")
    void createUser_Success() throws Exception {
        // Given
        UserCreateReq req = new UserCreateReq();
        req.setUsername("测试用户");
        req.setEmail("test@example.com");

        // When & Then
        mockMvc.perform(post("/api/users")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(objectMapper.writeValueAsString(req)))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.code").value(200))
                .andExpect(jsonPath("$.data").isNumber());
    }

    @Test
    @DisplayName("创建用户 - 参数校验失败")
    void createUser_ValidationFail() throws Exception {
        // Given
        UserCreateReq req = new UserCreateReq();
        req.setUsername("");
        req.setEmail("invalid-email");

        // When & Then
        mockMvc.perform(post("/api/users")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(objectMapper.writeValueAsString(req)))
                .andExpect(status().isBadRequest());
    }
}
```

## E2E 测试（RestAssured + TestContainers）

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@Testcontainers
@DisplayName("用户端到端测试")
class UserE2ETest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15-alpine")
            .withDatabaseName("testdb")
            .withUsername("test")
            .withPassword("test");

    @DynamicPropertySource
    static void postgresProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }

    @LocalServerPort
    private int port;

    @BeforeEach
    void setUp() {
        RestAssured.baseURI = "http://localhost";
        RestAssured.port = port;
    }

    @Test
    @DisplayName("完整的用户注册流程")
    void completeUserRegistrationFlow() {
        // 1. 注册用户
        String userId = given()
                .contentType(ContentType.JSON)
                .body("{\"username\":\"e2e-test\",\"email\":\"e2e@example.com\"}")
                .when().post("/api/users/register")
                .then().statusCode(200)
                .extract().path("data");

        // 2. 查询用户
        given().pathParam("id", userId)
                .when().get("/api/users/{id}")
                .then().statusCode(200)
                .body("data.username", equalTo("e2e-test"));

        // 3. 删除用户
        given().pathParam("id", userId)
                .when().delete("/api/users/{id}")
                .then().statusCode(200);
    }
}
```

## 测试数据管理

### 数据工厂模式

```java
@Component
public class UserTestDataFactory {
    public static User createDefaultUser() {
        return User.builder()
                .username("测试用户")
                .email("test@example.com")
                .age(25)
                .build();
    }

    public static UserCreateReq createValidCreateReq() {
        UserCreateReq req = new UserCreateReq();
        req.setUsername("测试用户");
        req.setEmail("test@example.com");
        return req;
    }
}
```

### 数据清理策略

| 策略 | 适用场景 | 优点 | 缺点 |
|------|----------|------|------|
| **@Transactional** | 单元测试、集成测试 | 自动回滚，快速 | 不测试事务提交 |
| **@BeforeEach 清理** | 需要测试提交的场景 | 灵活控制 | 需要手动编写清理代码 |
| **TestContainers** | E2E 测试 | 完全隔离 | 启动较慢 |

```java
// 策略 1: @Transactional 回滚
@SpringBootTest
@Transactional
class UserServiceTest { }

// 策略 2: 显式清理
@SpringBootTest
class UserServiceCommitTest {
    @Autowired
    private UserMapper userMapper;

    @AfterEach
    void cleanup() {
        userMapper.delete(new LambdaQueryWrapper<>());
    }
}
```

## 测试覆盖率

### JaCoCo 配置

```xml
<plugin>
    <groupId>org.jacoco</groupId>
    <artifactId>jacoco-maven-plugin</artifactId>
    <version>0.8.11</version>
    <executions>
        <execution>
            <goals><goal>prepare-agent</goal></goals>
        </execution>
        <execution>
            <id>check</id>
            <goals><goal>check</goal></goals>
            <configuration>
                <rules>
                    <rule>
                        <element>BUNDLE</element>
                        <limits>
                            <limit>
                                <counter>INSTRUCTION</counter>
                                <value>COVEREDRATIO</value>
                                <minimum>0.80</minimum>
                            </limit>
                        </limits>
                    </rule>
                </rules>
            </configuration>
        </execution>
    </executions>
</plugin>
```

### 查看覆盖率

```bash
mvn clean test jacoco:report
open target/site/jacoco/index.html
```

## TDD 工作流

```
RED   → 编写失败的测试
GREEN → 编写最小实现使测试通过
REFACTOR → 重构优化代码
VERIFY → 验证 80%+ 覆盖率
```

### TDD 示例

```java
// 步骤 1: RED - 编写失败的测试
@Test
void calculateDiscount_WithVIPUser_ShouldReturn20Percent() {
    User user = new User();
    user.setLevel(UserLevel.VIP);

    BigDecimal discount = discountService.calculateDiscount(user);

    assertThat(discount).isEqualByComparingTo("0.20");
}

// 步骤 2: GREEN - 编写最小实现
public BigDecimal calculateDiscount(User user) {
    return new BigDecimal("0.20");
}

// 步骤 3: REFACTOR - 重构优化
public BigDecimal calculateDiscount(User user) {
    return switch (user.getLevel()) {
        case VIP -> new BigDecimal("0.20");
        case REGULAR -> new BigDecimal("0.10");
        default -> BigDecimal.ZERO;
    };
}

// 步骤 4: VERIFY - 验证覆盖率
// mvn test jacoco:report
```

## 测试最佳实践

| 原则 | 说明 |
|------|------|
| **测试隔离性** | 每个测试独立，不依赖其他测试 |
| **Given-When-Then** | 清晰的测试结构：准备-执行-验证 |
| **Mock 明确行为** | 避免过度使用 any() |
| **边界条件** | 测试空值、边界值、异常场景 |
| **测试命名** | 使用描述性命名，清楚表达测试意图 |

### 测试隔离性示例

```java
// 推荐：每个测试独立准备数据
@Test
void test1() {
    User user = createTestUser();
    // 执行测试
}

@Test
void test2() {
    User user = createTestUser(); // 不依赖 test1
    // 执行测试
}

// 不推荐：测试间有依赖
static User sharedUser;
@Test
void test1() { sharedUser = new User(); }
@Test
void test2() { /* 使用 sharedUser */ }
```

## Agent 支持

| Agent | 用途 |
|-------|------|
| **java-test** | 强制 TDD 工作流，先编写测试后实现 |
| **e2e** | 生成和运行集成测试 |

## 测试检查清单

- [ ] 所有测试通过
- [ ] 测试覆盖率 >= 80%
- [ ] 没有忽略的测试（@Disabled）
- [ ] 测试命名清晰（Given-When-Then）
- [ ] 测试独立（无依赖）
- [ ] Mock 行为正确验证
- [ ] 边界条件已覆盖
- [ ] 异常场景已测试
- [ ] 集成测试使用 TestContainers
