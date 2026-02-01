---
name: backend-patterns
description: 后端架构模式：适配 Java 21 + Spring Boot 3 + MyBatis-Plus + MySQL 技术栈。涵盖 REST API 设计、分层架构、数据库优化、缓存策略、异常处理、认证授权、限流、异步处理、日志监控等最佳实践。
---

# 后端开发模式

基于 Java 21 + Spring Boot 3 + MyBatis-Plus + MySQL 技术栈的后端架构模式和最佳实践。

## REST API 设计

### 统一 API 结构

```java
// 基于资源的 URL 设计
@RestController
@RequestMapping("/api/users")
public class UserController {

    // 查询列表
    @GetMapping
    Result<Page<UserVO>> list(
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size,
            @RequestParam(required = false) String keyword) {
        Page<UserVO> users = userService.list(page, size, keyword);
        return Result.ok(users);
    }

    // 查询单个资源
    @GetMapping("/{id}")
    Result<UserVO> getById(@PathVariable Long id) {
        UserVO user = userService.getById(id);
        return Result.ok(user);
    }

    // 创建资源
    @PostMapping
    Result<UserVO> create(@Valid @RequestBody CreateUserDTO dto) {
        UserVO user = userService.create(dto);
        return Result.ok(user);
    }

    // 替换资源（全量更新）
    @PutMapping("/{id}")
    Result<UserVO> replace(@PathVariable Long id, @Valid @RequestBody UpdateUserDTO dto) {
        UserVO user = userService.update(id, dto);
        return Result.ok(user);
    }

    // 部分更新
    @PatchMapping("/{id}")
    Result<UserVO> partialUpdate(@PathVariable Long id, @RequestBody Map<String, Object> fields) {
        UserVO user = userService.partialUpdate(id, fields);
        return Result.ok(user);
    }

    // 删除资源
    @DeleteMapping("/{id}")
    Result<Void> delete(@PathVariable Long id) {
        userService.delete(id);
        return Result.ok();
    }
}
```

### 查询参数规范

```java
// 分页、排序、过滤参数
@GetMapping
Result<Page<UserVO>> list(
    // 分页参数
    @RequestParam(defaultValue = "0") int page,
    @RequestParam(defaultValue = "20") int size,
    @RequestParam(required = false) String sortField,
    @RequestParam(required = false) String sortOrder,

    // 过滤参数
    @RequestParam(required = false) String status,
    @RequestParam(required = false) @DateTimeFormat(pattern = "yyyy-MM-dd") LocalDate startDate,
    @RequestParam(required = false) @DateTimeFormat(pattern = "yyyy-MM-dd") LocalDate endDate,

    // 搜索关键词
    @RequestParam(required = false) String keyword
) {
    PageRequest pageRequest = PageRequest.of(page, size);
    // 业务逻辑
}
```

## 分层架构

### Controller 层

```java
@RestController
@RequestMapping("/api/orders")
@RequiredArgsConstructor
public class OrderController {

    private final OrderService orderService;

    @PostMapping
    @Transactional(rollbackFor = Exception.class)
    public Result<OrderVO> createOrder(@Valid @RequestBody CreateOrderDTO dto) {
        OrderVO order = orderService.createOrder(dto);
        return Result.ok(order);
    }
}
```

### Service 层

```java
@Service
@RequiredArgsConstructor
public class OrderService {

    private final OrderMapper orderMapper;
    private final ProductService productService;
    private final UserService userService;
    private final OrderEventPublisher eventPublisher;

    @Transactional(rollbackFor = Exception.class)
    public OrderVO createOrder(CreateOrderDTO dto) {
        // 1. 参数校验
        validateOrderDTO(dto);

        // 2. 查询用户
        User user = userService.getById(dto.getUserId());
        if (user == null) {
            throw new BusinessException(ErrorCode.USER_NOT_FOUND);
        }

        // 3. 检查库存
        for (OrderItem item : dto.getItems()) {
            boolean available = productService.checkStock(item.getProductId(), item.getQuantity());
            if (!available) {
                throw new BusinessException(ErrorCode.STOCK_INSUFFICIENT);
            }
        }

        // 4. 创建订单
        OrderEntity entity = buildOrderEntity(dto, user);
        orderMapper.insert(entity);

        // 5. 扣减库存
        for (OrderItem item : dto.getItems()) {
            productService.deductStock(item.getProductId(), item.getQuantity());
        }

        // 6. 发布订单创建事件（异步处理）
        eventPublisher.publishOrderCreatedEvent(entity.getId());

        // 7. 返回结果
        return OrderVO.from(entity);
    }

    @Transactional(readOnly = true)
    public Page<OrderVO> listOrders(int page, int size, OrderQuery query) {
        Page<OrderEntity> entityPage = orderMapper.selectPage(
                new Page<>(page, size),
                buildQueryWrapper(query)
        );
        return entityPage.convert(OrderVO::from);
    }

    private LambdaQueryWrapper<OrderEntity> buildQueryWrapper(OrderQuery query) {
        return Wrappers.lambdaQuery();
        // 构建查询条件
    }
}
```

### Mapper 层

```java
@Mapper
public interface OrderMapper extends BaseMapper<OrderEntity> {

    // 自定义查询方法
    @Select("SELECT * FROM orders WHERE user_id = #{userId} AND status = #{status}")
    List<OrderEntity> findByUserIdAndStatus(
            @Param("userId") Long userId,
            @Param("status") OrderStatus status
    );

    // 复杂查询使用 XML 映射文件
    List<OrderVO> selectOrderWithDetails(@Param("userId") Long userId);
}
```

### 实体类

```java
@Data
@TableName("orders")
public class OrderEntity {

    @TableId(type = IdType.AUTO)
    private Long id;

    @TableField("user_id")
    private Long userId;

    @TableField("order_no")
    private String orderNo;

    @TableField("total_amount")
    private BigDecimal totalAmount;

    @TableField("status")
    private OrderStatus status;

    @TableField(fill = FieldFill.INSERT)
    private LocalDateTime createTime;

    @TableField(fill = FieldFill.INSERT_UPDATE)
    private LocalDateTime updateTime;

    @TableLogic
    @TableField("deleted")
    private Integer deleted;
}
```

## 数据库优化

### 只查询需要的字段

```java
// 错误 - 查询所有字段
List<UserEntity> users = userMapper.selectList(null);

// 正确 - 只查询需要的字段
@Select("SELECT id, name, email FROM users WHERE status = #{status}")
List<User> selectActiveUsers(@Param("status") Integer status);

// 或使用 QueryWrapper 指定字段
LambdaQueryWrapper<UserEntity> wrapper = Wrappers.lambdaQuery();
wrapper.select(UserEntity::getId, UserEntity::getName, UserEntity::getEmail)
       .eq(UserEntity::getStatus, 1);
```

### 避免 N+1 查询问题

```java
// 错误 - N+1 查询
List<OrderEntity> orders = orderMapper.selectList(null);
for (OrderEntity order : orders) {
    User user = userMapper.selectById(order.getUserId());  // N 次查询
    order.setUser(user);
}

// 正确 - 批量查询
List<OrderEntity> orders = orderMapper.selectList(null);
Set<Long> userIds = orders.stream()
        .map(OrderEntity::getUserId)
        .collect(Collectors.toSet());

Map<Long, User> userMap = userService.listByIds(userIds).stream()
        .collect(Collectors.toMap(User::getId, Function.identity()));

orders.forEach(order -> {
    order.setUser(userMap.get(order.getUserId()));
});
```

### 批量操作

```java
// 批量插入
@Service
@RequiredArgsConstructor
public class UserService {

    private final UserMapper userMapper;

    public void batchInsert(List<CreateUserDTO> dtos) {
        if (CollUtil.isEmpty(dtos)) {
            return;
        }

        List<UserEntity> entities = dtos.stream()
                .map(this::toEntity)
                .collect(Collectors.toList());

        // 使用 MyBatis-Plus 批量插入
        userMapper.insert(entities);
    }
}
```

### 事务管理

```java
@Service
@RequiredArgsConstructor
public class OrderService {

    // 编程式事务
    @Autowired
    private TransactionTemplate transactionTemplate;

    public void createOrderWithItems(CreateOrderDTO orderDto, List<OrderItem> items) {
        transactionTemplate.execute(status -> {
            // 所有操作在同一事务中
            OrderEntity order = createOrder(orderDto);
            createOrderItems(order.getId(), items);
            // 异常会自动回滚
        });
    }

    // 声明式事务（推荐）
    @Transactional(rollbackFor = Exception.class)
    public void createOrderWithItems(CreateOrderDTO orderDto, List<OrderItem> items) {
        OrderEntity order = createOrder(orderDto);
        createOrderItems(order.getId(), items);
    }

    // 只读事务
    @Transactional(readOnly = true)
    public Page<OrderVO> listOrders(Pageable pageable) {
        return orderMapper.selectPage(pageable, null);
    }

    // 指定事务传播行为
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void logOperation(String operation) {
        // 新事务，不受调用方事务影响
    }
}
```

## 缓存策略

### Spring Cache 注解

```java
@Service
@RequiredArgsConstructor
public class UserService {

    private final UserMapper userMapper;

    @Cacheable(value = "user", key = "#id", unless = "#result == null")
    public UserVO getById(Long id) {
        UserEntity entity = userMapper.selectById(id);
        if (entity == null) {
            throw new BusinessException(ErrorCode.USER_NOT_FOUND);
        }
        return UserVO.from(entity);
    }

    @CacheEvict(value = "user", key = "#id")
    public void deleteUser(Long id) {
        userMapper.deleteById(id);
    }

    @CachePut(value = "user", key = "#result.id")
    public UserVO updateUser(UpdateUserDTO dto) {
        UserEntity entity = userMapper.selectById(dto.getId());
        BeanUtil.copyProperties(dto, entity);
        userMapper.updateById(entity);
        return UserVO.from(entity);
    }

    @CacheEvict(value = "user", allEntries = true)
    public void clearCache() {
        // 清空所有缓存
    }
}
```

### Redis 缓存配置

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

### 编程式缓存

```java
@Service
@RequiredArgsConstructor
public class ProductService {

    private final ProductMapper productMapper;
    private final StringRedisTemplate redisTemplate;

    public ProductVO getById(Long id) {
        String key = "product:" + id;

        // 先查缓存
        String cached = redisTemplate.opsForValue().get(key);
        if (StrUtil.isNotBlank(cached)) {
            return JSONUtil.toBean(cached, ProductVO.class);
        }

        // 缓存未命中，查数据库
        ProductEntity entity = productMapper.selectById(id);
        if (entity == null) {
            throw new BusinessException(ErrorCode.PRODUCT_NOT_FOUND);
        }

        ProductVO vo = ProductVO.from(entity);

        // 写入缓存，5 分钟过期
        redisTemplate.opsForValue().set(key, JSONUtil.toJsonStr(vo), 5, TimeUnit.MINUTES);

        return vo;
    }

    public void evictCache(Long id) {
        String key = "product:" + id;
        redisTemplate.delete(key);
    }
}
```

## 异常处理

### 统一异常体系

```java
// 基础异常类
@Data
@Getter
public abstract class BaseException extends RuntimeException {
    private final ErrorCode code;

    public BaseException(ErrorCode code, String message) {
        super(message);
        this.code = code;
    }
}

// 业务异常
public class BusinessException extends BaseException {
    public BusinessException(ErrorCode code, String message) {
        super(code, message);
    }
}

// 系统异常
public class SystemException extends BaseException {
    public SystemException(ErrorCode code, String message) {
        super(code, message);
    }
}
```

### 全局异常处理器

```java
@RestControllerAdvice
@Slf4j
public class GlobalExceptionHandler {

    @ExceptionHandler(BusinessException.class)
    public Result<Void> handleBusinessException(BusinessException ex) {
        log.warn("业务异常: {}", ex.getMessage());
        return Result.fail(ex.getCode(), ex.getMessage());
    }

    @ExceptionHandler(SystemException.class)
    public Result<Void> handleSystemException(SystemException ex) {
        log.error("系统异常: {}", ex.getMessage(), ex);
        return Result.fail(ex.getCode(), "系统繁忙，请稍后再试");
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public Result<Void> handleValidation(MethodArgumentNotValidException ex) {
        String message = ex.getBindingResult().getFieldErrors().stream()
                .map(e -> e.getField() + ": " + e.getDefaultMessage())
                .collect(Collectors.joining(", "));
        log.warn("参数校验失败: {}", message);
        return Result.fail(ErrorCode.PARAM_ERROR, message);
    }

    @ExceptionHandler(Exception.class)
    public Result<Void> handleException(Exception ex) {
        log.error("未知异常", ex);
        return Result.fail(ErrorCode.SYSTEM_ERROR, "系统错误");
    }
}
```

### 错误码枚举

```java
@Getter
@AllArgsConstructor
public enum ErrorCode {
    SUCCESS(200, "操作成功"),
    PARAM_ERROR(400, "参数错误"),
    UNAUTHORIZED(401, "未登录"),
    FORBIDDEN(403, "无权限"),
    NOT_FOUND(404, "资源不存在"),
    SYSTEM_ERROR(500, "系统错误"),

    // 业务错误码（从 1000 开始）
    USER_NOT_FOUND(1001, "用户不存在"),
    STOCK_INSUFFICIENT(1002, "库存不足"),
    ORDER_NOT_FOUND(1003, "订单不存在"),
    ORDER_STATUS_ERROR(1004, "订单状态错误");

    private final Integer code;
    private final String message;
}
```

## 认证与授权

### JWT 工具类

```java
@Component
@Slf4j
public class JwtUtil {

    @Value("${jwt.secret}")
    private String secret;

    @Value("${jwt.expiration}")
    private Long expiration;

    public String generateToken(Long userId, String username) {
        Map<String, Object> claims = new HashMap<>();
        claims.put("userId", userId);
        claims.put("username", username);

        return Jwts.builder()
                .setClaims(claims)
                .setSubject(String.valueOf(userId))
                .setIssuedAt(new Date())
                .setExpiration(new Date(System.currentTimeMillis() + expiration))
                .signWith(SignatureAlgorithm.HS512, secret)
                .compact();
    }

    public Claims parseToken(String token) {
        try {
            return Jwts.parserBuilder()
                    .setSigningKey(secret)
                    .build()
                    .parseClaimsJws(token)
                    .getBody();
        } catch (JwtException e) {
            throw new BusinessException(ErrorCode.UNAUTHORIZED, "Token 无效");
        }
    }

    public Long getUserIdFromToken(String token) {
        Claims claims = parseToken(token);
        return Long.valueOf(claims.getSubject());
    }
}
```

### 认证拦截器

```java
@Component
@Slf4j
public class AuthInterceptor implements HandlerInterceptor {

    @Autowired
    private JwtUtil jwtUtil;

    @Override
    public boolean preHandle(HttpServletRequest request,
                            HttpServletResponse response,
                            Object handler) {
        // 跨域预检请求直接通过
        if ("OPTIONS".equals(request.getMethod())) {
            return true;
        }

        // 获取 Token
        String token = request.getHeader("Authorization");
        if (StrUtil.isBlank(token) || !token.startsWith("Bearer ")) {
            throw new BusinessException(ErrorCode.UNAUTHORIZED, "未登录");
        }

        token = token.substring(7);

        try {
            Long userId = jwtUtil.getUserIdFromToken(token);
            request.setAttribute("userId", userId);
            return true;
        } catch (Exception e) {
            throw new BusinessException(ErrorCode.UNAUTHORIZED, "Token 无效");
        }
    }
}
```

### 注册拦截器

```java
@Configuration
public class WebMvcConfig implements WebMvcConfigurer {

    @Autowired
    private AuthInterceptor authInterceptor;

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(authInterceptor)
                .addPathPatterns("/api/**")
                .excludePathPatterns("/api/auth/login", "/api/auth/register");
    }
}
```

### 权限注解

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface RequirePermission {
    String value();
}

@Aspect
@Component
public class PermissionAspect {

    @Autowired
    private UserService userService;

    @Around("@annotation(requirePermission)")
    public Object checkPermission(ProceedingJoinPoint joinPoint,
                                   RequirePermission requirePermission) throws Throwable {
        HttpServletRequest request = ((ServletRequestAttributes) RequestContextHolder
                .getRequestAttributes()).getRequest();

        Long userId = (Long) request.getAttribute("userId");
        User user = userService.getById(userId);

        String permission = requirePermission.value();
        if (!userService.hasPermission(user, permission)) {
            throw new BusinessException(ErrorCode.FORBIDDEN, "无权限");
        }

        return joinPoint.proceed();
    }
}

// 使用方式
@GetMapping("/admin/users")
@RequirePermission("user:read")
public Result<List<UserVO>> listUsers() {
    return Result.ok(userService.listUsers());
}
```

## 限流

### Redis + Lua 限流

```java
@Component
@Slf4j
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

### 限流拦截器

```java
@Component
@Slf4j
public class RateLimitInterceptor implements HandlerInterceptor {

    @Autowired
    private RateLimitService rateLimitService;

    @Override
    public boolean preHandle(HttpServletRequest request,
                            HttpServletResponse response,
                            Object handler) {
        String ip = getClientIp(request);
        String key = "rate_limit:" + ip;

        // 100 请求/分钟
        boolean allowed = rateLimitService.allowRequest(key, 100, 60);

        if (!allowed) {
            throw new BusinessException(ErrorCode.TOO_MANY_REQUESTS, "请求过于频繁");
        }

        return true;
    }

    private String getClientIp(HttpServletRequest request) {
        String ip = request.getHeader("X-Forwarded-For");
        if (StrUtil.isBlank(ip)) {
            ip = request.getRemoteAddr();
        }
        return ip;
    }
}
```

## 异步处理

### 异步配置

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

### 异步服务

```java
@Service
@Slf4j
public class NotificationService {

    @Async("taskExecutor")
    public CompletableFuture<Void> sendNotification(Notification notification) {
        try {
            // 发送短信/邮件
            sendSms(notification);
            sendEmail(notification);
            return CompletableFuture.completedFuture(null);
        } catch (Exception e) {
            log.error("发送通知失败", e);
            return CompletableFuture.failedFuture(e);
        }
    }
}
```

### 事件发布

```java
@Component
@Slf4j
public class OrderEventPublisher {

    @Autowired
    private ApplicationEventPublisher eventPublisher;

    public void publishOrderCreatedEvent(Long orderId) {
        OrderCreatedEvent event = new OrderCreatedEvent(orderId);
        eventPublisher.publishEvent(event);
    }
}

@Component
@Slf4j
public class OrderEventListener {

    @Autowired
    private NotificationService notificationService;

    @EventListener
    @Async("taskExecutor")
    public void handleOrderCreated(OrderCreatedEvent event) {
        log.info("处理订单创建事件: {}", event.getOrderId());
        // 发送通知
        notificationService.sendOrderNotification(event.getOrderId());
    }
}
```

## 日志与监控

### 日志配置

```xml
<!-- logback-spring.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <springProfile name="dev">
        <root level="DEBUG">
            <appender-ref ref="CONSOLE"/>
        </root>
    </springProfile>

    <springProfile name="prod">
        <root level="INFO">
            <appender-ref ref="FILE"/>
        </root>
    </springProfile>

    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{50} - %msg%n</pattern>
            <charset>UTF-8</charset>
        </encoder>
    </appender>

    <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>logs/app.log</file>
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <fileNamePattern>logs/app.%d{yyyy-MM-dd}.log</fileNamePattern>
            <maxHistory>30</maxHistory>
        </rollingPolicy>
        <encoder>
            <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{50} - %msg%n</pattern>
        </encoder>
    </appender>
</configuration>
```

### 请求日志拦截器

```java
@Component
@Slf4j
public class RequestLoggingInterceptor extends HandlerInterceptor {

    @Override
    public boolean preHandle(HttpServletRequest request,
                            HttpServletResponse response,
                            Object handler) {
        request.setAttribute("startTime", System.currentTimeMillis());
        return true;
    }

    @Override
    public void afterCompletion(HttpServletRequest request,
                               HttpServletResponse response,
                               Object handler,
                               Exception ex) {
        Long startTime = (Long) request.getAttribute("startTime");
        long duration = System.currentTimeMillis() - startTime;

        log.info("请求 method={} uri={} status={}耗时={}ms",
                request.getMethod(),
                request.getRequestURI(),
                response.getStatus(),
                duration);
    }
}
```

## 定时任务

```java
@Component
@Slf4j
public class ScheduledTasks {

    @Autowired
    private OrderService orderService;

    // 每天凌晨 2 点执行
    @Scheduled(cron = "0 0 2 * * ?")
    public void dailyReport() {
        log.info("开始执行日报任务");
        try {
            orderService.generateDailyReport();
        } catch (Exception e) {
            log.error("日报任务执行失败", e);
        }
    }

    // 每 5 分钟执行一次
    @Scheduled(fixedRate = 5 * 60 * 1000)
    public void heartbeat() {
        log.debug("心跳检测");
    }

    // 上一次执行完成后延迟 10 分钟再执行
    @Scheduled(fixedDelay = 10 * 60 * 1000)
    public void cleanExpiredData() {
        log.info("清理过期数据");
    }
}
```

## MyBatis-Plus 最佳实践

### 配置类

```java
@Configuration
@MapperScan("com.example.app.mapper")
public class MyBatisPlusConfig {

    @Bean
    public MybatisPlusInterceptor mybatisPlusInterceptor() {
        MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();

        // 分页插件
        PaginationInnerInterceptor paginationInterceptor =
                new PaginationInnerInterceptor(DbType.MYSQL);
        paginationInterceptor.setMaxLimit(1000L);
        interceptor.addInnerInterceptor(paginationInterceptor);

        // 乐观锁插件
        interceptor.addInnerInterceptor(new OptimisticLockerInnerInterceptor());

        // 逻辑删除插件（自动填充已配置 @TableLogic）
        return interceptor;
    }

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

### 条件构造器

```java
@Service
@RequiredArgsConstructor
public class UserService {

    private final UserMapper userMapper;

    public Page<UserVO> searchUsers(UserQuery query) {
        LambdaQueryWrapper<UserEntity> wrapper = Wrappers.lambdaQuery();

        // 等值条件
        wrapper.eq(StrUtil.isNotBlank(query.getName()),
                UserEntity::getName, query.getName());

        // 模糊查询
        wrapper.like(StrUtil.isNotBlank(query.getKeyword()),
                UserEntity::getName, query.getKeyword());

        // 范围查询
        wrapper.ge(query.getMinAge() != null,
                UserEntity::getAge, query.getMinAge());
        wrapper.le(query.getMaxAge() != null,
                UserEntity::getAge, query.getMaxAge());

        // in 条件
        wrapper.in(CollUtil.isNotEmpty(query.getStatusList()),
                UserEntity::getStatus, query.getStatusList());

        // 排序
        if (StrUtil.isNotBlank(query.getSortField())) {
            if ("asc".equalsIgnoreCase(query.getSortOrder())) {
                wrapper.orderByAsc(UserEntity::getCreateTime);
            } else {
                wrapper.orderByDesc(UserEntity::getCreateTime);
            }
        }

        Page<UserEntity> entityPage = userMapper.selectPage(
                new Page<>(query.getPage(), query.getSize()),
                wrapper
        );

        return entityPage.convert(UserVO::from);
    }
}
```

**记住**：遵循分层架构，保持 Controller 精简、Service 专注、Mapper 简单。优先考虑可维护性和可测试性。
