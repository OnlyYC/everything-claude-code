---
name: java-testing
description: Java 测试模式：基于 JUnit 5 + Mockito 5 + Spring Boot Test，适配 Java 21 + Spring Boot 3 + MyBatis-Plus 技术栈。涵盖单元测试、集成测试、MockMvc 测试、测试覆盖率、测试数据构建器等最佳实践。
---

# Java 测试模式

基于 JUnit 5 + Mockito 5 + Spring Boot Test，适配 Java 21 + Spring Boot 3 + MyBatis-Plus 技术栈。测试覆盖率目标 80%+。

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
| Spring Boot Test | 3.x | Spring Boot 测试支持 |
| TestContainers | 最新 | 容器化集成测试 |
| JaCoCo | 最新 | 测试覆盖率工具 |

## 单元测试（JUnit 5 + Mockito）

### 基础结构

```java
@ExtendWith(MockitoExtension.class)
@DisplayName("用户服务测试")
class UserServiceTest {

    @Mock
    private UserMapper userMapper;

    @InjectMocks
    private UserService userService;

    @Test
    @DisplayName("创建用户")
    void shouldCreateUser() {
        // Arrange（准备）
        CreateUserDTO dto = new CreateUserDTO();
        dto.setName("测试用户");
        dto.setEmail("test@example.com");

        UserEntity entity = new UserEntity();
        entity.setId(1L);
        entity.setName("测试用户");

        when(userMapper.insert(any())).thenReturn(1);
        when(userMapper.selectById(any())).thenReturn(entity);

        // Act（执行）
        User result = userService.createUser(dto);

        // Assert（断言）
        assertThat(result).isNotNull();
        assertThat(result.getName()).isEqualTo("测试用户");
        verify(userMapper).insert(any());
    }
}
```

### Mockito 注解

```java
@ExtendWith(MockitoExtension.class)
class ServiceTest {
    // 创建 Mock 对象
    @Mock
    private UserMapper userMapper;

    // 将 Mock 对象注入到被测对象
    @InjectMocks
    private UserService userService;

    // 捕获参数
    @Captor
    private ArgumentCaptor<UserEntity> captor;

    @Test
    void test() {
        // 使用 captor 捕获参数
        verify(userMapper).insert(captor.capture());
        assertThat(captor.getValue().getName()).isEqualTo("测试");
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

// thenCallRealMethod - 调用真实方法
when(userMapper.selectById(1L)).thenCallRealMethod();

// 链式调用
when(userMapper.selectById(1L))
    .thenReturn(entity)
    .thenThrow(new RuntimeException())
    .thenReturn(null); // 多次调用不同返回

// 无返回值
doNothing().when(userMapper).deleteById(1L);

// 抛出异常（无返回值）
doThrow(new RuntimeException()).when(userMapper).deleteById(1L);

// 调用真实方法（无返回值）
doCallRealMethod().when(userMapper).deleteById(1L);
```

### 参数匹配

```java
// 精确匹配
when(userMapper.selectById(1L)).thenReturn(entity);

// any() - 匹配任意值
when(userMapper.insert(any())).thenReturn(1);

// anyLong() - 匹配任意 Long
when(userMapper.selectById(anyLong())).thenReturn(entity);

// anyString() - 匹配任意字符串
when(userMapper.findByName(anyString())).thenReturn(entity);

// eq() - 精确匹配
when(userMapper.insert(eq(entity))).thenReturn(1);

// isNull() - 匹配 null
when(userMapper.selectById(isNull())).thenThrow(new Exception());

// argThat - 自定义匹配器
when(userMapper.insert(argThat(e -> e.getName().length() > 3)))
    .thenReturn(1);
```

### 验证调用
不验证调用，只做黑盒测试

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

## 测试异常

```java
@Test
@DisplayName("用户不存在时抛出异常")
void shouldThrowExceptionWhenUserNotFound() {
    // Arrange
    when(userMapper.selectById(999L)).thenReturn(null);

    // Act & Assert
    assertThatThrownBy(() -> userService.getUserById(999L))
        .isInstanceOf(BusinessException.class)
        .hasMessageContaining("用户不存在")
        .hasFieldOrPropertyWithValue("code", ErrorCode.NOT_FOUND);
}

// 或使用 assertThrows
@Test
@DisplayName("用户不存在时抛出异常")
void shouldThrowExceptionWhenUserNotFound() {
    when(userMapper.selectById(999L)).thenReturn(null);

    BusinessException exception = assertThrows(
        BusinessException.class,
        () -> userService.getUserById(999L)
    );

    assertThat(exception.getCode()).isEqualTo(ErrorCode.NOT_FOUND);
    assertThat(exception.getMessage()).contains("用户不存在");
}
```

## AssertJ 断言

```java
// 基础断言
assertThat(result).isNotNull();
assertThat(result.getName()).isEqualTo("测试");
assertThat(result.getAge()).isGreaterThan(18);
assertThat(list).isNotEmpty();
assertThat(list).hasSize(3);

// 集合断言
assertThat(list)
    .hasSize(3)
    .contains("a", "b")
    .doesNotContain("d");

// 提取属性断言
assertThat(users)
    .extracting("name")
    .containsExactly("张三", "李四");

// 链式断言
assertThat(user)
    .isNotNull()
    .hasFieldOrPropertyWithValue("id", 1L)
    .hasFieldOrPropertyWithValue("name", "测试");

// 异常断言
assertThatThrownBy(() -> service.getById(null))
    .isInstanceOf(BusinessException.class)
    .hasMessageContaining("ID不能为空")
    .hasNoCause();

// 条件断言
assertThat(user)
    .matches(u -> u.getAge() >= 18, "年龄必须大于等于18岁");

// 柔性断言（SoftAssertion）
SoftAssertions softly = new SoftAssertions();
softly.assertThat(user.getName()).isEqualTo("张三");
softly.assertThat(user.getAge()).isGreaterThan(18);
softly.assertAll(); // 统一报告所有失败
```

## Web 层测试（MockMvc）

```java
@WebMvcTest(UserController.class)
@DisplayName("用户控制器测试")
class UserControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @MockBean
    private UserService userService;

    @Test
    @DisplayName("查询用户列表")
    void shouldReturnUsers() throws Exception {
        // Arrange
        List<UserVO> users = List.of(
            new UserVO(1L, "张三"),
            new UserVO(2L, "李四")
        );
        when(userService.listUsers()).thenReturn(Result.ok(users));

        // Act & Assert
        mockMvc.perform(get("/api/users"))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.code").value(200))
                .andExpect(jsonPath("$.data").isArray())
                .andExpect(jsonPath("$.data.length()").value(2))
                .andExpect(jsonPath("$.data[0].name").value("张三"));
    }

    @Test
    @DisplayName("创建用户")
    void shouldCreateUser() throws Exception {
        // Arrange
        when(userService.createUser(any())).thenReturn(Result.ok(new UserVO(1L, "张三")));

        // Act & Assert
        mockMvc.perform(post("/api/users")
                .contentType(MediaType.APPLICATION_JSON)
                .content("""
                    {
                        "name": "张三",
                        "email": "zhangsan@example.com"
                    }
                """))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.code").value(200))
                .andExpect(jsonPath("$.data.name").value("张三"));
    }

    @Test
    @DisplayName("参数校验失败")
    void shouldReturnErrorWhenValidationFails() throws Exception {
        mockMvc.perform(post("/api/users")
                .contentType(MediaType.APPLICATION_JSON)
                .content("""
                    {
                        "name": "",
                        "email": "invalid-email"
                    }
                """))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.code").value(400));
    }
}
```

## 集成测试（@SpringBootTest）

```java
@SpringBootTest
@AutoConfigureMockMvc
@ActiveProfiles("test")
@DisplayName("用户集成测试")
class UserIntegrationTest {

    @Autowired
    private MockMvc mockMvc;

    @Autowired
    private UserService userService;

    @Autowired
    private UserMapper userMapper;

    @Test
    @DisplayName("完整的用户创建流程")
    void shouldCreateUserSuccessfully() throws Exception {
        // Act & Assert
        mockMvc.perform(post("/api/users")
                .contentType(MediaType.APPLICATION_JSON)
                .content("""
                    {
                        "name": "集成测试用户",
                        "email": "integration@test.com"
                    }
                """))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.code").value(200));

        // 验证数据库
        UserEntity entity = userMapper.selectOne(
            Wrappers.lambdaQuery(UserEntity.class)
                .eq(UserEntity::getEmail, "integration@test.com")
        );

        assertThat(entity).isNotNull();
        assertThat(entity.getName()).isEqualTo("集成测试用户");
    }
}
```

## MyBatis-Plus Mapper 测试

```java
@DataJpaTest
@Import(MyBatisPlusConfig.class)
@DisplayName("用户 Mapper 测试")
class UserMapperTest {

    @Autowired
    private UserMapper userMapper;

    @Test
    @DisplayName("插入并查询用户")
    void shouldInsertAndSelectUser() {
        // Arrange
        UserEntity entity = new UserEntity();
        entity.setName("测试用户");
        entity.setEmail("test@example.com");

        // Act
        int rows = userMapper.insert(entity);

        // Assert
        assertThat(rows).isEqualTo(1);
        assertThat(entity.getId()).isNotNull();

        UserEntity found = userMapper.selectById(entity.getId());
        assertThat(found).isNotNull();
        assertThat(found.getName()).isEqualTo("测试用户");
    }

    @Test
    @DisplayName("条件查询")
    void shouldSelectByCondition() {
        // Arrange
        userMapper.insert(createUser("张三", 20));
        userMapper.insert(createUser("李四", 25));

        // Act
        List<UserEntity> users = userMapper.selectList(
            Wrappers.lambdaQuery(UserEntity.class)
                .ge(UserEntity::getAge, 22)
                .orderByAsc(UserEntity::getAge)
        );

        // Assert
        assertThat(users).hasSize(1);
        assertThat(users.get(0).getName()).isEqualTo("李四");
    }

    private UserEntity createUser(String name, int age) {
        UserEntity entity = new UserEntity();
        entity.setName(name);
        entity.setAge(age);
        userMapper.insert(entity);
        return entity;
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
    private LocalDateTime createTime = LocalDateTime.now();

    public UserEntity buildEntity() {
        UserEntity entity = new UserEntity();
        entity.setId(id);
        entity.setName(name);
        entity.setEmail(email);
        entity.setAge(age);
        entity.setCreateTime(createTime);
        return entity;
    }

    public UserVO buildVO() {
        return new UserVO(id, name);
    }

    public CreateUserDTO buildDTO() {
        CreateUserDTO dto = new CreateUserDTO();
        dto.setName(name);
        dto.setEmail(email);
        dto.setAge(age);
        return dto;
    }
}

// 使用方式
UserEntity user = UserBuilder.builder()
        .name("张三")
        .age(30)
        .buildEntity();
```

## 静态方法 Mock（MockedStatic）

```java
@Test
@DisplayName("Mock 静态方法")
void testMockStatic() {
    try (MockedStatic<StaticUtil> mockedStatic = Mockito.mockStatic(StaticUtil.class)) {
        // Arrange
        mockedStatic.when(() -> StaticUtil.generateId())
                .thenReturn(100L);

        // Act
        Long id = StaticUtil.generateId();

        // Assert
        assertThat(id).isEqualTo(100L);

        // 验证调用
        mockedStatic.verify(() -> StaticUtil.generateId());
    }
}
```

## 测试配置

### 测试配置文件

```yaml
# application-test.yml
spring:
  datasource:
    url: jdbc:h2:mem:testdb
    driver-class-name: org.h2.Driver
  jpa:
    hibernate:
      ddl-auto: create-drop
  redis:
    host: localhost
    port: 6379
    database: 15  # 使用独立的测试数据库

logging:
  level:
    com.example.app.mapper: debug
```

### 测试配置类

```java
@TestConfiguration
public class TestConfig {

    @Bean
    @Primary
    public UserService testUserService(UserMapper userMapper) {
        return new UserService(userMapper);
    }
}
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

### 运行命令

```bash
# 运行测试
mvn test

# 生成覆盖率报告
mvn jacoco:report

# 检查覆盖率
mvn jacoco:check

# 跳过测试
mvn -DskipTests

# 运行指定测试类
mvn test -Dtest=UserServiceTest

# 运行指定测试方法
mvn test -Dtest=UserServiceTest#shouldCreateUser
```

## 测试最佳实践

### DO（应该做的）

1. **先写测试** - 遵循 TDD 原则
2. **使用描述性名称** - 测试方法名应该描述测试意图
3. **AAA 模式** - Arrange、Act、Assert 结构清晰
4. **测试行为而非实现** - 关注公共接口行为
5. **使用 @DisplayName** - 提供友好的测试名称
6. **Mock 外部依赖** - 隔离单元测试
7. **测试边界条件** - Null、空值、边界值
8. **测试异常路径** - 不只测试快乐路径
9. **保持测试快速** - 单元测试应该快速执行
10. **测试后清理** - 使用 @BeforeEach/@AfterEach

### DON'T（不应该做的）

1. **不要直接测试私有方法** - 通过公共接口测试
2. **不要在测试中使用 sleep** - 使用 Mock 或 CountdownLatch
3. **不要忽略不稳定测试** - 修复或删除
4. **不要 Mock 所有东西** - 适当使用集成测试
5. **不要跳过错误路径测试** - 异常场景同样重要
6. **不要在测试中硬编码数据** - 使用测试数据构建器
7. **不要共享测试状态** - 每个测试独立

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
