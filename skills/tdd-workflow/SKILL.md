---
name: tdd-workflow
description: Use this skill when writing new features, fixing bugs, or refactoring code. Enforces test-driven development with 80%+ coverage including unit, integration, and E2E tests.
---

# 测试驱动开发工作流程

此技能确保所有代码开发遵循 TDD 原则，并具有完整的测试覆盖率。

## 技术栈

- **Java 21** - 使用现代 Java 特性（Record、Pattern Matching、Virtual Threads 等）
- **Spring Boot 3.2+** - 最新 Spring Boot 框架
- **Spring MVC** - Web 层
- **MyBatis 3 + MyBatis-Plus** - 持久层
- **Maven** - 构建工具
- **MySQL** - 数据库
- **JUnit 5** - 测试框架
- **Mockito** - Mock 框架
- **Spring Boot Test** - 集成测试支持
- **Testcontainers** - 容器化测试
- **RestAssured** - REST API 测试

## 何时启用

- 编写新功能或功能性代码
- 修复 Bug 或问题
- 重构现有代码
- 新增 REST API 端点
- 创建新 Service 或 Repository

## 核心原则

### 1. 测试先于代码
总是先写测试，然后实现代码使测试通过。

### 2. 覆盖率要求
- 最低 80% 覆盖率（单元 + 集成）
- 涵盖所有边界案例
- 测试错误场景
- 验证边界条件

### 3. 测试类型

#### 单元测试
- Service 层业务逻辑
- 纯函数和工具类
- 辅助方法和工具
- 使用 Mockito 隔离依赖

#### 集成测试
- REST API 端点（@WebMvcTest / @SpringBootTest）
- 数据库操作（MyBatis Mapper）
- Service 与 Repository 交互
- 事务处理
- 使用 Testcontainers 进行真实数据库测试

## TDD 工作流程步骤

### 步骤 1：编写用户旅程
```
身为 [角色]，我想要 [动作]，以便 [好处]

示例：
身为用户，我想要搜索产品，
以便即使没有精确关键字也能找到相关产品。
```

### 步骤 2：生成测试案例
为每个用户旅程建立完整的测试案例：

```java
@DisplayName("产品搜索服务测试")
class ProductServiceTest {

    @Test
    @DisplayName("根据关键词返回相关产品")
    void shouldReturnProductsForQuery() {
        // 测试实现
    }

    @Test
    @DisplayName("空查询时优雅处理")
    void shouldHandleEmptyQueryGracefully() {
        // 测试边界案例
    }

    @Test
    @DisplayName("数据库不可用时降级到缓存搜索")
    void shouldFallbackToCacheWhenDatabaseUnavailable() {
        // 测试降级行为
    }

    @Test
    @DisplayName("按相关性分数排序结果")
    void shouldSortResultsByRelevanceScore() {
        // 测试排序逻辑
    }
}
```

### 步骤 3：执行测试（应该失败）
```bash
mvn test
# 测试应该失败 - 我们还没实现
```

### 步骤 4：实现代码
编写最少的代码使测试通过：

```java
@Service
@RequiredArgsConstructor
public class ProductService {
    private final ProductMapper productMapper;

    public List<ProductDTO> searchProducts(String query) {
        // 实现在此
    }
}
```

### 步骤 5：再次执行测试
```bash
mvn test
# 测试现在应该通过
```

### 步骤 6：重构
在保持测试通过的同时改善代码质量：
- 移除重复
- 改善命名
- 优化性能
- 增强可读性

### 步骤 7：验证覆盖率
```bash
mvn clean test jacoco:report
# 验证达到 80%+ 覆盖率
```

## 测试模式

### 单元测试模式（JUnit 5 + Mockito）

```java
@ExtendWith(MockitoExtension.class)
@DisplayName("用户服务单元测试")
class UserServiceTest {

    @Mock
    private UserMapper userMapper;

    @Mock
    private PasswordEncoder passwordEncoder;

    @InjectMocks
    private UserService userService;

    @Test
    @DisplayName("创建用户时成功加密密码")
    void shouldEncryptPasswordWhenCreatingUser() {
        // Arrange
        String rawPassword = "password123";
        String encodedPassword = "$2a$10$encoded...";
        UserCreateRequest request = new UserCreateRequest(
            "test@example.com",
            rawPassword,
            "Test User"
        );

        when(passwordEncoder.encode(rawPassword)).thenReturn(encodedPassword);
        when(userMapper.insert(any(User.class))).thenReturn(1);

        // Act
        Long userId = userService.createUser(request);

        // Assert
        assertThat(userId).isNotNull();
        verify(passwordEncoder).encode(rawPassword);
        verify(userMapper).insert(argThat(user ->
            user.getPassword().equals(encodedPassword) &&
            user.getEmail().equals("test@example.com")
        ));
    }

    @Test
    @DisplayName("邮箱已存在时抛出异常")
    void shouldThrowExceptionWhenEmailExists() {
        // Arrange
        UserCreateRequest request = new UserCreateRequest(
            "existing@example.com",
            "password123",
            "Test User"
        );
        when(userMapper.existsByEmail("existing@example.com")).thenReturn(true);

        // Act & Assert
        assertThatThrownBy(() -> userService.createUser(request))
            .isInstanceOf(DuplicateEmailException.class)
            .hasMessageContaining("Email already exists");
    }

    @Nested
    @DisplayName("密码验证相关测试")
    class PasswordValidationTests {

        @ParameterizedTest
        @ValueSource(strings = {"", "  ", "123456"})
        @DisplayName("无效密码时抛出异常")
        void shouldThrowExceptionForInvalidPassword(String invalidPassword) {
            UserCreateRequest request = new UserCreateRequest(
                "test@example.com",
                invalidPassword,
                "Test User"
            );

            assertThatThrownBy(() -> userService.createUser(request))
                .isInstanceOf(InvalidPasswordException.class);
        }
    }
}
```

### Service 层集成测试模式

```java
@DataJpaTest
@Import({UserService.class, PasswordEncoderConfig.class})
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
@Testcontainers
@DisplayName("用户服务集成测试")
class UserServiceIntegrationTest {

    @Container
    static MySQLContainer<?> mysql = new MySQLContainer<>(
        "mysql:8.0"
    );

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", mysql::getJdbcUrl);
        registry.add("spring.datasource.username", mysql::getUsername);
        registry.add("spring.datasource.password", mysql::getPassword);
    }

    @Autowired
    private UserService userService;

    @Autowired
    private UserMapper userMapper;

    @Test
    @DisplayName("完整用户创建流程")
    void shouldCreateUserSuccessfully() {
        // Arrange
        UserCreateRequest request = new UserCreateRequest(
            "integration@example.com",
            "SecurePassword123!",
            "Integration Test"
        );

        // Act
        Long userId = userService.createUser(request);

        // Assert
        assertThat(userId).isNotNull();
        User user = userMapper.selectById(userId);
        assertThat(user).isNotNull();
        assertThat(user.getEmail()).isEqualTo("integration@example.com");
    }

    @Test
    @Transactional
    @DisplayName("事务回滚测试")
    void shouldRollbackOnError() {
        // 测试事务回滚逻辑
    }
}
```

### REST API 集成测试模式

```java
@WebMvcTest(ProductController.class)
@Import({ProductService.class, ProductMapper.class})
@DisplayName("产品 API 集成测试")
class ProductControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @MockBean
    private ProductService productService;

    @Test
    @DisplayName("GET /api/products - 成功返回产品列表")
    void shouldReturnProducts() throws Exception {
        // Arrange
        List<ProductDTO> products = List.of(
            new ProductDTO(1L, "Product A", 100.0),
            new ProductDTO(2L, "Product B", 200.0)
        );
        when(productService.getProducts(any())).thenReturn(products);

        // Act & Assert
        mockMvc.perform(get("/api/products")
                .param("page", "0")
                .param("size", "10")
                .contentType(MediaType.APPLICATION_JSON))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.success").value(true))
            .andExpect(jsonPath("$.data").isArray())
            .andExpect(jsonPath("$.data", hasSize(2)))
            .andExpect(jsonPath("$.data[0].name").value("Product A"));
    }

    @Test
    @DisplayName("POST /api/products - 创建新产品")
    void shouldCreateProduct() throws Exception {
        // Arrange
        String requestBody = """
            {
                "name": "New Product",
                "price": 299.99,
                "description": "Product description"
            }
            """;

        when(productService.createProduct(any())).thenReturn(1L);

        // Act & Assert
        mockMvc.perform(post("/api/products")
                .contentType(MediaType.APPLICATION_JSON)
                .content(requestBody))
            .andExpect(status().isCreated())
            .andExpect(jsonPath("$.success").value(true))
            .andExpect(jsonPath("$.data").value(1));
    }

    @Test
    @DisplayName("无效请求参数返回 400")
    void shouldReturn400ForInvalidRequest() throws Exception {
        String invalidBody = """
            {
                "name": "",
                "price": -100
            }
            """;

        mockMvc.perform(post("/api/products")
                .contentType(MediaType.APPLICATION_JSON)
                .content(invalidBody))
            .andExpect(status().isBadRequest());
    }
}
```

### 完整集成测试（RestAssured）

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@Testcontainers
@DisplayName("产品 API E2E 测试")
class ProductApiE2ETest {

    @Container
    static MySQLContainer<?> mysql = new MySQLContainer<>(
        "mysql:8.0"
    );

    @LocalServerPort
    private int port;

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", mysql::getJdbcUrl);
        registry.add("spring.datasource.username", mysql::getUsername);
        registry.add("spring.datasource.password", mysql::getPassword);
    }

    @BeforeEach
    void setUp() {
        RestAssured.port = port;
        RestAssured.basePath = "/api";
    }

    @Test
    @DisplayName("用户可以搜索和筛选产品")
    void userCanSearchAndFilterProducts() {
        // 创建测试数据
        given()
            .body("""
                {
                    "name": "Election Market",
                    "price": 100.0,
                    "category": "POLITICS"
                }
                """)
            .contentType(ContentType.JSON)
        .when()
            .post("/products")
        .then()
            .statusCode(201);

        // 搜索产品
        ValidatableResponse response = given()
            .queryParam("keyword", "election")
        .when()
            .get("/products/search")
        .then()
            .statusCode(200)
            .body("success", is(true))
            .body("data", hasSize(greaterThan(0)));

        // 验证搜索结果
        response.body("data[0].name", containsStringIgnoringCase("election"));

        // 按类别筛选
        given()
            .queryParam("category", "POLITICS")
        .when()
            .get("/products")
        .then()
            .statusCode(200)
            .body("data", everyItem(
                hasEntry("category", "POLITICS")
            ));
    }
}
```

## 测试文件组织

```
src/
├── main/
│   ├── java/
│   │   └── com/example/demo/
│   │       ├── controller/
│   │       │   └── ProductController.java
│   │       ├── service/
│   │       │   ├── ProductService.java
│   │       │   └── impl/
│   │       │       └── ProductServiceImpl.java
│   │       ├── mapper/
│   │       │   ├── ProductMapper.java
│   │       │   └── ProductMapper.xml
│   │       ├── entity/
│   │       │   └── Product.java
│   │       └── config/
│   │           └── SecurityConfig.java
│   └── resources/
│       ├── application.yml
│       ├── application-test.yml
│       └── mapper/
│           └── ProductMapper.xml
└── test/
    ├── java/
    │   └── com/example/demo/
    │       ├── controller/
    │       │   └── ProductControllerTest.java      # API 集成测试
    │       ├── service/
    │       │   ├── ProductServiceTest.java          # 单元测试
    │       │   └── ProductServiceIntegrationTest.java  # 集成测试
    │       ├── mapper/
    │       │   └── ProductMapperTest.java           # Mapper 测试
    │       └── util/
    │           └── PasswordUtilTest.java            # 工具类测试
    └── resources/
        ├── application-test.yml
        └── test-data.sql
```

## pom.xml 配置

```xml
<dependencies>
    <!-- Spring Boot Starter -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <!-- MyBatis-Plus -->
    <dependency>
        <groupId>com.baomidou</groupId>
        <artifactId>mybatis-plus-spring-boot3-starter</artifactId>
        <version>3.5.5</version>
    </dependency>

    <!-- MySQL Driver -->
    <dependency>
        <groupId>com.mysql</groupId>
        <artifactId>mysql-connector-j</artifactId>
        <scope>runtime</scope>
    </dependency>

    <!-- Test Dependencies -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>

    <!-- Testcontainers -->
    <dependency>
        <groupId>org.testcontainers</groupId>
        <artifactId>testcontainers</artifactId>
        <version>1.19.3</version>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.testcontainers</groupId>
        <artifactId>mysql</artifactId>
        <version>1.19.3</version>
        <scope>test</scope>
    </dependency>

    <!-- RestAssured -->
    <dependency>
        <groupId>io.rest-assured</groupId>
        <artifactId>rest-assured</artifactId>
        <version>5.4.0</version>
        <scope>test</scope>
    </dependency>
</dependencies>

<build>
    <plugins>
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
        </plugin>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-surefire-plugin</artifactId>
            <version>3.2.2</version>
        </plugin>
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
                                <element>CLASS</element>
                                <limits>
                                    <limit>
                                        <counter>LINE</counter>
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
    </plugins>
</build>
```

## Mock 数据与测试工具类

### Mapper Mock

```java
@ExtendWith(MockitoExtension.class)
class ProductMapperTest {

    @Mock
    private ProductMapper productMapper;

    @Test
    void shouldInsertProduct() {
        Product product = new Product(null, "Test", 100.0);
        when(productMapper.insert(any())).thenReturn(1);

        int result = productMapper.insert(product);

        assertThat(result).isEqualTo(1);
    }
}
```

### 测试数据构建器

```java
public class TestDataBuilder {

    public static User.CreateRequest userCreateRequest() {
        return new User.CreateRequest(
            "test@example.com",
            "SecurePassword123!",
            "Test User"
        );
    }

    public static User user() {
        return User.builder()
            .id(1L)
            .email("test@example.com")
            .password("$2a$10$encoded...")
            .username("Test User")
            .build();
    }

    public static Product product() {
        return Product.builder()
            .id(1L)
            .name("Test Product")
            .price(100.0)
            .build();
    }
}
```

### 测试配置

```java
@TestConfiguration
public class TestConfig {

    @Bean
    @Primary
    public PasswordEncoder testPasswordEncoder() {
        return new PasswordEncoder() {
            @Override
            public String encode(CharSequence rawPassword) {
                return "$2a$10$encoded" + rawPassword;
            }

            @Override
            public boolean matches(CharSequence rawPassword, String encodedPassword) {
                return encodedPassword.contains(rawPassword);
            }
        };
    }
}
```

## 测试覆盖率验证

### 执行覆盖率报告
```bash
mvn clean test jacoco:report
```

### JaCoCo 覆盖率门槛
在 pom.xml 中已配置，最低 80% 覆盖率。

## 常见测试错误避免

### 错误：测试实现细节
```java
// 不要测试私有方法或内部状态
// userService.doInternalCalculation(); // 不应该直接调用
```

### 正确：测试公开行为
```java
// 测试公开接口的行为
UserDTO result = userService.getUserById(1L);
assertThat(result.getName()).isEqualTo("Expected Name");
```

### 错误：脆弱的测试数据
```java
// 依赖数据库中特定数据
User user = userMapper.selectById(1L);
```

### 正确：测试隔离
```java
// 每个测试创建自己的数据
User user = User.builder()
    .email("test@example.com")
    .username("Test")
    .build();
userMapper.insert(user);
```

### 错误：无事务清理
```java
@Test
void createAndModifyUser() {
    userService.createUser(request1);
    // 可能影响下一个测试
}
```

### 正确：使用 @Transactional 回滚
```java
@Test
@Transactional
void createAndModifyUser() {
    // 测试结束自动回滚
}
```

## 持续测试

### 开发期间
```bash
# 自动重新运行测试
mvn test -DskipTests=false
```

### Pre-Commit Hook
```bash
# .git/hooks/pre-commit
mvn test && mvn checkstyle:check
```

### CI/CD 集成
```yaml
# GitHub Actions
- name: Set up JDK 21
  uses: actions/setup-java@v4
  with:
    java-version: '21'
    distribution: 'temurin'

- name: Run Tests
  run: mvn clean test

- name: Generate Coverage Report
  run: mvn jacoco:report

- name: Upload Coverage
  uses: codecov/codecov-action@v3
  with:
    file: target/site/jacoco/jacoco.xml
```

## 最佳实践

1. **先写测试** - 总是 TDD
2. **一个测试一个断言（或相关断言组）** - 聚焦单一行为
3. **描述性测试名称** - 使用 @DisplayName 说明测试内容
4. **AAA 模式** - Arrange-Act-Assert 清晰结构
5. **Mock 外部依赖** - 隔离单元测试
6. **测试边界案例** - Null、空值、负数、大值
7. **测试错误路径** - 不只是快乐路径
8. **保持测试快速** - 单元测试 < 100ms
9. **使用 Testcontainers** - 真实环境集成测试
10. **@Transactional 回滚** - 保证测试隔离
11. **使用 Record 作为 DTO** - Java 21 最佳实践
12. **AssertJ 断言** - 比原生断言更易读

## Java 21 特性在测试中的应用

### 使用 Record 简化测试数据

```java
record TestData(String name, Integer age, String email) {}

@Test
void testWithRecord() {
    TestData data = new TestData("Alice", 25, "alice@example.com");
    assertThat(data.name()).isEqualTo("Alice");
}
```

### 使用 Pattern Matching

```java
@Test
void testPatternMatching(Object obj) {
    switch (obj) {
        case String s && s.length() > 5 ->
            assertThat(s).hasSizeGreaterThan(5);
        case Integer i ->
            assertThat(i).isPositive();
        default ->
            fail("Unexpected type");
    }
}
```

### 使用 Virtual Threads 进行并发测试

```java
@Test
void testConcurrentOperations() throws Exception {
    try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
        List<Future<Void>> futures = new ArrayList<>();
        for (int i = 0; i < 100; i++) {
            futures.add(executor.submit(() -> {
                userService.createUser(testRequest());
                return null;
            }));
        }
        for (var future : futures) {
            future.get(); // 等待所有任务完成
        }
    }
}
```

## 成功指标

- 达到 80%+ 代码覆盖率
- 所有测试通过
- 无跳过或停用的测试
- 快速测试执行（单元测试 < 1分钟）
- 集成测试涵盖关键业务流程
- 测试在生产前捕捉 Bug

---

**记住**：测试不是可选的。它们是实现自信重构、快速开发和生产可靠性的安全网。
