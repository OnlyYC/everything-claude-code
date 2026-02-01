---
name: springboot-tdd
description: Spring Boot 测试驱动开发：@WebMvcTest、@DataJpaTest、@MockBean、Testcontainers、MockMvc。专注 Spring Boot 特定测试模式。
version: 1.1.0
tech_stack: [Java 21, Spring Boot 3.2, JUnit 5, Mockito, Testcontainers]
tools: Read, Write, Edit, Bash, Grep, Glob
related_skills:
  tdd-workflow: "通用 TDD 方法论"
  springboot-verification: "Spring Boot 项目验证"
  eval-harness: "Eval 驱动开发框架"
---

# Spring Boot TDD 工作流程

Spring Boot 特定的测试驱动开发指南。本技能专注于 **Spring Boot 测试注解和工具**，通用 TDD 方法论请参考 `tdd-workflow` 技能。

## 技能职责划分

| 技能 | 职责 |
|------|------|
| `springboot-tdd` | Spring Boot 特定测试：@WebMvcTest、@DataJpaTest、@MockBean、Testcontainers |
| `tdd-workflow` | 通用 TDD 方法论：红-绿-重构、测试设计原则、测试组织 |

**核心区别**：本技能聚焦 "如何使用 Spring Boot 测试工具"，而非 "为什么要写测试"。

## 何时使用

- 新增功能或接口
- 修复 Bug 或重构
- 新增数据访问逻辑或安全规则

## TDD 工作流程

1. **红** - 先编写测试（测试应该失败）
2. **绿** - 实现最小代码使测试通过
3. **重构** - 在测试保持绿色的同时重构
4. **验证** - 强制测试覆盖率（JaCoCo 80%+）

## 单元测试（JUnit 5 + Mockito）

### 基础结构

```java
@ExtendWith(MockitoExtension.class)
@DisplayName("市场服务测试")
class MarketServiceTest {

    @Mock
    private MarketMapper marketMapper;

    @InjectMocks
    private MarketService marketService;

    @Test
    @DisplayName("创建市场")
    void shouldCreateMarket() {
        // Arrange（准备）
        CreateMarketDTO dto = new CreateMarketDTO("测试市场", "描述",
            LocalDateTime.now(), List.of("分类"));

        MarketEntity entity = new MarketEntity();
        entity.setId(1L);
        entity.setName("测试市场");

        when(marketMapper.insert(any())).thenReturn(1);
        when(marketMapper.selectById(any())).thenReturn(entity);

        // Act（执行）
        Market result = marketService.create(dto);

        // Assert（断言）
        assertThat(result).isNotNull();
        assertThat(result.getName()).isEqualTo("测试市场");
        verify(marketMapper).insert(any());
    }
}
```

### Mockito 注解说明

```java
@ExtendWith(MockitoExtension.class)  // 启用 Mockito
class ServiceTest {
    // @Mock - 创建 Mock 对象（模拟依赖）
    @Mock
    private MarketMapper marketMapper;

    // @InjectMocks - 将 Mock 对象注入到被测对象
    @InjectMocks
    private MarketService marketService;

    // @Captor - 捕获参数（用于验证传递的参数）
    @Captor
    private ArgumentCaptor<MarketEntity> captor;

    @Test
    void test() {
        // 使用 captor 捕获参数
        verify(marketMapper).insert(captor.capture());
        assertThat(captor.getValue().getName()).isEqualTo("测试");
    }
}
```

### Mock 行为设置

```java
// thenReturn - 设置返回值
when(marketMapper.selectById(1L)).thenReturn(entity);

// thenThrow - 抛出异常
when(marketMapper.selectById(1L))
    .thenThrow(new BusinessException(ErrorCode.NOT_FOUND));

// 链式调用 - 多次调用不同返回
when(marketMapper.selectById(1L))
    .thenReturn(entity)
    .thenThrow(new RuntimeException())
    .thenReturn(null);

// 无返回值 - doNothing
doNothing().when(marketMapper).deleteById(1L);

// 抛出异常（无返回值）- doThrow
doThrow(new RuntimeException()).when(marketMapper).deleteById(1L);
```

### 参数匹配器

```java
// 精确匹配
when(marketMapper.selectById(1L)).thenReturn(entity);

// any() - 匹配任意值
when(marketMapper.insert(any())).thenReturn(1);

// anyLong() - 匹配任意 Long
when(marketMapper.selectById(anyLong())).thenReturn(entity);

// eq() - 精确匹配（配合 any() 使用）
when(marketMapper.insert(eq(entity))).thenReturn(1);

// argThat - 自定义匹配器
when(marketMapper.insert(argThat(e -> e.getName().length() > 3)))
    .thenReturn(1);
```

### 验证调用

```java
// verify - 确认方法被调用
verify(marketMapper).insert(any());

// times(n) - 确认调用次数
verify(marketMapper, times(1)).insert(any());
verify(marketMapper, never()).deleteById(any());
verify(marketMapper, atLeastOnce()).selectById(any());
```

## Web 层测试（@WebMvcTest）

### 基础 Controller 测试

```java
@WebMvcTest(MarketController.class)
@DisplayName("市场控制器测试")
class MarketControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @MockBean
    private MarketService marketService;

    @Test
    @DisplayName("返回市场列表")
    void shouldReturnMarkets() throws Exception {
        // Given
        when(marketService.list(any())).thenReturn(Page.empty());

        // When & Then
        mockMvc.perform(get("/api/markets"))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.data.content").isArray());
    }

    @Test
    @DisplayName("创建市场")
    void shouldCreateMarket() throws Exception {
        // Given
        CreateMarketDTO dto = new CreateMarketDTO(
            "新市场", "市场描述", LocalDateTime.now().plusDays(7), List.of("通用")
        );
        Market market = new Market(1L, "新市场", MarketStatus.ACTIVE);
        when(marketService.create(any())).thenReturn(market);

        // When & Then
        mockMvc.perform(post("/api/markets")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(dto)))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.code").value(200))
                .andExpect(jsonPath("$.data.id").value(1))
                .andExpect(jsonPath("$.data.name").value("新市场"));
    }

    @Test
    @DisplayName("参数验证失败")
    void shouldReturnValidationError() throws Exception {
        // Given
        CreateMarketDTO dto = new CreateMarketDTO("", "描述", null, List.of());

        // When & Then
        mockMvc.perform(post("/api/markets")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(dto)))
                .andExpect(status().isBadRequest())
                .andExpect(jsonPath("$.code").value(400));
    }
}
```

### JSON Path 断言

```java
// 基础断言
.andExpect(jsonPath("$.code").value(200))
.andExpect(jsonPath("$.message").value("操作成功"))

// 数组断言
.andExpect(jsonPath("$.data.content").isArray())
.andExpect(jsonPath("$.data.content.length()").value(2))
.andExpect(jsonPath("$.data.content[0].name").value("市场1"))

// 嵌套属性
.andExpect(jsonPath("$.data.user.id").value(1))

// 存在性检查
.andExpect(jsonPath("$.data.createdAt").exists())
.andExpect(jsonPath("$.data.deleted").doesNotExist())

// 正则匹配
.andExpect(jsonPath("$.data.email").value("\\w+@\\w+\\.\\w+"))
```

## 集成测试（@SpringBootTest）

### 完整集成测试

```java
@SpringBootTest
@AutoConfigureMockMvc
@ActiveProfiles("test")
@DisplayName("市场集成测试")
class MarketIntegrationTest {

    @Autowired
    private MockMvc mockMvc;

    @Autowired
    private MarketMapper marketMapper;

    @Test
    @DisplayName("完整的创建市场流程")
    void shouldCreateMarketSuccessfully() throws Exception {
        // When & Then
        mockMvc.perform(post("/api/markets")
                .contentType(MediaType.APPLICATION_JSON)
                .content("""
                    {
                        "name": "集成测试市场",
                        "description": "测试描述",
                        "endDate": "2030-01-01T00:00:00",
                        "categories": ["通用"]
                    }
                """))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.code").value(200));

        // 验证数据库
        MarketEntity entity = marketMapper.selectOne(
            Wrappers.lambdaQuery(MarketEntity.class)
                .eq(MarketEntity::getName, "集成测试市场")
        );

        assertThat(entity).isNotNull();
        assertThat(entity.getName()).isEqualTo("集成测试市场");
    }
}
```

## 持久层测试（@DataJpaTest）

### Mapper/Repository 测试

```java
@DataJpaTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
@Import(TestContainersConfig.class)
@DisplayName("市场 Mapper 测试")
class MarketMapperTest {

    @Autowired
    private MarketMapper marketMapper;

    @Test
    @DisplayName("保存并查询")
    void shouldSaveAndFind() {
        // Given
        MarketEntity entity = new MarketEntity();
        entity.setName("测试市场");

        // When
        marketMapper.insert(entity);

        // Then
        MarketEntity found = marketMapper.selectById(entity.getId());
        assertThat(found).isNotNull();
        assertThat(found.getName()).isEqualTo("测试市场");
    }

    @Test
    @DisplayName("条件查询")
    void shouldSelectByCondition() {
        // Given
        marketMapper.insert(createMarket("市场1", MarketStatus.ACTIVE));
        marketMapper.insert(createMarket("市场2", MarketStatus.ACTIVE));
        marketMapper.insert(createMarket("市场3", MarketStatus.CLOSED));

        // When
        List<MarketEntity> active = marketMapper.selectList(
            Wrappers.lambdaQuery(MarketEntity.class)
                .eq(MarketEntity::getStatus, MarketStatus.ACTIVE)
        );

        // Then
        assertThat(active).hasSize(2);
    }

    private MarketEntity createMarket(String name, MarketStatus status) {
        MarketEntity entity = new MarketEntity();
        entity.setName(name);
        entity.setStatus(status);
        marketMapper.insert(entity);
        return entity;
    }
}
```

## Testcontainers 集成测试

### MySQL Testcontainers

```java
@Testcontainers
@TestMethodOrder(OrderAnnotation.class)
@DisplayName("市场仓库测试（Testcontainers）")
class MarketRepositoryTest {

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
    private MarketRepository marketRepository;

    @Test
    @Order(1)
    @DisplayName("保存市场")
    void shouldSaveMarket() {
        // Given
        Market market = Market.builder()
            .name("测试市场")
            .description("测试描述")
            .status(MarketStatus.ACTIVE)
            .build();

        // When
        Market saved = marketRepository.save(market);

        // Then
        assertThat(saved.getId()).isNotNull();
        assertThat(saved.getName()).isEqualTo("测试市场");
    }

    @Test
    @Order(2)
    @DisplayName("根据 ID 查询")
    void shouldFindById() {
        // When
        Optional<Market> found = marketRepository.findById(1L);

        // Then
        assertThat(found).isPresent();
        assertThat(found.get().getName()).isEqualTo("测试市场");
    }
}
```

### Redis Testcontainers

```java
@Testcontainers
@DisplayName("缓存服务测试")
class CacheServiceTest {

    @Container
    static GenericContainer<?> redis = new GenericContainer<>("redis:7-alpine")
        .withExposedPorts(6379);

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.data.redis.host", redis::getHost);
        registry.add("spring.data.redis.port", () -> redis.getMappedPort(6379));
    }

    @Autowired
    private CacheService cacheService;

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

## 测试覆盖率（JaCoCo）

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
# 运行测试并生成报告
mvn test jacoco:report

# 检查覆盖率
mvn jacoco:check

# 查看报告
# 打开 target/site/jacoco/index.html
```

## 断言（AssertJ）

### 基础断言

```java
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
assertThat(list).extracting("name").contains("测试");

// 链式断言
assertThat(user)
    .isNotNull()
    .hasFieldOrPropertyWithValue("id", 1L)
    .hasFieldOrPropertyWithValue("name", "测试");
```

### 异常断言

```java
// assertThatThrownBy
assertThatThrownBy(() -> service.getById(null))
    .isInstanceOf(BusinessException.class)
    .hasMessageContaining("ID不能为空")
    .hasNoCause();

// 或使用 assertThrows
BusinessException exception = assertThrows(
    BusinessException.class,
    () -> service.getById(null)
);
assertThat(exception.getCode()).isEqualTo(ErrorCode.PARAM_ERROR);
```

## 测试数据构建器

### Builder 模式

```java
public class MarketBuilder {
    private String name = "测试市场";
    private String description = "测试描述";
    private MarketStatus status = MarketStatus.ACTIVE;

    public static MarketBuilder builder() {
        return new MarketBuilder();
    }

    public MarketBuilder name(String name) {
        this.name = name;
        return this;
    }

    public MarketBuilder description(String description) {
        this.description = description;
        return this;
    }

    public MarketBuilder status(MarketStatus status) {
        this.status = status;
        return this;
    }

    public Market build() {
        return new Market(null, name, description, status);
    }

    public MarketEntity buildEntity() {
        MarketEntity entity = new MarketEntity();
        entity.setName(name);
        entity.setDescription(description);
        entity.setStatus(status);
        return entity;
    }
}

// 使用方式
Market market = MarketBuilder.builder()
    .name("新市场")
    .status(MarketStatus.PENDING)
    .build();
```

## 参数化测试

### @ValueSource

```java
@ParameterizedTest
@DisplayName("验证市场名称")
@ValueSource(strings = {"", "   "})
void shouldRejectInvalidName(String name) {
    // Given
    CreateMarketDTO dto = new CreateMarketDTO();
    dto.setName(name);

    // When & Then
    Set<ConstraintViolation<CreateMarketDTO>> violations =
            validator.validate(dto);

    assertThat(violations).isNotEmpty();
}
```

### @CsvSource

```java
@ParameterizedTest
@CsvSource({
    "valid_name, true",
    "ab, false",
    ", false"
})
@DisplayName("验证市场名称格式")
void shouldValidateName(String name, boolean expected) {
    boolean result = marketService.isValidName(name);
    assertThat(result).isEqualTo(expected);
}
```

### @MethodSource

```java
@ParameterizedTest
@MethodSource("provideMarkets")
@DisplayName("处理市场")
void shouldProcessMarket(Market market, boolean expected) {
    boolean result = marketService.isValid(market);
    assertThat(result).isEqualTo(expected);
}

static Stream<Arguments> provideMarkets() {
    return Stream.of(
        Arguments.of(new Market(1L, "Valid", MarketStatus.ACTIVE), true),
        Arguments.of(new Market(2L, "", MarketStatus.ACTIVE), false)
    );
}
```

## Mock 静态方法

```java
@Test
@DisplayName("Mock 静态方法")
void testMockStatic() {
    try (MockedStatic<StaticUtil> mockedStatic = Mockito.mockStatic(StaticUtil.class)) {
        // Given
        mockedStatic.when(() -> StaticUtil.generateId())
                .thenReturn(100L);

        // When
        Long id = StaticUtil.generateId();

        // Then
        assertThat(id).isEqualTo(100L);

        // 验证调用
        mockedStatic.verify(() -> StaticUtil.generateId());
    }
}
```

## Spring Boot 测试注解对比

| 注解 | 用途 | 加载内容 | 适用场景 |
|------|------|---------|---------|
| @WebMvcTest | Controller 层 | MVC 组件 + MockMvc | 测试 API 端点 |
| @JsonTest | JSON 序列化 | JSON 相关 | 测试 DTO/VO |
| @DataJpaTest | Repository 层 | JPA 组件 | 测试数据访问 |
| @MockBean | Mock Bean | 替换 Spring Bean | 隔离依赖 |
| @SpyBean | Spy Bean | 部分替换 | 验证调用 |
| @SpringBootTest | 完整集成 | 全部上下文 | 端到端测试 |

## 安全测试

```java
@SpringBootTest
@AutoConfigureMockMvc
@DisplayName("安全测试")
class SecurityTest {

    @Autowired
    private MockMvc mockMvc;

    @WithMockUser(username = "user", roles = "USER")
    @Test
    @DisplayName("用户可访问")
    void shouldAccessWithUserRole() throws Exception {
        mockMvc.perform(get("/api/user/profile"))
                .andExpect(status().isOk());
    }

    @WithMockUser(username = "admin", roles = "ADMIN")
    @Test
    @DisplayName("管理员可删除")
    void shouldDeleteWithAdminRole() throws Exception {
        mockMvc.perform(delete("/api/admin/users/1"))
                .andExpect(status().isNoContent());
    }

    @Test
    @DisplayName("未登录拒绝访问")
    void shouldDenyAccessWithoutAuth() throws Exception {
        mockMvc.perform(get("/api/user/profile"))
                .andExpect(status().isUnauthorized());
    }
}
```

## 最佳实践

### DO（应该做的）

1. **先写测试** - 总是 TDD
2. **一个测试一个断言** - 聚焦单一行为
3. **描述性测试名称** - 使用 `@DisplayName`
4. **AAA 模式** - Arrange、Act、Assert 结构清晰
5. **Mock 外部依赖** - 隔离单元测试
6. **测试边界条件** - Null、空值、边界值
7. **测试错误路径** - 不只测试快乐路径
8. **保持测试快速** - 单元测试每个 < 50ms
9. **测试后清理** - 无副作用
10. **检查覆盖率报告** - 识别缺口

### DON'T（不应该做的）

1. **不要直接测试私有方法** - 通过公共接口测试
2. **不要在测试中使用 sleep** - 使用 Mock 或 CountdownLatch
3. **不要忽略不稳定测试** - 修复或删除
4. **不要 Mock 所有东西** - 适当使用集成测试
5. **不要跳过错误路径测试** - 异常场景同样重要
6. **不要在测试中硬编码数据** - 使用测试数据构建器
7. **不要共享测试状态** - 每个测试独立

**记住**：保持测试快速、独立和确定性。测试行为而非实现细节。目标覆盖率 80%+。

## 相关技能

- `java-testing` - 完整的 Java 测试指南
- `tdd-workflow` - 通用 TDD 工作流程
- `springboot-verification` - 项目验证流程
- `eval-harness` - Eval 驱动开发框架
