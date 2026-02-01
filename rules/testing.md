# 测试规范

> **适用范围**：以下测试规范适用于 **Java / Spring Boot** 项目

## 最低测试覆盖率：80%

测试类型（全部必需）：

| 测试类型 | 覆盖范围 | 运行频率 | 工具 |
|----------|----------|----------|------|
| **单元测试** | 单个方法、工具类、Service 层 | 每次构建 | JUnit 5 + Mockito |
| **集成测试** | API 接口、数据库操作 | 每次构建 | @SpringBootTest + MockMvc |
| **E2E 测试** | 关键用户流程 | 每日/发布前 | RestAssured + TestContainers |

## 测试依赖配置

```xml
<!-- pom.xml -->
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

    <!-- Mockito JUnit 5 扩展 -->
    <dependency>
        <groupId>org.mockito</groupId>
        <artifactId>mockito-junit-jupiter</artifactId>
        <scope>test</scope>
    </dependency>

    <!-- Spring Boot Test -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>

    <!-- AssertJ (更好的断言库) -->
    <dependency>
        <groupId>org.assertj</groupId>
        <artifactId>assertj-core</artifactId>
        <scope>test</scope>
    </dependency>

    <!-- TestContainers -->
    <dependency>
        <groupId>org.testcontainers</groupId>
        <artifactId>testcontainers</artifactId>
        <scope>test</scope>
    </dependency>

    <dependency>
        <groupId>org.testcontainers</groupId>
        <artifactId>postgresql</artifactId>
        <scope>test</scope>
    </dependency>

    <!-- RestAssured (API 测试) -->
    <dependency>
        <groupId>io.rest-assured</groupId>
        <artifactId>rest-assured</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>
```

## 测试目录结构

```
src/test/java/com/company/module/
├── controller/     # Controller 测试
├── service/        # Service 测试
├── mapper/         # Mapper 测试
└── util/           # 工具类测试

src/test/resources/
├── application-test.yml    # 测试配置
├── db/
│   └── schema.sql          # 测试数据库脚本
└── data/
    └── test-data.json      # 测试数据
```

## 测试命名规范

| 类型 | 命名格式 | 示例 |
|------|----------|------|
| **测试类** | `{ClassName}Test` | `UserServiceTest` |
| **测试方法** | `{methodName}_{scenario}_{expectedResult}` | `getUserById_Success` |

```java
// 推荐的测试方法命名
@Test
void getUserById_WhenUserExists_ShouldReturnUser() {}

@Test
void getUserById_WhenUserNotFound_ShouldThrowException() {}

@Test
void createUser_WithValidData_ShouldReturnUserId() {}

@Test
void createUser_WithDuplicateEmail_ShouldThrowException() {}
```

## 单元测试（JUnit 5 + Mockito）

### Service 层测试

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
        User user = User.builder()
                .id(userId)
                .username("张三")
                .email("zhangsan@example.com")
                .build();
        when(userMapper.selectById(userId)).thenReturn(user);

        // When
        User result = userService.getById(userId);

        // Then
        assertThat(result).isNotNull();
        assertThat(result.getId()).isEqualTo(userId);
        assertThat(result.getUsername()).isEqualTo("张三");
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
        verify(userMapper, times(1)).selectById(userId);
    }

    @ParameterizedTest
    @DisplayName("创建用户 - 参数校验")
    @NullSource
    @EmptySource
    void create_WithInvalidUsername_ThrowsException(String username) {
        // Given
        UserCreateReq req = new UserCreateReq();
        req.setUsername(username);
        req.setEmail("test@example.com");

        // When & Then
        assertThatThrownBy(() -> userService.create(req))
                .isInstanceOf(BusinessException.class);
    }
}
```

### 工具类测试

```java
class StringUtilsTest {

    @Test
    @DisplayName("判断字符串是否为空")
    void isEmpty() {
        assertThat(StringUtils.isEmpty(null)).isTrue();
        assertThat(StringUtils.isEmpty("")).isTrue();
        assertThat(StringUtils.isEmpty(" ")).isTrue();
        assertThat(StringUtils.isEmpty("abc")).isFalse();
    }

    @Test
    @DisplayName("手机号脱敏")
    void maskPhone() {
        assertThat(StringUtils.maskPhone("13812345678"))
                .isEqualTo("138****5678");
        assertThat(StringUtils.maskPhone(null))
                .isNull();
    }
}
```

## 集成测试（@SpringBootTest）

### Controller 集成测试

```java
@SpringBootTest
@AutoConfigureMockMvc
@Transactional
@DisplayName("UserController 集成测试")
class UserControllerIntegrationTest {

    @Autowired
    private MockMvc mockMvc;

    @Autowired
    private ObjectMapper objectMapper;

    @Autowired
    private UserService userService;

    @Autowired
    private UserMapper userMapper;

    @BeforeEach
    void setUp() {
        // 清理测试数据
        userMapper.delete(new LambdaQueryWrapper<>());
    }

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
                .andExpect(jsonPath("$.message").value("success"))
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

    @Test
    @DisplayName("分页查询用户")
    void page_Success() throws Exception {
        // Given - 准备测试数据
        UserCreateReq req1 = new UserCreateReq("用户1", "user1@example.com");
        UserCreateReq req2 = new UserCreateReq("用户2", "user2@example.com");
        userService.create(req1);
        userService.create(req2);

        // When & Then
        mockMvc.perform(get("/api/users")
                        .param("current", "1")
                        .param("size", "10"))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.code").value(200))
                .andExpect(jsonPath("$.data.total").value(2))
                .andExpect(jsonPath("$.data.records").isArray());
    }
}
```

### MyBatis Mapper 测试

```java
@MybatisTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
@Import(UserMapper.class)
@DisplayName("UserMapper 测试")
class UserMapperTest {

    @Autowired
    private UserMapper userMapper;

    @BeforeEach
    void setUp() {
        // 清理测试数据
        userMapper.delete(new LambdaQueryWrapper<>());
    }

    @Test
    @DisplayName("插入用户")
    void insertUser() {
        // Given
        User user = User.builder()
                .username("李四")
                .email("lisi@example.com")
                .build();

        // When
        int rows = userMapper.insert(user);

        // Then
        assertThat(rows).isEqualTo(1);
        assertThat(user.getId()).isNotNull();
    }

    @Test
    @DisplayName("根据用户名查询")
    void selectByUsername() {
        // Given
        String username = "王五";
        User user = User.builder()
                .username(username)
                .email("wangwu@example.com")
                .build();
        userMapper.insert(user);

        // When
        User result = userMapper.selectOne(
                new LambdaQueryWrapper<User>().eq(User::getUsername, username)
        );

        // Then
        assertThat(result).isNotNull();
        assertThat(result.getUsername()).isEqualTo(username);
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
                .body("""
                        {
                            "username": "e2e-test-user",
                            "email": "e2e@example.com",
                            "password": "password123"
                        }
                        """)
                .when()
                .post("/api/users/register")
                .then()
                .statusCode(200)
                .extract()
                .path("data");

        // 2. 查询用户信息
        given()
                .pathParam("id", userId)
                .when()
                .get("/api/users/{id}")
                .then()
                .statusCode(200)
                .body("data.username", equalTo("e2e-test-user"));

        // 3. 更新用户信息
        given()
                .contentType(ContentType.JSON)
                .body("""
                        {
                            "id": "%s",
                            "username": "e2e-test-user-updated",
                            "email": "e2e-updated@example.com"
                        }
                        """.formatted(userId))
                .when()
                .put("/api/users")
                .then()
                .statusCode(200);

        // 4. 删除用户
        given()
                .pathParam("id", userId)
                .when()
                .delete("/api/users/{id}")
                .then()
                .statusCode(200);
    }
}
```

## 测试数据管理

### 测试数据工厂模式

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
        req.setAge(25);
        return req;
    }

    public static UserCreateReq createInvalidEmailReq() {
        UserCreateReq req = createValidCreateReq();
        req.setEmail("invalid-email");
        return req;
    }
}

// 使用示例
@Test
void createUser_Success() {
    UserCreateReq req = UserTestDataFactory.createValidCreateReq();
    // ...
}
```

### 测试数据清理策略

| 策略 | 适用场景 | 优点 | 缺点 |
|------|----------|------|------|
| **@Transactional** | 单元测试、集成测试 | 自动回滚，快速 | 不测试事务提交 |
| **@BeforeEach 清理** | 需要测试提交的场景 | 灵活控制 | 需要手动编写清理代码 |
| **TestContainers** | E2E 测试 | 完全隔离，接近生产环境 | 启动较慢 |
| **随机测试数据** | 并行测试场景 | 避免冲突 | 数据可读性降低 |

```java
// 策略 1: @Transactional 回滚（推荐用于单元测试）
@SpringBootTest
@Transactional
class UserServiceTest {
    // 每个测试方法后自动回滚
}

// 策略 2: 显式清理（推荐用于需要验证提交的场景）
@SpringBootTest
class UserServiceCommitTest {
    @Autowired
    private UserMapper userMapper;

    @AfterEach
    void cleanup() {
        userMapper.delete(new LambdaQueryWrapper<>());
    }
}

// 策略 3: TestContainers 完全隔离
@SpringBootTest
@Testcontainers
class UserE2ETest {
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15-alpine")
            .withDatabaseName("testdb")
            .withUsername("test")
            .withPassword("test");
}

// 策略 4: 使用随机数据避免并发冲突
@Component
public class TestDataGenerator {
    public String randomUsername() {
        return "user_" + UUID.randomUUID().toString().substring(0, 8);
    }
}
```

## 测试覆盖率

### JaCoCo 配置

```xml
<!-- pom.xml -->
<plugin>
    <groupId>org.jacoco</groupId>
    <artifactId>jacoco-maven-plugin</artifactId>
    <version>0.8.11</version>
    <executions>
        <execution>
            <goals>
                <goal>prepare-agent</goal>
            </goals>
        </execution>
        <execution>
            <id>report</id>
            <phase>test</phase>
            <goals>
                <goal>report</goal>
            </goals>
        </execution>
        <execution>
            <id>check</id>
            <goals>
                <goal>check</goal>
            </goals>
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

### 查看覆盖率报告

```bash
# 生成测试报告
mvn clean test jacoco:report

# 查看报告
open target/site/jacoco/index.html
```

## 测试驱动开发（TDD）

### TDD 工作流程

```
RED   → 编写失败的测试
GREEN → 编写最小实现使测试通过
REFACTOR → 重构优化代码
VERIFY → 验证覆盖率达到 80%+
```

### 使用 java-test Agent 强制 TDD

使用 **java-test** Agent 可以强制执行此流程，确保先编写测试。

### TDD 示例

```java
// 步骤 1: RED - 编写失败的测试
@Test
void calculateDiscount_WithVIPUser_ShouldReturn20Percent() {
    // Given
    User user = new User();
    user.setLevel(UserLevel.VIP);

    // When
    BigDecimal discount = discountService.calculateDiscount(user);

    // Then
    assertThat(discount).isEqualByComparingTo("0.20");
}

// 步骤 2: GREEN - 编写最小实现
public BigDecimal calculateDiscount(User user) {
    return new BigDecimal("0.20"); // 最小实现
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
// 运行: mvn test jacoco:report
```

## 测试最佳实践

### 测试隔离性

```java
// 每个测试应该独立，不依赖其他测试

// ✓ 推荐：每个测试独立准备数据
@Test
void test1() {
    // 准备数据
    // 执行测试
    // 验证结果
}

@Test
void test2() {
    // 准备数据（不依赖 test1）
    // 执行测试
    // 验证结果
}

// ✗ 不推荐：测试间有依赖
static User sharedUser;
@Test
void test1() { sharedUser = new User(); }
@Test
void test2() { /* 使用 sharedUser */ }
```

### 测试可读性

```java
// ✓ 推荐：使用 Given-When-Then 结构
@Test
void getUserById_WhenUserExists_ShouldReturnUser() {
    // Given: 准备测试数据
    Long userId = 1L;
    User user = createTestUser(userId);
    when(userMapper.selectById(userId)).thenReturn(user);

    // When: 执行被测试方法
    User result = userService.getById(userId);

    // Then: 验证结果
    assertThat(result).isNotNull();
    assertThat(result.getId()).isEqualTo(userId);
}

// ✗ 不推荐：所有代码混在一起
@Test
void test() {
    when(userMapper.selectById(1L)).thenReturn(new User());
    assertThat(userService.getById(1L)).isNotNull();
}
```

### Mock 最佳实践

```java
// ✓ 推荐：明确指定 mock 行为
when(userMapper.selectById(1L)).thenReturn(user);
when(userMapper.insert(any())).thenReturn(1);

// ✗ 不推荐：过度使用 any()
when(userMapper.selectById(any())).thenReturn(user); // 不够精确
```

## 测试失败排查

### 诊断步骤

1. **使用 java-test Agent** - 自动诊断测试失败
2. **检查测试隔离性** - 确保测试间无依赖
3. **验证 mock 行为** - 使用 `verify()` 确认调用
4. **修复实现代码** - 优先修复业务逻辑，而非测试代码

### 常见问题

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| 测试间相互影响 | 共享状态 | 使用 `@BeforeEach` 清理 |
| Mock 不生效 | Mock 设置错误 | 检查 `@ExtendWith(MockitoExtension.class)` |
| 事务未回滚 | 缺少 `@Transactional` | 添加 `@Transactional` |
| 测试数据冲突 | 并行执行冲突 | 使用随机测试数据 |

## Agent 支持

| Agent | 用途 |
|-------|------|
| **java-test** | 强制 TDD 工作流，先编写测试后实现 |
| **e2e** | 生成和运行集成测试（JUnit 5、RestAssured、TestContainers） |

## 测试检查清单

在标记测试完成前：

- [ ] 所有测试通过
- [ ] 测试覆盖率 >= 80%
- [ ] 没有忽略的测试（`@Disabled`）
- [ ] 测试命名清晰
- [ ] 测试独立（无依赖）
- [ ] Mock 行为正确验证
- [ ] 边界条件已覆盖
- [ ] 异常场景已测试
- [ ] 集成测试使用 TestContainers
