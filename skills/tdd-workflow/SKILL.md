---
name: tdd-workflow
description: 通用 TDD 方法论：红-绿-重构、测试设计原则、测试组织。关注测试驱动开发的核心流程，而非特定框架实现。
version: 1.1.0
tech_stack: [Java 21, JUnit 5, Mockito, Testcontainers]
tools: Read, Write, Edit, Bash, Grep, Glob
related_skills: [springboot-tdd, springboot-verification, eval-harness]
---

# 测试驱动开发工作流程

此技能确保所有代码开发遵循 TDD 原则，并具有完整的测试覆盖率。

## 技能职责划分

| 技能 | 职责 |
|------|------|
| `tdd-workflow` | 通用 TDD 方法论：红-绿-重构、测试设计原则、测试组织 |
| `springboot-tdd` | Spring Boot 特定测试：@WebMvcTest、@DataJpaTest、@MockBean |
| `verification-loop` | 完整项目验证：构建、静态分析、安全扫描、Docker 验证、性能验证 |

**核心区别**：
- 本技能聚焦 "为什么要测试和如何设计测试"
- Spring Boot 特定注解请参考 `springboot-tdd`
- 完整项目验证流程（包括构建、安全、性能）请参考 `verification-loop`
- `tdd-workflow` = 开发阶段的测试实践
- `verification-loop` = 部署前的全面质量验证

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
        // Arrange
        String invalidBody = """
            {
                "name": "",
                "price": -100
            }
            """;

        // Act & Assert
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

---

## 高级测试数据构建器

> **注意**：以下为高级测试模式，适用于复杂测试场景。简单测试可使用基本模式。

### 流式构建器模式（简化版）

```java
public class UserBuilder {
    private Long id = 1L;
    private String email = "test@example.com";
    private String username = "testuser";
    private String password = "Password123!";
    private UserStatus status = UserStatus.ACTIVE;
    private LocalDateTime createdAt = LocalDateTime.now();

    private UserBuilder() {}

    public static UserBuilder builder() {
        return new UserBuilder();
    }

    public UserBuilder id(Long id) {
        this.id = id;
        return this;
    }

    public UserBuilder email(String email) {
        this.email = email;
        return this;
    }

    public UserBuilder username(String username) {
        this.username = username;
        return this;
    }

    public UserBuilder password(String password) {
        this.password = password;
        return this;
    }

    public UserBuilder status(UserStatus status) {
        this.status = status;
        return this;
    }

    public UserBuilder createdAt(LocalDateTime createdAt) {
        this.createdAt = createdAt;
        return this;
    }

    public User build() {
        return User.builder()
            .id(id)
            .email(email)
            .username(username)
            .password(password)
            .status(status)
            .createdAt(createdAt)
            .build();
    }

    // 预设配置
    public UserBuilder asActiveUser() {
        return status(UserStatus.ACTIVE);
    }

    public UserBuilder asInactiveUser() {
        return status(UserStatus.INACTIVE);
    }

    public UserBuilder asAdmin() {
        return username("admin")
            .email("admin@example.com")
            .status(UserStatus.ACTIVE);
    }
}

// 使用示例
User user = UserBuilder.builder()
    .asActiveUser()
    .email("custom@example.com")
    .build();
```

### 组合构建器

```java
public class OrderTestData {

    public static class OrderBuilder {
        private Long id = 1L;
        private Long userId = 1L;
        private OrderStatus status = OrderStatus.PENDING;
        private List<OrderItem> items = new ArrayList<>();
        private BigDecimal totalAmount = BigDecimal.ZERO;

        private OrderBuilder() {}

        public OrderBuilder id(Long id) {
            this.id = id;
            return this;
        }

        public OrderBuilder userId(Long userId) {
            this.userId = userId;
            return this;
        }

        public OrderBuilder status(OrderStatus status) {
            this.status = status;
            return this;
        }

        public OrderBuilder addItem(OrderItem item) {
            this.items.add(item);
            return this;
        }

        public OrderBuilder addItem(Long productId, int quantity, BigDecimal price) {
            OrderItem item = OrderItem.builder()
                .productId(productId)
                .quantity(quantity)
                .price(price)
                .build();
            return addItem(item);
        }

        public OrderBuilder withDefaultItems() {
            return addItem(1L, 2, new BigDecimal("100.00"))
                .addItem(2L, 1, new BigDecimal("50.00"));
        }

        public Order build() {
            // 自动计算总金额
            totalAmount = items.stream()
                .map(item -> item.getPrice().multiply(
                    BigDecimal.valueOf(item.getQuantity())))
                .reduce(BigDecimal.ZERO, BigDecimal::add);

            return Order.builder()
                .id(id)
                .userId(userId)
                .status(status)
                .items(items)
                .totalAmount(totalAmount)
                .createdAt(LocalDateTime.now())
                .build();
        }
    }

    public static OrderBuilder order() {
        return new OrderBuilder();
    }

    // 预设场景
    public static Order pendingOrder() {
        return order()
            .status(OrderStatus.PENDING)
            .withDefaultItems()
            .build();
    }

    public static Order completedOrder() {
        return order()
            .status(OrderStatus.COMPLETED)
            .withDefaultItems()
            .build();
    }

    public static Order cancelledOrder() {
        return order()
            .status(OrderStatus.CANCELLED)
            .build();
    }
}

// 使用示例
Order order = OrderTestData.pendingOrder();
Order customOrder = OrderTestData.order()
    .status(OrderStatus.PROCESSING)
    .addItem(3L, 5, new BigDecimal("29.99"))
    .build();
```

### 测试数据工厂

```java
public class TestDataFactory {

    private static final Random random = new Random();
    private static long idCounter = 1;

    public static User randomUser() {
        return User.builder()
            .id(nextId())
            .email(randomEmail())
            .username(randomUsername())
            .password(randomPassword())
            .status(randomStatus())
            .createdAt(LocalDateTime.now())
            .build();
    }

    public static Product randomProduct() {
        return Product.builder()
            .id(nextId())
            .name("Product " + randomString(5))
            .price(new BigDecimal(random.nextInt(10000) / 100.0))
            .category(randomCategory())
            .build();
    }

    public static List<User> randomUsers(int count) {
        return IntStream.range(0, count)
            .mapToObj(i -> randomUser())
            .toList();
    }

    // 辅助方法
    private static Long nextId() {
        return idCounter++;
    }

    private static String randomEmail() {
        return "user" + random.nextInt(10000) + "@example.com";
    }

    private static String randomUsername() {
        return "user_" + randomString(8);
    }

    private static String randomPassword() {
        return "Pass" + random.nextInt(10000) + "!";
    }

    private static String randomString(int length) {
        return random.ints('a', 'z' + 1)
            .limit(length)
            .collect(StringBuilder::new, StringBuilder::appendCodePoint, StringBuilder::append)
            .toString();
    }

    private static UserStatus randomStatus() {
        UserStatus[] values = UserStatus.values();
        return values[random.nextInt(values.length)];
    }

    private static String randomCategory() {
        String[] categories = {"ELECTRONICS", "CLOTHING", "FOOD", "BOOKS"};
        return categories[random.nextInt(categories.length)];
    }
}
```

### Fixture 模式

```java
public class Fixtures {

    private final Map<String, Object> fixtures = new HashMap<>();

    public <T> void add(String name, T fixture) {
        fixtures.put(name, fixture);
    }

    @SuppressWarnings("unchecked")
    public <T> T get(String name) {
        return (T) fixtures.get(name);
    }

    public <T> T getOrCreate(String name, Supplier<T> supplier) {
        return (T) fixtures.computeIfAbsent(name, k -> supplier.get());
    }

    public void clear() {
        fixtures.clear();
    }

    // 在测试基类中使用
    public static class FixturesExtension implements BeforeEachCallback, AfterEachCallback {

        private final Fixtures fixtures = new Fixtures();

        public Fixtures getFixtures() {
            return fixtures;
        }

        @Override
        public void beforeEach(ExtensionContext context) {
            fixtures.clear();
        }

        @Override
        public void afterEach(ExtensionContext context) {
            fixtures.clear();
        }
    }
}

// 使用
@ExtendWith(Fixtures.FixturesExtension.class)
class MyTest {

    @Autowired
    private Fixtures.FixturesExtension fixturesExtension;

    @Test
    void testWithFixtures() {
        Fixtures fixtures = fixturesExtension.getFixtures();

        // 创建或获取 fixture
        User user = fixtures.getOrCreate("defaultUser", () ->
            UserBuilder.builder().asActiveUser().build()
        );

        // 使用 fixture
        service.processUser(user);
    }
}
```

---

## 高级 Testcontainers 配置

### 复合容器配置

```java
@Testcontainers
@SpringBootTest
class CompositeContainerTest {

    // MySQL 容器
    @Container
    static MySQLContainer<?> mysql = new MySQLContainer<>("mysql:8.0")
        .withDatabaseName("testdb")
        .withUsername("test")
        .withPassword("test")
        .withReuse(true); // 开发时可重用容器

    // Redis 容器
    @Container
    static GenericContainer<?> redis = new GenericContainer<>("redis:7-alpine")
        .withExposedPorts(6379)
        .withReuse(true);

    // MinIO（S3 兼容）
    @Container
    static GenericContainer<?> minio = new GenericContainer<>("minio/minio")
        .withExposedPorts(9000, 9001)
        .withEnv("MINIO_ROOT_USER", "minioadmin")
        .withEnv("MINIO_ROOT_PASSWORD", "minioadmin")
        .withCommand("server /data --console-address ':9001'")
        .withReuse(true);

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        // MySQL 配置
        registry.add("spring.datasource.url", mysql::getJdbcUrl);
        registry.add("spring.datasource.username", mysql::getUsername);
        registry.add("spring.datasource.password", mysql::getPassword);

        // Redis 配置
        registry.add("spring.data.redis.host", redis::getHost);
        registry.add("spring.data.redis.port", () -> redis.getMappedPort(6379));

        // MinIO 配置
        registry.add("s3.endpoint", () ->
            "http://" + minio.getHost() + ":" + minio.getMappedPort(9000)
        );
        registry.add("s3.access-key", () -> "minioadmin");
        registry.add("s3.secret-key", () -> "minioadmin");
    }
}
```

### 自定义容器配置

```java
public class CustomMySQLContainer extends MySQLContainer<CustomMySQLContainer> {

    private String initScriptPath;

    public CustomMySQLContainer(String dockerImageName) {
        super(dockerImageName);
    }

    public CustomMySQLContainer withInitScript(String path) {
        this.initScriptPath = path;
        return this;
    }

    @Override
    public void start() {
        super.start();

        if (initScriptPath != null) {
            executeInitScript();
        }
    }

    private void executeInitScript() {
        try {
            String script = new String(
                Files.readAllBytes(Paths.get(initScriptPath)),
                StandardCharsets.UTF_8
            );

            String[] statements = script.split(";");
            for (String statement : statements) {
                if (!statement.trim().isEmpty()) {
                    execInContainer("mysql",
                        "-h" + getHost(),
                        "-P" + getFirstMappedPort(),
                        "-u" + getUsername(),
                        "-p" + getPassword(),
                        getDatabaseName(),
                        "-e", statement
                    );
                }
            }
        } catch (Exception e) {
            throw new RuntimeException("Failed to execute init script", e);
        }
    }
}

// 使用
@Container
static CustomMySQLContainer mysql = new CustomMySQLContainer("mysql:8.0")
    .withInitScript("src/test/resources/init.sql");
```

### 容器网络

```java
@Testcontainers
class NetworkTest {

    @Container
    static Network network = Network.newNetwork();

    @Container
    static GenericContainer<?> app = new GenericContainer<>("myapp:latest")
        .withNetwork(network)
        .withNetworkAliases("app")
        .dependsOn(db);

    @Container
    static MySQLContainer<?> db = new MySQLContainer<>("mysql:8.0")
        .withNetwork(network)
        .withNetworkAliases("db");
}
```

### Docker Compose 模式

```java
@Testcontainers
class DockerComposeTest {

    @Container
    static DockerComposeContainer<?> environment =
        new DockerComposeContainer<>(new File("src/test/resources/docker-compose.yml"))
            .withExposedService("mysql_1", 3306)
            .withExposedService("redis_1", 6379);

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url",
            () -> "jdbc:mysql://localhost:" +
                  environment.getServicePort("mysql_1", 3306) + "/testdb"
        );
        registry.add("spring.data.redis.port",
            () -> environment.getServicePort("redis_1", 6379)
        );
    }
}
```

---

## 性能测试模式

### JUnit 5 性能测试

```java
@Test
@Timeout(value = 2, unit = TimeUnit.SECONDS)
void shouldCompleteWithinTwoSeconds() {
    // 测试必须在 2 秒内完成
    service.performHeavyOperation();
}

@Test
void shouldHandleThousandRequestsQuickly() {
    long start = System.nanoTime();

    for (int i = 0; i < 1000; i++) {
        service.processRequest(i);
    }

    long duration = System.nanoTime() - start;
    long durationMs = duration / 1_000_000;

    assertThat(durationMs).isLessThan(1000); // 1000 次请求在 1 秒内完成
}
```

### JMH（Java Microbenchmark Harness）

```java
@State(Scope.Thread)
@BenchmarkMode(Mode.AverageTime)
@OutputTimeUnit(TimeUnit.MILLISECONDS)
public class ServiceBenchmark {

    private UserService userService;

    @Setup
    public void setup() {
        userService = new UserService();
    }

    @Benchmark
    public User benchmarkCreateUser() {
        return userService.createUser(
            new UserCreateRequest("test@example.com", "pass", "Test")
        );
    }

    @Benchmark
    public void benchmarkSearch() {
        userService.searchUsers("test");
    }

    @Benchmark
    @Threads(10)
    public void benchmarkConcurrent() {
        userService.createUser(new UserCreateRequest(
            "concurrent@example.com", "pass", "Concurrent"
        ));
    }
}
```

### 并发压力测试

```java
@SpringBootTest
class ConcurrencyTest {

    @Autowired
    private UserService userService;

    @Test
    void shouldHandleConcurrentRequests() throws Exception {
        int threadCount = 100;
        int requestsPerThread = 10;

        ExecutorService executor = Executors.newFixedThreadPool(threadCount);
        CountDownLatch latch = new CountDownLatch(threadCount * requestsPerThread);
        AtomicInteger successCount = new AtomicInteger();
        AtomicInteger errorCount = new AtomicInteger();

        for (int i = 0; i < threadCount; i++) {
            executor.submit(() -> {
                for (int j = 0; j < requestsPerThread; j++) {
                    try {
                        userService.createUser(new UserCreateRequest(
                            "test" + Thread.currentThread().getId() +
                            "-" + j + "@example.com",
                            "Password123!",
                            "Test User"
                        ));
                        successCount.incrementAndGet();
                    } catch (Exception e) {
                        errorCount.incrementAndGet();
                    } finally {
                        latch.countDown();
                    }
                }
            });
        }

        latch.await(30, TimeUnit.SECONDS);
        executor.shutdown();

        // 验证所有请求都成功
        assertThat(errorCount.get()).isEqualTo(0);
        assertThat(successCount.get()).isEqualTo(threadCount * requestsPerThread);
    }

    @Test
    void shouldHandleRaceCondition() throws Exception {
        int threadCount = 50;
        Long userId = 1L;
        CountDownLatch startLatch = new CountDownLatch(1);
        CountDownLatch endLatch = new CountDownLatch(threadCount);

        AtomicInteger successCount = new AtomicInteger();

        for (int i = 0; i < threadCount; i++) {
            new Thread(() -> {
                try {
                    startLatch.await();
                    userService.incrementUserCredit(userId, 10);
                    successCount.incrementAndGet();
                } catch (Exception e) {
                    // 预期可能有部分失败
                } finally {
                    endLatch.countDown();
                }
            }).start();
        }

        startLatch.countDown(); // 同时启动所有线程
        endLatch.await(10, TimeUnit.SECONDS);

        // 验证最终结果正确（无竞态条件）
        User user = userService.getById(userId);
        assertThat(user.getCredit()).isEqualTo(100 + threadCount * 10);
    }
}
```

### 内存泄漏测试

```java
@Test
void shouldNotLeakMemory() {
    Runtime runtime = Runtime.getRuntime();
    long initialMemory = runtime.totalMemory() - runtime.freeMemory();

    for (int i = 0; i < 10000; i++) {
        userService.processRequest(new Request("test" + i));
    }

    System.gc(); // 建议垃圾回收
    long finalMemory = runtime.totalMemory() - runtime.freeMemory();
    long memoryIncrease = finalMemory - initialMemory;

    // 内存增长不应超过 10MB
    assertThat(memoryIncrease).isLessThan(10 * 1024 * 1024);
}
```

### 负载测试模拟

```java
@SpringBootTest
@AutoConfigureMockMvc
class LoadSimulationTest {

    @Autowired
    private MockMvc mockMvc;

    @Test
    void simulatePeakLoad() throws Exception {
        int concurrentUsers = 100;
        int requestsPerUser = 20;
        ExecutorService executor = Executors.newFixedThreadPool(concurrentUsers);

        List<Future<Result>> futures = new ArrayList<>();

        long startTime = System.currentTimeMillis();

        for (int i = 0; i < concurrentUsers; i++) {
            final int userId = i;
            Future<Result> future = executor.submit(() -> {
                int success = 0;
                int failure = 0;

                for (int j = 0; j < requestsPerUser; j++) {
                    try {
                        mockMvc.perform(get("/api/users/" + userId))
                            .andExpect(status().isOk());
                        success++;
                    } catch (Exception e) {
                        failure++;
                    }
                }

                return new Result(success, failure);
            });

            futures.add(future);
        }

        int totalSuccess = 0;
        int totalFailure = 0;

        for (Future<Result> future : futures) {
            Result result = future.get();
            totalSuccess += result.success;
            totalFailure += result.failure;
        }

        long duration = System.currentTimeMillis() - startTime;

        // 验证性能指标
        double successRate = (double) totalSuccess /
            (totalSuccess + totalFailure);
        assertThat(successRate).isGreaterThan(0.99); // 99% 成功率

        double requestsPerSecond = (concurrentUsers * requestsPerUser) /
            (duration / 1000.0);
        assertThat(requestsPerSecond).isGreaterThan(100); // 至少 100 RPS

        executor.shutdown();
    }

    private record Result(int success, int failure) {}
}
```

### 数据库性能测试

```java
@DataJpaTest
@Testcontainers
class DatabasePerformanceTest {

    @Container
    static MySQLContainer<?> mysql = new MySQLContainer<>("mysql:8.0");

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", mysql::getJdbcUrl);
    }

    @Autowired
    private UserRepository userRepository;

    @Test
    void bulkInsertPerformance() {
        long start = System.nanoTime();

        for (int i = 0; i < 10000; i++) {
            User user = User.builder()
                .email("user" + i + "@example.com")
                .username("user" + i)
                .build();
            userRepository.save(user);
        }

        long duration = System.nanoTime() - start;
        long durationMs = duration / 1_000_000;

        // 10000 条插入应在 5 秒内完成
        assertThat(durationMs).isLessThan(5000);
    }

    @Test
    void queryPerformance() {
        // 先插入数据
        for (int i = 0; i < 1000; i++) {
            userRepository.save(User.builder()
                .email("user" + i + "@example.com")
                .username("user" + i)
                .build());
        }

        long start = System.nanoTime();

        List<User> users = userRepository.findAll();

        long duration = System.nanoTime() - start;
        long durationMs = duration / 1_000_000;

        assertThat(users).hasSize(1000);
        assertThat(durationMs).isLessThan(500); // 查询应在 500ms 内完成
    }
}
```
