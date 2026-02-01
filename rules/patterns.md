---
name: common-patterns
description: Java/Spring Boot 常见设计模式和代码模板
priority: should
tags: [java, spring-boot, patterns, dto, vo]
---

# 常见模式

> **适用范围**：以下模式适用于 **Java / Spring Boot / MyBatis-Plus** 项目

## 统一响应格式

```java
@Data
@Builder
public class Result<T> {
    private Integer code;
    private String message;
    private T data;
    private Long timestamp;

    public static <T> Result<T> ok(T data) {
        return Result.<T>builder().code(200).message("success")
            .data(data).timestamp(System.currentTimeMillis()).build();
    }

    public static <T> Result<T> fail(ErrorCode errorCode) {
        return Result.<T>builder().code(errorCode.getCode())
            .message(errorCode.getMessage()).timestamp(System.currentTimeMillis()).build();
    }
}

@Data
@Builder
public class PageResult<T> {
    private List<T> records;
    private Long total;
    private Long current;
    private Long size;

    public static <T> PageResult<T> of(IPage<T> page) {
        return PageResult.<T>builder().records(page.getRecords())
            .total(page.getTotal()).current(page.getCurrent())
            .size(page.getSize()).pages(page.getPages()).build();
    }
}
```

## 错误码模式

```java
@Getter
@AllArgsConstructor
public enum ErrorCode {
    SYSTEM_ERROR(1000, "系统繁忙"),
    PARAM_ERROR(1001, "参数错误"),
    USER_NOT_FOUND(2001, "用户不存在"),
    USER_ALREADY_EXISTS(2002, "用户已存在");

    private final Integer code;
    private final String message;
}

@Data
@EqualsAndHashCode(callSuper = true)
public class BusinessException extends RuntimeException {
    private final Integer code;

    public BusinessException(ErrorCode errorCode) {
        super(errorCode.getMessage());
        this.code = errorCode.getCode();
    }
}
```

## DTO/VO 分层

```java
// 请求 DTO
@Data
public class UserCreateReq {
    @NotBlank private String username;
    @NotBlank @Email private String email;
    @Min(0) @Max(150) private Integer age;
}

// 响应 VO
@Data
@Builder
public class UserRespVO {
    private Long id;
    private String username;
    private String email;
    private Integer age;
}

// 更新请求 DTO
@Data
public class UserUpdateReq {
    private Long id;
    @NotBlank private String username;
}

// 分页请求 DTO
@Data
public class PageReq {
    @Min(1) private Long current = 1L;
    @Min(1) @Max(100) private Long size = 10L;
    private String orderBy;
    private Boolean isAsc = true;
}
```

## 对象转换模式

```java
@Component
public class UserConverter {
    public UserRespVO toVO(User user) {
        if (user == null) return null;
        return UserRespVO.builder()
            .id(user.getId()).username(user.getUsername())
            .email(user.getEmail()).build();
    }

    public User toEntity(UserCreateReq req) {
        if (req == null) return null;
        return User.builder()
            .username(req.getUsername()).email(req.getEmail()).build();
    }

    public List<UserRespVO> toVOList(List<User> users) {
        if (users == null) return Collections.emptyList();
        return users.stream().map(this::toVO)
            .filter(Objects::nonNull).collect(Collectors.toList());
    }
}
```

## Mapper 模式（MyBatis-Plus）

```java
@Data
@TableName("t_user")
@Builder
public class User {
    @TableId(type = IdType.AUTO) private Long id;
    private String username;
    private String email;

    @TableField(fill = FieldFill.INSERT) private LocalDateTime createTime;
    @TableField(fill = FieldFill.INSERT_UPDATE) private LocalDateTime updateTime;
    @TableLogic private Integer deleted;
    @Version private Integer version;
}

public interface UserMapper extends BaseMapper<User> {
    @Select("SELECT * FROM t_user WHERE username = #{username} AND deleted = 0")
    User findByUsername(@Param("username") String username);
}

@Service
@RequiredArgsConstructor
public class UserServiceImpl extends ServiceImpl<UserMapper, User> implements UserService {
    private final UserMapper userMapper;
    private final UserConverter userConverter;

    @Override
    @Transactional(rollbackFor = Exception.class)
    public Long create(UserCreateReq req) {
        User user = userConverter.toEntity(req);
        userMapper.insert(user);
        return user.getId();
    }
}
```

## Controller 模式（RESTful）

```java
@RestController
@RequestMapping("/api/users")
@RequiredArgsConstructor
@Tag(name = "用户管理")
public class UserController {
    private final UserService userService;

    @PostMapping
    @Operation(summary = "创建用户")
    public Result<Long> create(@Valid @RequestBody UserCreateReq req) {
        return Result.ok(userService.create(req));
    }

    @GetMapping("/{id}")
    @Operation(summary = "根据ID查询用户")
    public Result<UserRespVO> getById(@PathVariable Long id) {
        return Result.ok(userService.getById(id));
    }

    @GetMapping
    @Operation(summary = "分页查询用户")
    public Result<PageResult<UserRespVO>> page(PageReq pageReq) {
        return Result.ok(userService.page(pageReq));
    }
}
```

## 配置管理模式

```java
@Data
@Component
@ConfigurationProperties(prefix = "app")
public class AppProperties {
    private String name;
    private Jwt jwt = new Jwt();

    @Data
    public static class Jwt {
        private String secret;
        private Long expiration = 604800L;
    }
}
```

## 异步处理模式

```java
@Configuration
@EnableAsync
public class AsyncConfig {
    @Bean(name = "taskExecutor")
    public Executor taskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(5);
        executor.setMaxPoolSize(10);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("async-");
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        return executor;
    }
}

@Service
@Slf4j
public class EmailService {
    @Async("taskExecutor")
    public void sendWelcomeEmail(String to, String username) {
        try {
            // 发送邮件逻辑
            log.info("欢迎邮件发送成功: to={}", to);
        } catch (Exception e) {
            log.error("欢迎邮件发送失败: to={}", to, e);
        }
    }
}
```

## 缓存模式

```java
@Configuration
@EnableCaching
public class CacheConfig {
    @Bean
    public CacheManager cacheManager(RedisConnectionFactory factory) {
        RedisCacheConfiguration config = RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofMinutes(30))
            .serializeKeysWith(RedisSerializationContext.SerializationPair
                .fromSerializer(new StringRedisSerializer()))
            .serializeValuesWith(RedisSerializationContext.SerializationPair
                .fromSerializer(new GenericJackson2JsonRedisSerializer()));
        return RedisCacheManager.builder(factory).cacheDefaults(config).build();
    }
}

@Service
@RequiredArgsConstructor
public class UserServiceImpl extends ServiceImpl<UserMapper, User> implements UserService {
    @Cacheable(value = "user", key = "#id", unless = "#result == null")
    public UserRespVO getById(Long id) { /* ... */ }

    @CacheEvict(value = "user", key = "#result.id")
    public Long create(UserCreateReq req) { /* ... */ }

    @CacheEvict(value = "user", key = "#req.id")
    public void update(UserUpdateReq req) { /* ... */ }
}
```

## 脚手架项目选择

> **快速参考**
> - 小型项目: Spring Initializr
> - 中型项目: RuoYi-Vue / jeecg-boot
> - 大型/微服务: pig

### 推荐脚手架

| 项目 | 技术栈 | 特点 | 适用场景 |
|------|--------|------|----------|
| **Spring Initializr** | Spring Boot + 任意 | 官方脚手架，灵活配置 | 新项目启动、学习原型 |
| **RuoYi-Vue** | Spring Boot + Vue3 + MyBatis-Plus | 前后端分离、权限完善 | 企业后台系统、RBAC权限 |
| **pig** | Spring Cloud + Vue | 微服务架构、OAuth2 | 分布式系统、SaaS平台 |
| **jeecg-boot** | Spring Boot + Ant Design Vue | 低代码平台、代码生成 | 快速开发、CRUD密集型 |

### 脚手架选择决策树

```
开始
  ├─ 需要微服务架构？
  │   ├─ 是 → pig 或 Spring Cloud Alibaba
  │   └─ 否 → 继续
  ├─ 需要快速开发大量 CRUD？
  │   ├─ 是 → jeecg-boot
  │   └─ 否 → 继续
  ├─ 需要完善的权限管理？
  │   ├─ 是 → RuoYi-Vue
  │   └─ 否 → 继续
  └─ 从零开始学习/验证？
      ├─ 是 → Spring Initializr
      └─ 否 → 检查现有代码库是否有类似功能可复用
```

### 脚手架评估维度

| 维度 | 权重 | 检查方法 |
|------|------|----------|
| **相关性** | 40% | 检查 pom.xml 依赖、包结构、业务模块 |
| **安全性** | 25% | 运行 `mvn dependency:tree`、检查 Spring Security 配置 |
| **可扩展性** | 20% | 检查分层结构、模块划分、配置灵活性 |
| **可维护性** | 15% | 检查 README、测试目录、代码注释 |

### 复制与迭代流程

```bash
# 1. 复制脚手架结构
git clone <脚手架仓库> my-project && cd my-project

# 2. 清理不必要代码
rm -rf .git README.md docs/

# 3. 保留核心结构
# 保留：src/main/java 的分层结构
# 保留：pom.xml 的核心依赖
# 保留：application.yml 的基础配置

# 4. 自定义包名
find . -name "*.java" -exec sed -i 's/old.package/new.package/g' {} \;
```

## 代码质量检查清单

在标记工作完成前：

- [ ] 使用统一的 Result/PageResult 响应格式
- [ ] DTO/VO 分层清晰，职责单一
- [ ] 使用 Converter 进行对象转换
- [ ] Mapper 继承 BaseMapper，使用 LambdaQueryWrapper
- [ ] Controller 使用 RESTful 风格，添加 Swagger 注解
- [ ] 配置使用 @ConfigurationProperties
- [ ] 异步方法使用 @Async，配置独立线程池
- [ ] 缓存使用 Spring Cache 注解
