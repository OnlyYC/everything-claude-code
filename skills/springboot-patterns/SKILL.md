---
name: springboot-patterns
description: Spring Boot 架构模式、REST API 设计、分层服务、数据访问、缓存、异步处理、日志。适配 Java 21 + Spring Boot 3 + MyBatis-Plus + MySQL 技术栈。
---

# Spring Boot 开发模式

Spring Boot 架构和 API 模式，用于构建可扩展、生产级服务。

## REST API 结构

```java
@RestController
@RequestMapping("/api/markets")
@Validated
class MarketController {
    private final MarketService marketService;

    MarketController(MarketService marketService) {
        this.marketService = marketService;
    }

    @GetMapping
    Result<Page<MarketVO>> list(
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size) {
        Page<Market> markets = marketService.list(PageRequest.of(page, size));
        return Result.ok(markets.map(MarketVO::from));
    }

    @PostMapping
    Result<MarketVO> create(@Valid @RequestBody CreateMarketDTO request) {
        Market market = marketService.create(request);
        return Result.ok(MarketVO.from(market));
    }
}
```

## MyBatis-Plus Mapper 模式

```java
// Mapper 接口继承 BaseMapper
@Mapper
public interface MarketMapper extends BaseMapper<MarketEntity> {

    // 自定义查询方法
    @Select("SELECT * FROM market WHERE status = #{status} ORDER BY volume DESC")
    List<MarketEntity> findActiveByPage(@Param("status") MarketStatus status,
                                         Page<MarketEntity> page);
}

// 实体类使用 MyBatis-Plus 注解
@Data
@TableName("market")
public class MarketEntity {
    @TableId(type = IdType.AUTO)
    private Long id;

    @TableField("name")
    private String name;

    @TableField("status")
    private MarketStatus status;

    @TableField(fill = FieldFill.INSERT)
    private LocalDateTime createTime;

    @TableField(fill = FieldFill.INSERT_UPDATE)
    private LocalDateTime updateTime;

    @TableLogic
    @TableField("deleted")
    private Integer deleted;
}
```

## Service 层与事务

```java
@Service
@RequiredArgsConstructor
public class MarketService {
    private final MarketMapper marketMapper;

    @Transactional(rollbackFor = Exception.class)
    public Market create(CreateMarketDTO dto) {
        MarketEntity entity = BeanUtil.copyProperties(dto, MarketEntity.class);
        marketMapper.insert(entity);
        return BeanUtil.copyProperties(entity, Market.class);
    }

    @Transactional(readOnly = true)
    public Page<Market> list(PageRequest pageRequest) {
        Page<MarketEntity> entityPage = marketMapper.selectPage(
            new Page<>(pageRequest.getPageNumber(), pageRequest.getPageSize()),
            Wrappers.<MarketEntity>lambdaQuery()
                .orderByDesc(MarketEntity::getVolume)
        );
        return entityPage.convert(entity -> BeanUtil.copyProperties(entity, Market.class));
    }
}
```

## DTO 和 VO 转换

```java
// DTO - 数据传输对象（接收请求）
@Data
public class CreateMarketDTO {
    @NotBlank(message = "名称不能为空")
    @Size(max = 200, message = "名称长度不能超过200")
    private String name;

    @NotBlank(message = "描述不能为空")
    @Size(max = 2000, message = "描述长度不能超过2000")
    private String description;

    @NotNull(message = "结束时间不能为空")
    @FutureOrPresent(message = "结束时间必须是未来或当前时间")
    private LocalDateTime endDate;

    @NotEmpty(message = "分类不能为空")
    private List<@NotBlank String> categories;
}

// VO - 视图对象（返回响应）
@Data
@Builder
public class MarketVO {
    private Long id;
    private String name;
    private MarketStatus status;

    public static MarketVO from(Market market) {
        return MarketVO.builder()
                .id(market.getId())
                .name(market.getName())
                .status(market.getStatus())
                .build();
    }
}
```

## 统一异常处理

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public Result<Void> handleValidation(MethodArgumentNotValidException ex) {
        String message = ex.getBindingResult().getFieldErrors().stream()
                .map(e -> e.getField() + ": " + e.getDefaultMessage())
                .collect(Collectors.joining(", "));
        return Result.fail(ErrorCode.PARAM_ERROR, message);
    }

    @ExceptionHandler(AccessDeniedException.class)
    public Result<Void> handleAccessDenied() {
        return Result.fail(ErrorCode.FORBIDDEN, "无权限访问");
    }

    @ExceptionHandler(BusinessException.class)
    public Result<Void> handleBusiness(BusinessException ex) {
        return Result.fail(ex.getCode(), ex.getMessage());
    }

    @ExceptionHandler(Exception.class)
    public Result<Void> handleGeneric(Exception ex) {
        log.error("系统异常", ex);
        return Result.fail(ErrorCode.SYSTEM_ERROR, "系统繁忙，请稍后再试");
    }
}

// 自定义业务异常
@Data
public class BusinessException extends RuntimeException {
    private final ErrorCode code;

    public BusinessException(ErrorCode code, String message) {
        super(message);
        this.code = code;
    }
}
```

## 统一响应封装

```java
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class Result<T> {
    private Integer code;
    private String message;
    private T data;

    public static <T> Result<T> ok() {
        return Result.<T>builder()
                .code(200)
                .message("操作成功")
                .build();
    }

    public static <T> Result<T> ok(T data) {
        return Result.<T>builder()
                .code(200)
                .message("操作成功")
                .data(data)
                .build();
    }

    public static <T> Result<T> fail(ErrorCode errorCode, String message) {
        return Result.<T>builder()
                .code(errorCode.getCode())
                .message(message)
                .build();
    }
}

// 错误码枚举
@Getter
@AllArgsConstructor
public enum ErrorCode {
    SUCCESS(200, "操作成功"),
    PARAM_ERROR(400, "参数错误"),
    UNAUTHORIZED(401, "未登录"),
    FORBIDDEN(403, "无权限"),
    NOT_FOUND(404, "资源不存在"),
    SYSTEM_ERROR(500, "系统错误");

    private final Integer code;
    private final String message;
}
```

## 缓存（Spring Cache + Redis）

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

@Service
public class MarketCacheService {
    private final MarketMapper marketMapper;

    @Cacheable(value = "market", key = "#id", unless = "#result == null")
    public Market getById(Long id) {
        MarketEntity entity = marketMapper.selectById(id);
        if (entity == null) {
            throw new BusinessException(ErrorCode.NOT_FOUND, "市场不存在");
        }
        return BeanUtil.copyProperties(entity, Market.class);
    }

    @CacheEvict(value = "market", key = "#id")
    public void evict(Long id) {}

    @CachePut(value = "market", key = "#market.id")
    public Market update(Market market) {
        return market;
    }
}
```

## 异步处理

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

@Service
public class NotificationService {

    @Async("taskExecutor")
    public CompletableFuture<Void> sendAsync(Notification notification) {
        // 发送短信/邮件
        return CompletableFuture.completedFuture(null);
    }
}
```

## 日志（Logback）

```java
@Slf4j
@Service
public class ReportService {

    public Report generate(Long marketId) {
        log.info("生成报告 marketId={}", marketId);
        try {
            // 业务逻辑
            return new Report();
        } catch (Exception ex) {
            log.error("生成报告失败 marketId={}", marketId, ex);
            throw ex;
        }
    }
}
```

## 请求日志过滤器

```java
@Component
public class RequestLoggingFilter extends OncePerRequestFilter {
    private static final Logger log = LoggerFactory.getLogger(RequestLoggingFilter.class);

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain filterChain)
            throws ServletException, IOException {
        long start = System.currentTimeMillis();
        try {
            filterChain.doFilter(request, response);
        } finally {
            long duration = System.currentTimeMillis() - start;
            log.info("请求 method={} uri={} status={}耗时={}ms",
                    request.getMethod(), request.getRequestURI(),
                    response.getStatus(), duration);
        }
    }
}
```

## 分页和排序

```java
@Service
public class MarketService {

    public Page<Market> list(int page, int size) {
        Page<MarketEntity> entityPage = marketMapper.selectPage(
                new Page<>(page, size),
                Wrappers.<MarketEntity>lambdaQuery()
                        .orderByDesc(MarketEntity::getCreateTime)
        );
        return entityPage.convert(e -> BeanUtil.copyProperties(e, Market.class));
    }
}
```

## 限流（Redis + Lua）

```java
@Component
public class RateLimitService {
    @Autowired
    private StringRedisTemplate redisTemplate;

    /**
     * 令牌桶限流
     * @param key 限流标识
     * @param limit 令牌数量
     * @param timeout 时间窗口（秒）
     * @return 是否允许通过
     */
    public boolean allowRequest(String key, long limit, long timeout) {
        String luaScript =
                "local key = KEYS[1] " +
                "local limit = tonumber(ARGV[1]) " +
                "local timeout = tonumber(ARGV[2]) " +
                "local current = tonumber(redis.call('get', key) or '0') " +
                "if current + 1 > limit then " +
                "  return 0 " +
                "else " +
                "  redis.call('incrby', key, 1) " +
                "  if current == 0 then " +
                "    redis.call('expire', key, timeout) " +
                "  end " +
                "  return 1 " +
                "end";

        Long result = redisTemplate.execute(
                new DefaultRedisScript<>(luaScript, Long.class),
                Collections.singletonList(key),
                String.valueOf(limit),
                String.valueOf(timeout)
        );

        return result != null && result == 1;
    }
}
```

## 定时任务

```java
@Component
@Slf4j
public class ScheduledTasks {

    @Scheduled(cron = "0 0 2 * * ?") // 每天凌晨2点执行
    public void dailyReport() {
        log.info("开始执行日报任务");
        // 业务逻辑
    }

    @Scheduled(fixedRate = 60000) // 每60秒执行一次
    public void heartbeat() {
        log.debug("心跳检测");
    }
}
```

## MyBatis-Plus 配置

```java
@Configuration
@MapperScan("com.example.app.mapper")
public class MyBatisPlusConfig {

    /**
     * 分页插件
     */
    @Bean
    public MybatisPlusInterceptor mybatisPlusInterceptor() {
        MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();

        // 分页插件
        PaginationInnerInterceptor paginationInterceptor =
                new PaginationInnerInterceptor(DbType.MYSQL);
        paginationInterceptor.setMaxLimit(1000L);
        paginationInterceptor.setOverflow(false);
        interceptor.addInnerInterceptor(paginationInterceptor);

        // 乐观锁插件
        interceptor.addInnerInterceptor(new OptimisticLockerInnerInterceptor());

        return interceptor;
    }

    /**
     * 自动填充处理器
     */
    @Bean
    public MetaObjectHandler metaObjectHandler() {
        return new MetaObjectHandler() {
            @Override
            public void insertFill(MetaObject metaObject) {
                this.strictInsertFill(metaObject, "createTime",
                        LocalDateTime.class, LocalDateTime.now());
                this.strictInsertFill(metaObject, "updateTime",
                        LocalDateTime.class, LocalDateTime.now());
            }

            @Override
            public void updateFill(MetaObject metaObject) {
                this.strictUpdateFill(metaObject, "updateTime",
                        LocalDateTime.class, LocalDateTime.now());
            }
        };
    }
}
```

## 应用配置（application.yml）

```yaml
spring:
  application:
    name: app-service
  datasource:
    driver-class-name: com.mysql.cj.jdbc.Driver
    url: jdbc:mysql://localhost:3306/app?useUnicode=true&characterEncoding=utf8&useSSL=false&serverTimezone=Asia/Shanghai&allowPublicKeyRetrieval=true
    username: root
    password: password
    hikari:
      minimum-idle: 5
      maximum-pool-size: 20
      idle-timeout: 600000
      max-lifetime: 1800000
      connection-timeout: 30000
      connection-test-query: SELECT 1
  data:
    redis:
      host: localhost
      port: 6379
      password:
      database: 0
      timeout: 3000
      lettuce:
        pool:
          min-idle: 0
          max-idle: 8
          max-active: 8
          max-wait: -1

mybatis-plus:
  configuration:
    map-underscore-to-camel-case: true
    cache-enabled: false
    log-impl: org.apache.ibatis.logging.slf4j.Slf4jImpl
  global-config:
    db-config:
      id-type: auto
      logic-delete-field: deleted
      logic-delete-value: 1
      logic-not-delete-value: 0
  mapper-locations: classpath*:mapper/**/*.xml

logging:
  level:
    com.example.app.mapper: debug
  pattern:
    console: '%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{50} - %msg%n'
```

## 生产环境默认配置

- 优先构造器注入，避免字段注入
- 启用 `spring.mvc.problemdetails.enabled=true` 以支持 RFC 7807 错误（Spring Boot 3+）
- 为工作负载配置 HikariCP 连接池大小，设置超时
- 查询使用 `@Transactional(readOnly = true)`
- 通过 `@NonNull` 和 `Optional` 适当地强制空值安全
- 使用 MyBatis-Plus 的 LambdaQueryWrapper 避免硬编码字段名

**记住**：保持控制器精简、服务专注、Mapper 简单、错误集中处理。优先考虑可维护性和可测试性。
