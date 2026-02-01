---
name: springboot-tdd
description: Spring Boot 测试驱动开发，使用 JUnit 5、Mockito、MockMvc、测试覆盖率工具。用于新增功能、修复 Bug 或重构代码。
---

# Spring Boot TDD 工作流程

Spring Boot 服务测试驱动开发指南，目标测试覆盖率 80%+（单元 + 集成测试）。

## 何时使用

- 新增功能或接口
- 修复 Bug 或重构
- 新增数据访问逻辑或安全规则

## 工作流程

1. 先编写测试（测试应该失败）
2. 实现最小代码使测试通过
3. 在测试保持绿色的同时重构
4. 强制测试覆盖率（JaCoCo）

## 单元测试（JUnit 5 + Mockito）

```java
@ExtendWith(MockitoExtension.class)
class MarketServiceTest {
    @Mock
    private MarketMapper marketMapper;

    @InjectMocks
    private MarketService marketService;

    @Test
    @DisplayName("创建市场")
    void createMarket() {
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
        assertThat(result.getName()).isEqualTo("测试市场");
        verify(marketMapper).insert(any());
    }
}
```

测试模式：
- 准备-执行-断言（AAA 模式）
- 避免部分 Mock；优先显式桩定
- 对变体使用 `@ParameterizedTest`

## Web 层测试（MockMvc）

```java
@WebMvcTest(MarketController.class)
class MarketControllerTest {
    @Autowired
    private MockMvc mockMvc;

    @MockBean
    private MarketService marketService;

    @Test
    @DisplayName("返回市场列表")
    void returnsMarkets() throws Exception {
        when(marketService.list(any())).thenReturn(Page.empty());

        mockMvc.perform(get("/api/markets"))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.data.content").isArray());
    }
}
```

## 集成测试（@SpringBootTest）

```java
@SpringBootTest
@AutoConfigureMockMvc
@ActiveProfiles("test")
class MarketIntegrationTest {
    @Autowired
    private MockMvc mockMvc;

    @Test
    @DisplayName("创建市场")
    void createMarket() throws Exception {
        mockMvc.perform(post("/api/markets")
                .contentType(MediaType.APPLICATION_JSON)
                .content("""
                    {
                        "name": "测试",
                        "description": "描述",
                        "endDate": "2030-01-01T00:00:00",
                        "categories": ["通用"]
                    }
                """))
                .andExpect(status().isOk());
    }
}
```

## 持久层测试（MyBatis-Plus）

```java
@DataJpaTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
@Import(TestContainersConfig.class)
class MarketMapperTest {
    @Autowired
    private MarketMapper marketMapper;

    @Test
    @DisplayName("保存并查询")
    void saveAndFind() {
        MarketEntity entity = new MarketEntity();
        entity.setName("测试");

        marketMapper.insert(entity);

        MarketEntity found = marketMapper.selectById(entity.getId());
        assertThat(found).isNotNull();
        assertThat(found.getName()).isEqualTo("测试");
    }
}
```

## 测试覆盖率（Maven）

pom.xml 配置：

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

## 断言（AssertJ）

- 优先使用 AssertJ（`assertThat`）以提高可读性
- JSON 响应使用 `jsonPath`
- 异常使用 `assertThatThrownBy(...)`

```java
// 基础断言
.assertThat(result).isNotNull();
.assertThat(result.getName()).isEqualTo("测试");

// 集合断言
.assertThat(list).hasSize(3);
.assertThat(list).extracting("name").contains("测试");

// 异常断言
.assertThatThrownBy(() -> service.getById(null))
    .isInstanceOf(BusinessException.class)
    .hasMessageContaining("不存在");
```

## 测试数据构建器

```java
public class MarketBuilder {
    private String name = "测试市场";
    private String description = "测试描述";

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

    public Market build() {
        return new Market(null, name, MarketStatus.ACTIVE);
    }
}

// 使用方式
Market market = MarketBuilder.builder()
        .name("新市场")
        .build();
```

## CI 命令

```bash
# Maven
mvn -T 4 test
mvn verify

# 跳过测试（不推荐）
mvn -DskipTests
```

## 参数化测试

```java
@ParameterizedTest
@DisplayName("验证市场名称")
@ValueSource(strings = {"", "   "})
@NullSource
void validateMarketName(String name) {
    CreateMarketDTO dto = new CreateMarketDTO();
    dto.setName(name);

    Set<ConstraintViolation<CreateMarketDTO>> violations =
            validator.validate(dto);

    assertThat(violations).isNotEmpty();
}
```

## Mock 静态方法（MockedStatic）

```java
@Test
void testStaticMock() {
    try (MockedStatic<StaticUtil> mockedStatic = Mockito.mockStatic(StaticUtil.class)) {
        mockedStatic.when(() -> StaticUtil.staticMethod("input"))
                .thenReturn("mocked");

        String result = StaticUtil.staticMethod("input");

        assertThat(result).isEqualTo("mocked");
    }
}
```

## 最佳实践

1. **先写测试** - 总是 TDD
2. **一个测试一个断言** - 聚焦单一行为
3. **描述性测试名称** - 解释测试内容
4. **AAA 模式** - 清晰的测试结构
5. **Mock 外部依赖** - 隔离单元测试
6. **测试边界条件** - Null、空值、大值
7. **测试错误路径** - 不只是快乐路径
8. **保持测试快速** - 单元测试每个 < 50ms
9. **测试后清理** - 无副作用
10. **检查覆盖率报告** - 识别缺口

**记住**：保持测试快速、独立和确定性。测试行为而非实现细节。
