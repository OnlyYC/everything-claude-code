---
name: coding-standards
description: 代码标准与最佳实践：适配 Java 21 + Spring Boot 3 + MyBatis-Plus + MySQL 技术栈。涵盖代码品质原则、Java 编码规范、命名规范、注释规范、性能优化、测试标准、代码异味检测等最佳实践。
---

# 代码标准与最佳实践

基于 Java 21 + Spring Boot 3 + MyBatis-Plus + MySQL 技术栈的通用代码标准。

## 代码品质原则

### 1. 可读性优先
- 代码被阅读的次数远多于被撰写的次数
- 使用清晰的变量和函数名称
- 优先使用自文档化的代码而非注释
- 保持一致的格式化
- 遵循阿里巴巴 Java 开发手册

### 2. KISS（保持简单）
- 使用最简单的解决方案
- 避免过度工程
- 不做过早优化
- 易于理解 > 聪明的代码

### 3. DRY（不重复自己）
- 将共用逻辑提取为方法
- 建立可重用的组件
- 在模块间共享工具函数
- 避免复制粘贴程序设计

### 4. YAGNI（你不会需要它）
- 在需要之前不要构建功能
- 避免推测性的通用化
- 只在需要时增加复杂度
- 从简单开始，需要时再重构

## Java 编码规范

### 变量命名

```java
// 良好：描述性名称
String marketSearchQuery = "election";
boolean isUserAuthenticated = true;
BigDecimal totalRevenue = BigDecimal.valueOf(1000);
int maxRetryCount = 3;

// 不良：不清楚的名称
String q = "election";
boolean flag = true;
int x = 1000;
```

### 函数命名

```java
// 良好：动词-名词模式
List<Market> fetchMarketData(Long marketId) { }
BigDecimal calculateSimilarity(List<Double> a, List<Double> b) { }
boolean isValidEmail(String email) { }

// 不良：不清楚或只有名词
List<Market> market(Long id) { }
BigDecimal similarity(List<Double> a, List<Double> b) { }
boolean email(String e) { }
```

### 不可变性模式

```java
// 良好：使用 final 关键字
public final class MarketDto {
    private final Long id;
    private final String name;
    private final MarketStatus status;

    // 只提供 getter，不提供 setter
}

// 良好：使用 Lombok @Value
@Value
public class MarketDto {
    private Long id;
    private String name;
    private MarketStatus status;
}

// 不良：可变字段
public class MarketDto {
    private String name;
    public void setName(String name) {
        this.name = name;  // 不良
    }
}
```

### 集合处理

```java
// 良好：使用 Stream API
List<String> names = markets.stream()
    .map(Market::getName)
    .filter(Objects::nonNull)
    .collect(Collectors.toList());

// 良好：不可变集合
List<String> list = List.of("a", "b", "c");

// 不良：直接修改
List<String> list = new ArrayList<>();
list.add("item");
```

### 错误处理

```java
// 良好：完整的错误处理
public Market fetchData(Long marketId) {
    try {
        Market market = marketMapper.selectById(marketId);
        if (market == null) {
            throw new BusinessException(ErrorCode.MARKET_NOT_FOUND);
        }
        return market;
    } catch (Exception e) {
        log.error("查询市场失败 marketId={}", marketId, e);
        throw new SystemException(ErrorCode.SYSTEM_ERROR, "查询市场失败");
    }
}

// 不良：无错误处理
public Market fetchData(Long marketId) {
    return marketMapper.selectById(marketId);
}
```

### 并发处理

```java
// 良好：可能时并行执行
CompletableFuture<List<User>> usersFuture = CompletableFuture.supplyAsync(() -> userService.listUsers());
CompletableFuture<List<Market>> marketsFuture = CompletableFuture.supplyAsync(() -> marketService.listMarkets());
CompletableFuture<Statistics> statsFuture = CompletableFuture.supplyAsync(() -> statisticsService.getStats());

CompletableFuture.allOf(usersFuture, marketsFuture, statsFuture).join();

// 不良：不必要的顺序执行
List<User> users = userService.listUsers();
List<Market> markets = marketService.listMarkets();
Statistics stats = statisticsService.getStats();
```

### 类型安全

```java
// 良好：正确的类型
public record Market(
    Long id,
    String name,
    MarketStatus status,
    LocalDateTime createTime
) {}

public Market getMarket(Long id) {
    // 实现
}

// 不良：使用 Object
public Object getMarket(Long id) {
    // 实现
}
```

## Spring Boot 组件规范

### Controller 组件

```java
// 良好：RESTful Controller
@RestController
@RequestMapping("/api/markets")
@RequiredArgsConstructor
public class MarketController {

    private final MarketService marketService;

    @GetMapping
    public Result<Page<MarketVO>> list(
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size) {
        return Result.ok(marketService.list(PageRequest.of(page, size)));
    }

    @PostMapping
    public Result<MarketVO> create(@Valid @RequestBody CreateMarketDTO dto) {
        return Result.ok(marketService.create(dto));
    }
}

// 不良：职责不清
@RestController
public class MarketController {

    @GetMapping
    public Result<Page<MarketVO>> list() {
        // 直接在 Controller 中查询数据库
        List<Market> markets = marketMapper.selectList(null);
        return Result.ok(markets);
    }
}
```

### Service 组件

```java
// 良好：业务逻辑清晰
@Service
@RequiredArgsConstructor
public class OrderService {

    private final OrderMapper orderMapper;
    private final ProductService productService;
    private final UserService userService;

    @Transactional(rollbackFor = Exception.class)
    public OrderVO createOrder(CreateOrderDTO dto) {
        // 业务逻辑
        validateOrder(dto);
        User user = userService.getById(dto.getUserId());
        checkStock(dto.getItems());
        OrderEntity entity = buildOrder(dto, user);
        orderMapper.insert(entity);
        return OrderVO.from(entity);
    }
}

// 不良：职责混乱
@Service
public class OrderService {

    @Autowired
    private OrderMapper orderMapper;
    @Autowired
    private UserService userService;

    public void createOrder(CreateOrderDTO dto) {
        // 直接操作 HttpServletRequest
    }
}
```

### Mapper 组件

```java
// 良好：Mapper 只负责数据访问
@Mapper
public interface MarketMapper extends BaseMapper<MarketEntity> {

    @Select("SELECT * FROM markets WHERE status = #{status}")
    List<MarketEntity> findByStatus(@Param("status") MarketStatus status);

    @Select("SELECT * FROM markets WHERE name LIKE CONCAT('%', #{keyword}, '%')")
    List<MarketEntity> searchByName(@Param("keyword") String keyword);
}

// 不良：在 Mapper 中包含业务逻辑
@Mapper
public interface MarketMapper extends BaseMapper<MarketEntity> {

    @Select("SELECT * FROM orders o WHERE o.total_amount > (SELECT AVG(total_amount) FROM orders)")
    List<OrderEntity> findHighValueOrders();
}
```

## 数据库最佳实践

### 查询优化

```java
// 良好：只查询需要的字段
@Select("SELECT id, name, status FROM markets WHERE status = #{status}")
List<Market> findActiveMarkets(@Param("status") MarketStatus status);

// 或使用 LambdaQueryWrapper 指定字段
LambdaQueryWrapper<MarketEntity> wrapper = Wrappers.lambdaQuery();
wrapper.select(MarketEntity::getId, MarketEntity::getName, MarketEntity::getStatus)
       .eq(MarketEntity::getStatus, MarketStatus.ACTIVE);

// 不良：查询所有字段
List<MarketEntity> markets = marketMapper.selectList(null);
```

### N+1 查询避免

```java
// 不良：N+1 查询
List<Order> orders = orderMapper.selectList(null);
for (Order order : orders) {
    User user = userMapper.selectById(order.getUserId());  // N 次查询
    order.setUser(user);
}

// 良好：批量查询
List<Order> orders = orderMapper.selectList(null);
Set<Long> userIds = orders.stream()
        .map(Order::getUserId)
        .collect(Collectors.toSet());

Map<Long, User> userMap = userService.listByIds(userIds).stream()
        .collect(Collectors.toMap(User::getId, Function.identity()));

orders.forEach(order -> {
    order.setUser(userMap.get(order.getUserId()));
});
```

### 事务管理

```java
// 良好：事务方法只读操作使用 readOnly
@Transactional(rollbackFor = Exception.class)
public void createOrder(CreateOrderDTO dto) {
    // 写操作
}

@Transactional(readOnly = true)
public Page<OrderVO> listOrders(PageRequest pageRequest) {
    // 只读查询
}

// 不良：不加 readOnly 的查询方法
@Transactional
public Page<OrderVO> listOrders(PageRequest pageRequest) {
    return orderMapper.selectPage(pageRequest, null);
}
```

## 缓存最佳实践

```java
// 良好：使用 Spring Cache 注解
@Service
@RequiredArgsConstructor
public class UserService {

    @Cacheable(value = "user", key = "#id", unless = "#result == null")
    public UserVO getById(Long id) {
        return UserVO.from(userMapper.selectById(id));
    }

    @CacheEvict(value = "user", key = "#id")
    public void deleteUser(Long id) {
        userMapper.deleteById(id);
    }
}

// 良好：编程式缓存
@Service
@RequiredArgsConstructor
public class ProductService {

    private final ProductMapper productMapper;
    private final StringRedisTemplate redisTemplate;

    public ProductVO getById(Long id) {
        String key = "product:" + id;
        String cached = redisTemplate.opsForValue().get(key);

        if (StrUtil.isNotBlank(cached)) {
            return JSONUtil.toBean(cached, ProductVO.class);
        }

        ProductEntity entity = productMapper.selectById(id);
        ProductVO vo = ProductVO.from(entity);

        redisTemplate.opsForValue().set(key, JSONUtil.toJsonStr(vo), 5, TimeUnit.MINUTES);
        return vo;
    }
}
```

## 异常处理最佳实践

### 统一异常体系

```java
// 良好：自定义异常体系
public class BusinessException extends BaseException {
    public BusinessException(ErrorCode code, String message) {
        super(code, message);
    }
}

@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(BusinessException.class)
    public Result<Void> handleBusinessException(BusinessException ex) {
        return Result.fail(ex.getCode(), ex.getMessage());
    }
}

// 不良：每个方法单独处理异常
@GetMapping
public Result<MarketVO> getMarket(Long id) {
    try {
        Market market = marketService.getById(id);
        return Result.ok(market);
    } catch (Exception e) {
        if (e instanceof BusinessException) {
            return Result.fail(((BusinessException) e).getCode(), e.getMessage());
        }
        return Result.fail(ErrorCode.SYSTEM_ERROR, "系统错误");
    }
}
```

## 测试标准

### 测试结构（AAA 模式）

```java
// 测试用例结构：Arrange（准备）、Act（执行）、Assert（断言）
@Test
@DisplayName("计算余弦相似度")
void shouldCalculateCosineSimilarity() {
    // Arrange（准备）
    List<Double> vector1 = List.of(1.0, 0.0, 0.0);
    List<Double> vector2 = List.of(0.0, 1.0, 0.0);

    // Act（执行）
    double similarity = calculateCosineSimilarity(vector1, vector2);

    // Assert（断言）
    assertThat(similarity).isCloseTo(0.0, within(0.001));
}
```

### 测试命名

```java
// 良好：描述性测试名称
@Test
@DisplayName("当市场不存在时抛出异常")
void shouldThrowExceptionWhenMarketNotFound() { }

@Test
@DisplayName("OpenAI API key 缺失时抛出异常")
void shouldThrowExceptionWhenOpenAIKeyMissing() { }

@Test
@DisplayName("Redis 不可用时回退到子串搜索")
void shouldFallbackToSubstringSearchWhenRedisUnavailable() { }

// 不良：模糊的测试名称
@Test
void test() { }
@Test
void testSearch() { }
```

## 项目结构规范

### Maven 项目结构

```
project/
├── pom.xml
├── src/
│   ├── main/
│   │   ├── java/
│   │   │   └── com/
│   │   │       └── example/
│   │   │           ├── config/          # 配置类
│   │   │           ├── controller/      # 控制器层
│   │   │           ├── service/         # 业务逻辑层
│   │   │           │   └── impl/      # 服务实现
│   │   │           ├── mapper/          # MyBatis Mapper
│   │   │           ├── entity/          # 实体类
│   │   │           ├── domain/          # 领域模型
│   │   │           ├── dto/             # 数据传输对象
│   │   │           ├── vo/              # 视图对象
│   │   │           ├── enums/           # 枚举类
│   │   │           ├── exception/       # 自定义异常
│   │   │           ├── util/            # 工具类
│   │   │           └── common/          # 公共类
│   │   └── resources/
│   │       ├── application.yml           # 主配置
│   │       ├── application-dev.yml       # 开发环境
│   │       ├── application-prod.yml      # 生产环境
│   │       └── mapper/                   # MyBatis XML
│   └── test/
│       └── java/                         # 测试代码
└── docs/                                 # 文档
```

## 代码异味检测

### 1. 过长方法

```java
// 不良：方法超过 50 行
public void processOrderData() {
    // 100 行代码
}

// 良好：拆分为较小的方法
public void processOrderData() {
    OrderData data = collectOrderData();
    OrderData transformed = transformOrderData(data);
    saveOrderData(transformed);
}
```

### 2. 过深嵌套

```java
// 不良：5 层以上嵌套
if (user != null) {
    if (user.isAdmin()) {
        if (market != null) {
            if (market.isActive()) {
                if (hasPermission()) {
                    // 做某事
                }
            }
        }
    }
}

// 良好：提前返回
if (user == null) return;
if (!user.isAdmin()) return;
if (market == null) return;
if (!market.isActive()) return;
if (!hasPermission()) return;

// 做某事
```

### 3. 魔术数字

```java
// 不良：无解释的数字
if (retryCount > 3) { }
Thread.sleep(500);

// 良好：命名常量
private static final int MAX_RETRY_COUNT = 3;
private static final long DEBOUNCE_DELAY_MS = 500;

if (retryCount > MAX_RETRY_COUNT) { }
Thread.sleep(DEBOUNCE_DELAY_MS);
```

### 4. 上帝类

```java
// 不良：类承担过多职责
public class OrderManager {
    public void createOrder() { }
    public void sendEmail() { }
    public void generateReport() { }
    public void processPayment() { }
}

// 良好：职责单一
@Service
public class OrderService { }
@Service
public class NotificationService { }
@Service
public class ReportService { }
```

### 5. 循环复杂度过高

```java
// 不良：复杂的嵌套循环
for (Order order : orders) {
    for (OrderItem item : order.getItems()) {
        for (Product product : products) {
            if (order.getId().equals(product.getOrderId())) {
                // 复杂逻辑
            }
        }
    }
}

// 良好：使用 Stream 简化
Map<Long, Product> productMap = products.stream()
    .collect(Collectors.toMap(Product::getOrderId, Function.identity()));

orders.stream()
    .flatMap(order -> order.getItems().stream())
    .filter(item -> productMap.containsKey(item.getProductId()))
    .forEach(item -> {
        Product product = productMap.get(item.getProductId());
        // 处理逻辑
    });
```

## 性能优化

### 数据库查询

```java
// 良好：批量查询
List<Long> userIds = users.stream()
    .map(User::getId)
    .collect(Collectors.toList());

List<User> userList = userService.listByIds(userIds);

// 不良：循环查询
for (User user : users) {
    User found = userService.getById(user.getId());
}
```

### 连接池配置

```yaml
# application.yml
spring:
  datasource:
    hikari:
      minimum-idle: 5
      maximum-pool-size: 20
      idle-timeout: 600000
      max-lifetime: 1800000
      connection-timeout: 30000
```

### 缓存配置

```java
@Configuration
@EnableCaching
public class CacheConfig {

    @Bean
    public RedisCacheManager cacheManager(RedisConnectionFactory factory) {
        RedisCacheConfiguration config = RedisCacheConfiguration.defaultCacheConfig()
                .entryTtl(Duration.ofMinutes(30))
                .serializeKeysWith(RedisSerializationContext.SerializationPair
                        .fromSerializer(new StringRedisSerializer()))
                .serializeValuesWith(RedisSerializationContext.SerializationPair
                        .fromSerializer(new GenericJackson2JsonRedisSerializer()));

        return RedisCacheManager.builder(factory)
                .cacheDefaults(config)
                .build();
    }
}
```

### 日志配置

```xml
<!-- logback-spring.xml -->
<configuration>
    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{50} - %msg%n</pattern>
        </encoder>
    </appender>
</configuration>
```

## 工具类使用

### Hutool 工具类

```java
// 字符串工具
StrUtil.isBlank(str);
StrUtil.isNotEmpty(str);
StrUtil.format("模板 {}", arg);

// 集合工具
CollUtil.isNotEmpty(list);
CollUtil.isEmpty(list);

// 对象工具
BeanUtil.copyProperties(source, target);

// JSON 工具
JSONUtil.toJsonStr(object);
JSONUtil.toBean(jsonStr, clazz);

// 日期工具
DateUtil.now();
DateUtil.format(date, "yyyy-MM-dd HH:mm:ss");
```

### Guava 工具类

```java
// 不可变集合
List<String> list = Lists.newArrayList();
Map<String, String> map = Maps.newHashMap();
Set<String> set = Sets.newHashSet();

// 不可变集合
ImmutableList.of("a", "b", "c");
ImmutableMap.of("key", "value");

// 函数式接口
Functions.forMap(map);
Predicates.alwaysTrue();
```

## 并发编程

### 线程池配置

```java
@Configuration
@EnableAsync
public class AsyncConfig {

    @Bean("taskExecutor")
    public Executor taskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(4);
        executor.setMaxPoolSize(8);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("async-");
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        executor.initialize();
        return executor;
    }
}
```

### CompletableFuture

```java
@Service
public class OrderService {

    @Async("taskExecutor")
    public CompletableFuture<Void> sendNotification(Notification notification) {
        return CompletableFuture.runAsync(() -> {
            notificationService.send(notification);
        });
    }

    // 并行执行
    public OrderResult processOrder(Order order) {
        CompletableFuture<User> userFuture = CompletableFuture.supplyAsync(
            () -> userService.getById(order.getUserId())
        );
        CompletableFuture<Market> marketFuture = CompletableFuture.supplyAsync(
            () -> marketService.getById(order.getMarketId())
        );

        CompletableFuture.allOf(userFuture, marketFuture).join();

        User user = userFuture.join();
        Market market = marketFuture.join();

        return buildResult(user, market);
    }
}
```

## 注释规范

### Javadoc 注释

```java
/**
 * 市场服务
 *
 * @author Claude
 * @since 1.0.0
 */
@Service
public class MarketService {

    /**
     * 根据ID查询市场
     *
     * @param id 市场ID
     * @return 市场信息
     * @throws BusinessException 当市场不存在时抛出
     */
    public Market getById(Long id) {
        // 实现
    }

    /**
     * 创建市场
     *
     * @param dto 创建市场DTO
     * @return 创建的市场信息
     * @throws IllegalArgumentException 参数校验失败时抛出
     */
    public Market create(CreateMarketDTO dto) {
        // 实现
    }
}
```

### 行内注释

```java
// 良好：解释"为什么"而非"什么"
// 使用指数退避以避免在服务中断时压垮 API
long delay = Math.min(1000 * (long) Math.pow(2, retryCount), 30000);

// 不良：陈述显而易见的事实
// 将计数器加 1
count++;
// 设置用户名称
user.setName(name);
```

## 常用注解

```java
// Spring 注解
@RestController          // REST 控制器
@RequestMapping        // 请求映射
@GetMapping             // GET 映射
@PostMapping            // POST 映射
@PutMapping             // PUT 映射
@DeleteMapping          // DELETE 映射
@PathVariable          // 路径变量
@RequestParam          // 请求参数
@RequestBody           // 请求体

@Service               // 服务组件
@Repository            // 数据访问组件
@Configuration         // 配置类
@Component            // 通用组件
@Bean                 // Bean 定义

@Autowired            // 依赖注入
@Value                // 配置值注入

@Transactional           // 事务
@Async                 // 异步方法

@Valid                // 参数校验
@Validated             // 分组校验

// Lombok 注解
@Data                 // getter/setter/equals/hashCode/toString
@Builder              // 建造者模式
@AllArgsConstructor    // 全参构造器
@NoArgsConstructor      // 无参构造器
@RequiredArgsConstructor // 必填参数构造器
@Slf4j                // 日志对象

// MyBatis-Plus 注解
@TableName            // 表名映射
@TableId              // 主键映射
@TableField           // 字段映射
@TableLogic           // 逻辑删除
@Version              // 乐观锁

// Validation 注解
@NotNull             // 不能为 null
@NotBlank           // 不能为空字符串
@NotEmpty            // 集合不能为空
@Size                // 大小范围
@Min/@Max            // 数值范围
@Email               // 邮箱格式
@Pattern             // 正则表达式

// 测试注解
@Test                 // 测试方法
@DisplayName         // 显示名称
@ParameterizedTest    // 参数化测试
@RepeatedTest        // 重复测试
@TempDir              // 临时目录
@Mock                // Mock 对象
@MockBean             // Mock Bean
@.Autowired          // 自动装配
```

## 常见错误避免

### 1. 使用 == 比较对象

```java
// 不良
if (user1 == user2) { }

// 良好
if (Objects.equals(user1, user2)) { }
```

### 2. 忽略异常

```java
// 不良
try {
    doSomething();
} catch (Exception e) {}

// 良好
try {
    doSomething();
} catch (Exception e) {
    log.error("操作失败", e);
    throw new BusinessException(ErrorCode.OPERATION_FAILED, "操作失败");
}
```

### 3. 硬编码配置

```java
// 不良
String url = "http://localhost:8080";

// 良好
@Value("${app.api.url}")
private String apiUrl;
```

### 4. 使用 System.out

```java
// 不良
System.out.println("debug");

// 良好
log.debug("debug");
```

### 5. 忽略日志级别

```java
// 良好：在生产环境使用适当的日志级别
if (log.isDebugEnabled()) {
    log.debug("详细调试信息: {}", data);
}

log.info("关键业务流程");
log.warn("需要注意的异常情况");
log.error("影响系统运行的异常", ex);
```

**记住**：遵循《阿里巴巴 Java 开发手册》，保持代码简洁、清晰、可维护。代码品质是不可协商的，清晰、可维护的代码能实现快速开发和自信的重构。
