# 项目级 CLAUDE.md 示例

这是一个项目级 CLAUDE.md 文件示例，请放置于项目根目录。

## 项目概述

基于 Java 21 + Spring Boot 3 + Spring MVC + MyBatis 3 + MyBatis-Plus + Lombok + JWT + Maven + Redis + MySQL + Docker 的后端服务项目。

## 核心规范

### 1. 代码组织

- 遵循标准 Maven 项目结构：分层架构（Controller -> Service -> Repository/Mapper）
- 宁可多写小文件，不写大文件
- 高内聚、低耦合
- 单文件 200-400 行为宜，上限 800 行
- 按功能域分包：`com.company.project.module`，禁止按类型分包

### 2. 代码风格

- 代码、注释、文档中禁止使用 emoji
- 使用 Lombok 简化样板代码：`@Data`, `@Builder`, `@RequiredArgsConstructor`, `@Slf4j`
- 使用 Spring Validation（`@Valid`, `@Validated`）进行参数校验
- 使用 `record` 定义不可变数据传输对象（DTO）
- 类型明显时使用 `var` 进行类型推断
- 禁止使用 `System.out.println` - 统一使用 `@Slf4j` logger
- 使用 `@ControllerAdvice` + 自定义异常统一处理异常
- 禁止返回 `null` - 使用 `Optional` 或空集合

### 3. 测试规范

- 测试驱动开发（TDD）：先写测试，后写实现
- 代码覆盖率最低 80%
- 使用 JUnit 5 + Mockito 编写单元测试
- 使用 `@SpringBootTest` 编写集成测试
- 使用 Testcontainers 进行真实数据库/Redis 测试
- Controller 层统一使用 `MockMvc` 进行测试

### 4. 安全规范

- 禁止硬编码密钥等敏感信息
- 使用 Spring Profiles + 配置文件管理敏感数据
- 启用 Spring Security + JWT 认证
- 使用 `@PreAuthorize` 进行方法级权限控制
- SQL 查询必须参数化（MyBatis `#{param}`）
- 接口入参使用 `@Valid` + Jakarta Validation 校验
- 敏感数据加密存储（密码使用 BCrypt）

## 目录结构

```
src/main/java/com/company/project/
|-- common/               # 通用模块
|   |-- aspect/          # AOP 切面
|   |-- config/          # 配置类
|   |-- constant/        # 常量定义
|   |-- exception/       # 自定义异常
|   |-- response/        # 统一响应封装
|   '-- security/        # 安全配置（JWT、Security）
|-- controller/          # 控制器层
|-- dto/                 # 数据传输对象
|   |-- request/         # 请求 DTO
|   '-- response/        # 响应 DTO
|-- entity/              # 实体类（数据库映射）
|-- enums/               # 枚举定义
|-- mapper/              # MyBatis Mapper 接口
|-- service/             # 服务层
|   |-- impl/            # 服务实现
|   '--ContextHolder.java # 上下文持有者（当前用户等）
|-- util/                # 工具类
'-- Application.java     # 启动类

src/main/resources/
|-- mapper/              # MyBatis XML 映射文件
|-- application.yml      # 主配置文件
|-- application-dev.yml  # 开发环境
|-- application-prod.yml # 生产环境
'-- logback-spring.xml   # 日志配置

src/test/java/
|-- unit/                # 单元测试
'-- integration/         # 集成测试
```

## 核心模式

### 统一响应格式

```java
@Data
@Builder
public class Result<T> {
    private Integer code;
    private String message;
    private T data;

    public static <T> Result<T> success(T data) {
        return Result.<T>builder()
            .code(200)
            .message("操作成功")
            .data(data)
            .build();
    }

    public static <T> Result<T> error(Integer code, String message) {
        return Result.<T>builder()
            .code(code)
            .message(message)
            .build();
    }
}
```

### 全局异常处理

```java
@RestControllerAdvice
@Slf4j
public class GlobalExceptionHandler {

    @ExceptionHandler(BusinessException.class)
    public Result<?> handleBusinessException(BusinessException e) {
        log.warn("业务异常: {}", e.getMessage());
        return Result.error(e.getCode(), e.getMessage());
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public Result<?> handleValidation(MethodArgumentNotValidException e) {
        String message = e.getBindingResult().getAllErrors().get(0).getDefaultMessage();
        return Result.error(400, message);
    }

    @ExceptionHandler(Exception.class)
    public Result<?> handleException(Exception e) {
        log.error("系统异常", e);
        return Result.error(500, "系统繁忙，请稍后重试");
    }
}
```

### Service 层模式

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class UserService {

    private final UserMapper userMapper;
    private final RedisTemplate<String, Object> redisTemplate;

    @Transactional(readOnly = true)
    public UserVO getById(Long id) {
        User user = userMapper.selectById(id);
        if (user == null) {
            throw new NotFoundException("用户不存在");
        }
        return UserVO.fromEntity(user);
    }

    @Transactional(rollbackFor = Exception.class)
    public Long create(UserCreateRequest request) {
        User user = User.builder()
            .username(request.getUsername())
            .password(passwordEncoder.encode(request.getPassword()))
            .build();
        userMapper.insert(user);
        return user.getId();
    }
}
```

### Controller 层模式

```java
@RestController
@RequestMapping("/api/users")
@RequiredArgsConstructor
@Slf4j
@Tag(name = "用户管理", description = "用户相关接口")
public class UserController {

    private final UserService userService;

    @GetMapping("/{id}")
    @Operation(summary = "根据ID查询用户")
    public Result<UserVO> getById(@PathVariable Long id) {
        return Result.success(userService.getById(id));
    }

    @PostMapping
    @Operation(summary = "创建用户")
    public Result<Long> create(@Valid @RequestBody UserCreateRequest request) {
        return Result.success(userService.create(request));
    }
}
```

### JWT 安全配置

```java
@Configuration
@EnableMethodSecurity(prePostEnabled = true)
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        return http
            .csrf(AbstractHttpConfigurer::disable)
            .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(a -> a
                .requestMatchers("/api/auth/**").permitAll()
                .anyRequest().authenticated())
            .addFilterBefore(jwtAuthenticationFilter, UsernamePasswordAuthenticationFilter.class)
            .build();
    }
}
```

## 环境变量配置

```bash
# application-prod.yml
spring:
  profiles:
    active: prod
  datasource:
    url: ${DB_URL}
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
  data:
    redis:
      host: ${REDIS_HOST}
      port: ${REDIS_PORT}
      password: ${REDIS_PASSWORD}

jwt:
  secret: ${JWT_SECRET}
  expiration: ${JWT_EXPIRATION}
```

## 常用命令

```bash
# Maven 构建
mvn clean compile
mvn clean package
mvn clean test

# Docker
docker-compose up -d
docker-compose logs -f app
docker-compose exec app bash

# 测试
mvn test                           # 运行所有测试
mvn test -Dtest=UserServiceTest    # 运行单个测试类
mvn verify                         # 包含集成测试
```

## Lombok 常用注解

```java
@Data                 // Getter + Setter + toString + equals + hashCode
@Builder              // 构建者模式
@RequiredArgsConstructor  // 基于final字段的构造函数
@AllArgsConstructor   // 全参构造函数
@NoArgsConstructor    // 无参构造函数
@Slf4j                // 日志对象 (log)
@Value                // 不可变类
@With                 // with方法（不可变对象修改）
```

## MyBatis-Plus 最佳实践

```java
// Mapper 接口
@Mapper
public interface UserMapper extends BaseMapper<User> {
    // BaseMapper 提供 CRUD：insert, selectById, updateById, deleteById 等
}

// 自定义 SQL（XML 方式）
@Mapper
public interface OrderMapper extends BaseMapper<Order> {
    List<OrderVO> selectOrderWithUser(@Param("userId") Long userId);
}
```

## Redis 使用模式

```java
@Service
@RequiredArgsConstructor
public class CacheService {

    private final RedisTemplate<String, Object> redisTemplate;

    public void set(String key, Object value, long timeout, TimeUnit unit) {
        redisTemplate.opsForValue().set(key, value, timeout, unit);
    }

    public <T> T get(String key, Class<T> clazz) {
        Object value = redisTemplate.opsForValue().get(key);
        return value != null ? (T) value : null;
    }

    public void delete(String key) {
        redisTemplate.delete(key);
    }
}
```

## Git 工作流

- 提交规范：`feat:` 新功能、`fix:` 修复、`refactor:` 重构、`docs:` 文档、`test:` 测试、`chore:` 构建
- 禁止直接提交到 main 分支
- 代码提交前必须通过 Code Review
- 合并前所有测试必须通过
- 使用 Maven Checkstyle 插件检查代码质量
- 使用 SonarQube 进行代码静态分析
