---
name: java-testing
description: Java 通用测试框架：JUnit 5 + Mockito 5 + AssertJ + TestContainers。适配 Java 21 技术栈。涵盖单元测试、参数化测试、Mock 使用、断言、测试覆盖率等。
version: 1.1.0
tech_stack: [Java 21, JUnit 5, Mockito 5, AssertJ, TestContainers]
tools: Read, Write, Edit, Bash, Grep, Glob
related_skills:
  springboot-tdd: "本 skill 负责通用测试框架；springboot-tdd 负责 Spring Boot 特定测试注解"
  tdd-workflow: "TDD 方法论"
---

# Java 通用测试框架

基于 JUnit 5 + Mockito 5 + AssertJ + TestContainers 的通用 Java 测试框架指南。测试覆盖率目标 80%+。

## 技能职责划分

| 技能 | 职责 | 内容范围 |
|------|------|---------|
| `java-testing` | **通用测试** | JUnit、Mockito、AssertJ、TestContainers 等通用测试框架 |
| `springboot-tdd` | **Spring 测试** | @WebMvcTest、@DataJpaTest、@MockBean 等 Spring 特定注解 |

## 核心原则

- **测试先行** - 遵循 TDD 原则，先写测试再写代码
- **AAA 模式** - Arrange（准备）、Act（执行）、Assert（断言）
- **独立性** - 测试之间相互独立，可按任意顺序执行
- **快速执行** - 单元测试应该快速执行
- **可读性** - 测试代码应该清晰表达意图

## 技术栈

| 技术 | 版本 | 说明 |
|------|------|------|
| JUnit | 5.x | Java 标准测试框架 |
| Mockito | 5.x | Mock 框架，兼容 Java 21 |
| AssertJ | 3.x | 流式断言库 |
| TestContainers | 最新 | 容器化集成测试 |
| JaCoCo | 最新 | 测试覆盖率工具 |

## JUnit 5 基础

### 基础注解

```java
import org.junit.jupiter.api.*;
import static org.junit.jupiter.api.Assertions.*;

@DisplayName("用户服务测试")
class UserServiceTest {

    @BeforeAll
    static void setUpAll() {
        // 所有测试前执行一次
    }

    @BeforeEach
    void setUp() {
        // 每个测试前执行
    }

    @Test
    @DisplayName("创建用户")
    void shouldCreateUser() {
        // Arrange
        String name = "测试用户";

        // Act
        User user = userService.create(name);

        // Assert
        assertNotNull(user);
        assertEquals("测试用户", user.getName());
    }

    @AfterEach
    void tearDown() {
        // 每个测试后执行
    }

    @AfterAll
    static void tearDownAll() {
        // 所有测试后执行一次
    }
}
```

### 常用断言

```java
// 相等性断言
assertEquals(expected, actual);
assertNotEquals(expected, actual);

// 布尔断言
assertTrue(condition);
assertFalse(condition);

// 空值断言
assertNull(object);
assertNotNull(object);

// 异常断言
assertThrows(ExceptionType.class, executable);

// 分组断言 - 所有断言都会执行
assertAll(
    () -> assertEquals("John", user.getFirstName()),
    () -> assertEquals("Doe", user.getLastName())
);
```

## Mockito 使用

### 基础 Mock

```java
@ExtendWith(MockitoExtension.class)
class UserServiceTest {

    @Mock
    private UserMapper userMapper;

    @InjectMocks
    private UserService userService;

    @Test
    @DisplayName("创建用户")
    void shouldCreateUser() {
        // Arrange
        CreateUserDTO dto = new CreateUserDTO();
        dto.setName("测试用户");

        UserEntity entity = new UserEntity();
        entity.setId(1L);
        entity.setName("测试用户");

        // 设置 Mock 行为
        when(userMapper.insert(any())).thenReturn(1);
        when(userMapper.selectById(any())).thenReturn(entity);

        // Act
        User result = userService.createUser(dto);

        // Assert
        assertThat(result).isNotNull();
        assertThat(result.getName()).isEqualTo("测试用户");

        // 验证调用
        verify(userMapper).insert(any());
        verify(userMapper, times(1)).insert(any());
        verify(userMapper, never()).deleteById(any());
    }
}
```

### Mock 行为设置

```java
// thenReturn - 设置返回值
when(userMapper.selectById(1L)).thenReturn(entity);

// thenThrow - 抛出异常
when(userMapper.selectById(1L))
    .thenThrow(new BusinessException(ErrorCode.NOT_FOUND));

// 链式调用 - 多次调用不同返回
when(userMapper.selectById(1L))
    .thenReturn(entity)
    .thenThrow(new RuntimeException())
    .thenReturn(null);

// 无返回值
doNothing().when(userMapper).deleteById(1L);

// 抛出异常（无返回值）
doThrow(new RuntimeException()).when(userMapper).deleteById(1L);
```

### 参数匹配器

```java
// 精确匹配
when(userMapper.selectById(1L)).thenReturn(entity);

// any() - 匹配任意值
when(userMapper.insert(any())).thenReturn(1);

// anyLong() - 匹配任意 Long
when(userMapper.selectById(anyLong())).thenReturn(entity);

// eq() - 精确匹配
when(userMapper.insert(eq(entity))).thenReturn(1);

// argThat - 自定义匹配器
when(userMapper.insert(argThat(e -> e.getName().length() > 3)))
    .thenReturn(1);
```

## AssertJ 断言

### 基础断言

```java
import static org.assertj.core.api.Assertions.assertThat;

// 对象断言
assertThat(result).isNotNull();
assertThat(result.getName()).isEqualTo("测试");

// 数值断言
assertThat(result.getAge()).isGreaterThan(18);
assertThat(result.getScore()).isBetween(0, 100);

// 集合断言
assertThat(list).hasSize(3);
assertThat(list).contains("a", "b");
assertThat(list).doesNotContain("d");

// 链式断言
assertThat(user)
    .isNotNull()
    .hasFieldOrPropertyWithValue("id", 1L)
    .hasFieldOrPropertyWithValue("name", "测试");
```

### 异常断言

```java
assertThatThrownBy(() -> service.getById(null))
    .isInstanceOf(BusinessException.class)
    .hasMessageContaining("ID不能为空")
    .hasNoCause();
```

## 参数化测试

```java
@ParameterizedTest
@DisplayName("验证用户名格式")
@ValueSource(strings = {"", "   ", "ab", "very_long_name_over_32_characters"})
void shouldRejectInvalidUsername(String username) {
    // Arrange
    CreateUserDTO dto = new CreateUserDTO();
    dto.setName(username);

    // Act & Assert
    assertThatThrownBy(() -> userService.createUser(dto))
        .isInstanceOf(BusinessException.class)
        .hasMessageContaining("用户名格式不正确");
}

@CsvSource({
    "valid_user, true",
    "ab, false",
    "very_long_name_over_32_characters, false"
})
@ParameterizedTest
@DisplayName("验证用户名")
void shouldValidateUsername(String username, boolean expected) {
    boolean result = userService.isValidUsername(username);
    assertThat(result).isEqualTo(expected);
}

@MethodSource("provideUsers")
@ParameterizedTest
@DisplayName("处理用户列表")
void shouldProcessUsers(User user, boolean expected) {
    boolean result = userService.isValidUser(user);
    assertThat(result).isEqualTo(expected);
}

static Stream<Arguments> provideUsers() {
    return Stream.of(
        Arguments.of(new User(1L, "valid"), true),
        Arguments.of(new User(2L, ""), false)
    );
}
```

## TestContainers

### MySQL 容器测试

```java
@Testcontainers
@TestMethodOrder(OrderAnnotation.class)
class UserRepositoryTest {

    @Container
    static MySQLContainer<?> mysql = new MySQLContainer<>("mysql:8.0")
        .withDatabaseName("test_db")
        .withUsername("test")
        .withPassword("test");

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", mysql::getJdbcUrl);
        registry.add("spring.datasource.username", mysql::getUsername);
        registry.add("spring.datasource.password", mysql::getPassword);
    }

    @Autowired
    private UserRepository userRepository;

    @Test
    @Order(1)
    @DisplayName("保存用户")
    void shouldSaveUser() {
        User user = new User();
        user.setName("测试用户");
        User saved = userRepository.save(user);

        assertThat(saved.getId()).isNotNull();
        assertThat(saved.getName()).isEqualTo("测试用户");
    }
}
```

### Redis 容器测试

```java
@Testcontainers
class CacheServiceTest {

    @Container
    static GenericContainer<?> redis = new GenericContainer<>("redis:7-alpine")
        .withExposedPorts(6379);

    @Test
    @DisplayName("缓存读写")
    void shouldCacheAndRetrieve() {
        // Given
        String key = "test-key";
        String value = "test-value";

        // When
        cacheService.set(key, value);
        String retrieved = cacheService.get(key);

        // Then
        assertThat(retrieved).isEqualTo(value);
    }
}
```

## 测试数据构建器

```java
@Builder
public class UserBuilder {
    private Long id = 1L;
    private String name = "测试用户";
    private String email = "test@example.com";
    private Integer age = 25;

    public UserEntity buildEntity() {
        UserEntity entity = new UserEntity();
        entity.setId(id);
        entity.setName(name);
        entity.setEmail(email);
        entity.setAge(age);
        return entity;
    }

    public UserVO buildVO() {
        return new UserVO(id, name);
    }
}

// 使用方式
UserEntity user = UserBuilder.builder()
        .name("张三")
        .age(30)
        .buildEntity();
```

## JaCoCo 测试覆盖率

### Maven 配置

```xml
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
    </executions>
</plugin>
```

### 运行命令

```bash
# 运行测试并生成报告
mvn test jacoco:report

# 查看报告
# 打开 target/site/jacoco/index.html
```

## 测试最佳实践

### DO（应该做的）

1. **先写测试** - 遵循 TDD 原则
2. **描述性名称** - 测试方法名应该描述测试意图
3. **AAA 模式** - Arrange、Act、Assert 结构清晰
4. **测试行为而非实现** - 关注公共接口行为
5. **使用 @DisplayName** - 提供友好的测试名称
6. **Mock 外部依赖** - 隔离单元测试
7. **测试边界条件** - Null、空值、边界值
8. **测试异常路径** - 不只测试快乐路径

### DON'T（不应该做的）

1. **不要测试私有方法** - 通过公共接口测试
2. **不要在测试中使用 sleep** - 使用 Mock 或 CountdownLatch
3. **不要忽略不稳定测试** - 修复或删除
4. **不要 Mock 所有东西** - 适当使用集成测试
5. **不要跳过错误路径测试** - 异常场景同样重要

**记住**：保持测试快速、独立、可读。测试行为而非实现细节。目标覆盖率 80%+。

## 常见测试场景

### Service 层测试

```java
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {

    @Mock
    private OrderMapper orderMapper;
    @Mock
    private ProductService productService;
    @Mock
    private UserService userService;

    @InjectMocks
    private OrderService orderService;

    @Test
    @DisplayName("创建订单时扣减库存")
    void shouldDeductStockWhenCreateOrder() {
        // Arrange
        CreateOrderDTO dto = new CreateOrderDTO();
        dto.setUserId(1L);
        dto.setProducts(List.of(new OrderItem(1L, 2)));

        when(userService.getUserById(1L))
            .thenReturn(new User(1L, "张三"));
        when(productService.deductStock(1L, 2))
            .thenReturn(true);
        when(orderMapper.insert(any())).thenReturn(1);

        // Act
        Order order = orderService.createOrder(dto);

        // Assert
        assertThat(order).isNotNull();
        verify(productService).deductStock(1L, 2);
        verify(orderMapper).insert(any());
    }

    @Test
    @DisplayName("库存不足时抛出异常")
    void shouldThrowExceptionWhenStockInsufficient() {
        // Arrange
        CreateOrderDTO dto = new CreateOrderDTO();
        dto.setProducts(List.of(new OrderItem(1L, 100)));

        when(productService.deductStock(1L, 100))
            .thenReturn(false);

        // Act & Assert
        assertThatThrownBy(() -> orderService.createOrder(dto))
            .isInstanceOf(BusinessException.class)
            .hasMessageContaining("库存不足");

        verify(orderMapper, never()).insert(any());
    }
}
```

### Controller 层测试

```java
@WebMvcTest(OrderController.class)
class OrderControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @MockBean
    private OrderService orderService;

    @Test
    @DisplayName("创建订单成功")
    void shouldCreateOrder() throws Exception {
        // Arrange
        when(orderService.createOrder(any()))
            .thenReturn(Result.ok(new OrderVO(1L, "PENDING")));

        // Act & Assert
        mockMvc.perform(post("/api/orders")
                .contentType(MediaType.APPLICATION_JSON)
                .content("""
                    {
                        "userId": 1,
                        "items": [{"productId": 1, "quantity": 2}]
                    }
                """))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.code").value(200))
                .andExpect(jsonPath("$.data.status").value("PENDING"));
    }
}
```

### 并发测试

```java
@Test
@DisplayName("并发创建订单")
void shouldHandleConcurrentOrderCreation() throws Exception {
    // Arrange
    int threadCount = 10;
    ExecutorService executor = Executors.newFixedThreadPool(threadCount);
    CountDownLatch latch = new CountDownLatch(threadCount);
    AtomicInteger successCount = new AtomicInteger(0);

    // Act
    for (int i = 0; i < threadCount; i++) {
        executor.submit(() -> {
            try {
                orderService.createOrder(new CreateOrderDTO());
                successCount.incrementAndGet();
            } finally {
                latch.countDown();
            }
        });
    }

    latch.await(10, TimeUnit.SECONDS);
    executor.shutdown();

    // Assert
    assertThat(successCount.get()).isEqualTo(threadCount);
}
```

## 快速参考

### JUnit 5 注解

```java
@Test                          // 标记测试方法
@DisplayName                   // 显示名称
@Nested                       // 嵌套测试类
@ParameterizedTest            // 参数化测试
@RepeatedTest                 // 重复测试
@Timeout                      // 超时测试
@Disabled                     // 禁用测试

@BeforeEach                   // 每个测试前执行
@AfterEach                    // 每个测试后执行
@BeforeAll                    // 所有测试前执行（静态）
@AfterAll                     // 所有测试后执行（静态）

@ExtendWith                   // 注册扩展
@TempDir                      // 临时目录

@Tag                          // 标签分组
@Tag("fast")                  // 快速测试
@Tag("slow")                  // 慢速测试
```

### Spring Boot Test 注解

```java
@SpringBootTest               // 完整集成测试
@WebMvcTest                   // Web 层测试
@DataJpaTest                  // JPA 测试
@MockBean                     // Mock Bean
@Import                       // 导入配置
@ActiveProfiles               // 激活配置文件
@TestConfiguration            // 测试配置类
@AutoConfigureMockMvc         // 自动配置 MockMvc
```

### Mockito 注解

```java
@ExtendWith(MockitoExtension.class)  // 启用 Mockito
@Mock                         // 创建 Mock 对象
@Spy                          // 创建 Spy 对象（部分 Mock）
@InjectMocks                  // 自动注入 Mock 对象
@Captor                       // 参数捕获器
```

### 断言方法

```java
// JUnit 5 断言
assertEquals(expected, actual);
assertNotEquals(expected, actual);
assertTrue(condition);
assertFalse(condition);
assertNull(object);
assertNotNull(object);
assertThrows(exceptionType, executable);
assertAll(executables);  // 分组断言

// AssertJ 断言（推荐）
assertThat(actual).isEqualTo(expected);
assertThat(actual).isNotNull();
assertThat(actual).hasSize(3);
assertThatThrownBy(() -> method()).isInstanceOf(Exception.class);
```

**记住**：保持测试快速、独立、可读。测试行为而非实现细节。目标是 80%+ 测试覆盖率。
