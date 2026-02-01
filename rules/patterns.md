# 常见模式

> **适用范围**：以下模式适用于 **Java / Spring Boot / MyBatis-Plus** 项目

## 统一响应格式

```java
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class Result<T> {
    private Integer code;
    private String message;
    private T data;
    private Long timestamp;

    public static <T> Result<T> ok(T data) {
        return Result.<T>builder()
                .code(200)
                .message("success")
                .data(data)
                .timestamp(System.currentTimeMillis())
                .build();
    }

    public static <T> Result<T> ok() {
        return ok(null);
    }

    public static <T> Result<T> fail(ErrorCode errorCode) {
        return Result.<T>builder()
                .code(errorCode.getCode())
                .message(errorCode.getMessage())
                .timestamp(System.currentTimeMillis())
                .build();
    }

    public static <T> Result<T> fail(Integer code, String message) {
        return Result.<T>builder()
                .code(code)
                .message(message)
                .timestamp(System.currentTimeMillis())
                .build();
    }
}

// 分页响应
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class PageResult<T> {
    private List<T> records;
    private Long total;
    private Long current;
    private Long size;
    private Long pages;

    public static <T> PageResult<T> of(IPage<T> page) {
        return PageResult.<T>builder()
                .records(page.getRecords())
                .total(page.getTotal())
                .current(page.getCurrent())
                .size(page.getSize())
                .pages(page.getPages())
                .build();
    }

    public static <T> PageResult<T> empty() {
        return PageResult.<T>builder()
                .records(Collections.emptyList())
                .total(0L)
                .current(1L)
                .size(10L)
                .pages(0L)
                .build();
    }
}
```

## 错误码模式

```java
@Getter
@AllArgsConstructor
public enum ErrorCode {

    // 通用错误码 1xxx
    SYSTEM_ERROR(1000, "系统繁忙，请稍后重试"),
    PARAM_ERROR(1001, "参数错误"),
    NOT_FOUND(1002, "资源不存在"),

    // 用户相关 2xxx
    USER_NOT_FOUND(2001, "用户不存在"),
    USER_ALREADY_EXISTS(2002, "用户已存在"),
    USER_PASSWORD_ERROR(2003, "密码错误"),
    USER_ACCOUNT_LOCKED(2004, "账号已被锁定"),

    // 业务相关 3xxx
    ORDER_NOT_FOUND(3001, "订单不存在"),
    ORDER_STATUS_ERROR(3002, "订单状态错误"),
    ;

    private final Integer code;
    private final String message;
}

// 自定义业务异常
@Data
@EqualsAndHashCode(callSuper = true)
public class BusinessException extends RuntimeException {
    private final Integer code;

    public BusinessException(ErrorCode errorCode) {
        super(errorCode.getMessage());
        this.code = errorCode.getCode();
    }

    public BusinessException(Integer code, String message) {
        super(message);
        this.code = code;
    }
}
```

## DTO/VO 分层模式

```java
// 请求 DTO
@Data
public class UserCreateReq {
    @NotBlank
    private String username;

    @NotBlank
    @Email
    private String email;

    @Min(0)
    @Max(150)
    private Integer age;
}

// 响应 VO
@Data
@Builder
public class UserRespVO {
    private Long id;
    private String username;
    private String email;
    private Integer age;
    private LocalDateTime createTime;
}

// 更新请求 DTO
@Data
public class UserUpdateReq {
    private Long id;

    @NotBlank
    private String username;

    @NotBlank
    @Email
    private String email;
}

// 查询请求 DTO
@Data
public class UserQueryReq {
    private String username;
    private String email;
    private Integer ageMin;
    private Integer ageMax;
}

// 分页请求 DTO
@Data
public class PageReq {
    @Min(1)
    private Long current = 1L;

    @Min(1)
    @Max(100)
    private Long size = 10L;

    private String orderBy;
    private Boolean isAsc = true;
}
```

## 对象转换模式

```java
@Component
public class UserConverter {

    public UserRespVO toVO(User user) {
        if (user == null) {
            return null;
        }
        return UserRespVO.builder()
                .id(user.getId())
                .username(user.getUsername())
                .email(user.getEmail())
                .age(user.getAge())
                .createTime(user.getCreateTime())
                .build();
    }

    public User toEntity(UserCreateReq req) {
        if (req == null) {
            return null;
        }
        return User.builder()
                .username(req.getUsername())
                .email(req.getEmail())
                .age(req.getAge())
                .build();
    }

    public void updateEntity(User user, UserUpdateReq req) {
        if (user == null || req == null) {
            return;
        }
        user.setUsername(req.getUsername());
        user.setEmail(req.getEmail());
    }

    public List<UserRespVO> toVOList(List<User> users) {
        if (users == null || users.isEmpty()) {
            return Collections.emptyList();
        }
        return users.stream()
                .map(this::toVO)
                .filter(Objects::nonNull)
                .collect(Collectors.toList());
    }

    public PageResult<UserRespVO> toPageResult(IPage<User> page) {
        if (page == null) {
            return PageResult.empty();
        }
        return PageResult.<UserRespVO>builder()
                .records(toVOList(page.getRecords()))
                .total(page.getTotal())
                .current(page.getCurrent())
                .size(page.getSize())
                .pages(page.getPages())
                .build();
    }
}
```

## Mapper 模式（MyBatis-Plus）

```java
// Entity 实体
@Data
@TableName("t_user")
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class User {
    @TableId(type = IdType.AUTO)
    private Long id;

    private String username;
    private String email;
    private Integer age;

    @TableField(fill = FieldFill.INSERT)
    private LocalDateTime createTime;

    @TableField(fill = FieldFill.INSERT_UPDATE)
    private LocalDateTime updateTime;

    @TableLogic
    private Integer deleted;

    // 乐观锁
    @Version
    private Integer version;
}

// Mapper 接口
public interface UserMapper extends BaseMapper<User> {
    // 继承 BaseMapper 获得 CRUD 方法

    // 自定义方法：使用 @Select 注解
    @Select("SELECT * FROM t_user WHERE username = #{username} AND deleted = 0")
    User findByUsername(@Param("username") String username);

    // 复杂查询使用 XML
    IPage<UserRespVO> selectPageByCondition(IPage<User> page, @Param("req") UserQueryReq req);
}

// Service 层
public interface UserService extends IService<User> {
    UserRespVO getById(Long id);
    PageResult<UserRespVO> page(PageReq pageReq);
    Long create(UserCreateReq req);
    void update(UserUpdateReq req);
    void delete(Long id);
}

@Service
@RequiredArgsConstructor
public class UserServiceImpl extends ServiceImpl<UserMapper, User> implements UserService {

    private final UserMapper userMapper;
    private final UserConverter userConverter;

    @Override
    public UserRespVO getById(Long id) {
        User user = super.getById(id);
        return userConverter.toVO(user);
    }

    @Override
    public PageResult<UserRespVO> page(PageReq pageReq) {
        IPage<User> page = new Page<>(pageReq.getCurrent(), pageReq.getSize());

        LambdaQueryWrapper<User> wrapper = new LambdaQueryWrapper<>();
        // 动态查询条件
        if (StringUtils.hasText(pageReq.getOrderBy())) {
            wrapper.orderBy(true, pageReq.getIsAsc(),
                    "asc".equalsIgnoreCase(pageReq.getOrderBy()) ? User::getId : User::getCreateTime);
        }

        IPage<User> result = super.page(page, wrapper);
        return userConverter.toPageResult(result);
    }

    @Override
    @Transactional(rollbackFor = Exception.class)
    public Long create(UserCreateReq req) {
        User user = userConverter.toEntity(req);
        userMapper.insert(user);
        return user.getId();
    }

    @Override
    @Transactional(rollbackFor = Exception.class)
    public void update(UserUpdateReq req) {
        User user = userMapper.selectById(req.getId());
        if (user == null) {
            throw new BusinessException(ErrorCode.USER_NOT_FOUND);
        }
        userConverter.updateEntity(user, req);
        userMapper.updateById(user);
    }

    @Override
    @Transactional(rollbackFor = Exception.class)
    public void delete(Long id) {
        boolean success = super.removeById(id);
        if (!success) {
            throw new BusinessException(ErrorCode.USER_NOT_FOUND);
        }
    }
}
```

## Controller 模式（RESTful）

```java
@RestController
@RequestMapping("/api/users")
@RequiredArgsConstructor
@Tag(name = "用户管理", description = "用户相关接口")
public class UserController {

    private final UserService userService;

    @PostMapping
    @Operation(summary = "创建用户")
    public Result<Long> create(@Valid @RequestBody UserCreateReq req) {
        Long id = userService.create(req);
        return Result.ok(id);
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

    @PutMapping
    @Operation(summary = "更新用户")
    public Result<Void> update(@Valid @RequestBody UserUpdateReq req) {
        userService.update(req);
        return Result.ok();
    }

    @DeleteMapping("/{id}")
    @Operation(summary = "删除用户")
    public Result<Void> delete(@PathVariable Long id) {
        userService.delete(id);
        return Result.ok();
    }
}
```

## 配置管理模式

```java
// 使用 @ConfigurationProperties
@Data
@Component
@ConfigurationProperties(prefix = "app")
public class AppProperties {
    private String name;
    private String version;
    private Jwt jwt = new Jwt();
    private DataSource datasource = new DataSource();

    @Data
    public static class Jwt {
        private String secret;
        private Long expiration = 604800L; // 7 days
    }

    @Data
    public static class DataSource {
        private String url;
        private String username;
        private String password;
    }
}

// 使用方式
@Service
@RequiredArgsConstructor
public class JwtTokenService {
    private final AppProperties appProperties;

    public String generateToken(String username) {
        String secret = appProperties.getJwt().getSecret();
        Long expiration = appProperties.getJwt().getExpiration();
        // ...
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
        executor.initialize();
        return executor;
    }
}

// 使用异步方法
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

        return RedisCacheManager.builder(factory)
                .cacheDefaults(config)
                .build();
    }
}

// 使用缓存
@Service
@RequiredArgsConstructor
public class UserServiceImpl extends ServiceImpl<UserMapper, User> implements UserService {

    @Cacheable(value = "user", key = "#id", unless = "#result == null")
    public UserRespVO getById(Long id) {
        // ...
    }

    @CacheEvict(value = "user", key = "#result.id")
    public Long create(UserCreateReq req) {
        // ...
    }

    @CacheEvict(value = "user", key = "#req.id")
    public void update(UserUpdateReq req) {
        // ...
    }

    @CacheEvict(value = "user", key = "#id")
    public void delete(Long id) {
        // ...
    }
}
```

## 脚手架项目选择

> **快速参考**
> - 小型项目: Spring Initializr
> - 中型项目: RuoYi-Vue / jeecg-boot
> - 大型/微服务: pig / Spring Cloud Alibaba

实现新功能时，建议按以下流程选择脚手架项目：

### 1. 搜索现有脚手架

```bash
# 使用并行 agents 评估选项
1. Explore Agent: 在代码库中搜索类似功能实现
2. architect Agent: 评估脚手架项目的架构适配性
3. general-purpose Agent: 研究外部脚手架选项
```

### 2. 脚手架评估维度

| 维度 | 权重 | 评估要点 | 检查方法 |
|------|------|----------|----------|
| **相关性** | 40% | 功能相似度、技术栈匹配、业务领域 | 检查 pom.xml 依赖、包结构、业务模块 |
| **安全性** | 25% | 依赖无已知漏洞、安全实践、认证授权 | 运行 `mvn dependency:tree`、检查 Spring Security 配置 |
| **可扩展性** | 20% | 架构是否支持未来扩展、模块化程度 | 检查分层结构、模块划分、配置灵活性 |
| **可维护性** | 15% | 代码质量、文档完整性、测试覆盖率 | 检查 README、测试目录、代码注释 |

### 3. 脚手架评估检查清单

在使用脚手架前，确认以下事项：

#### 3.1 技术栈检查
```bash
# 检查 Spring Boot 版本
grep -E "spring-boot-starter-parent|spring.boot.version" pom.xml

# 检查核心依赖
grep -E "mybatis-plus|mysql|redis|jwt" pom.xml

# 检查 Java 版本
grep -E "java.version|maven.compiler" pom.xml
```

#### 3.2 安全配置检查
```bash
# 检查是否存在安全配置
find . -name "*Security*.java" -o -name "*Jwt*.java"

# 检查敏感信息管理
find . -name "application*.yml" -exec grep -l "password\|secret" {} \;

# 确认使用环境变量而非硬编码
grep -r "password\|secret" src/main/resources/
```

#### 3.3 架构模式检查
```bash
# 检查分层结构
ls -la src/main/java/com/example/*/controller/
ls -la src/main/java/com/example/*/service/
ls -la src/main/java/com/example/*/mapper/

# 检查是否使用本文件定义的标准模式
grep -r "Result<" src/main/java/
grep -r "PageResult<" src/main/java/
grep -r "BusinessException" src/main/java/
```

#### 3.4 代码质量检查
```bash
# 检查测试覆盖率
find . -name "*Test.java" | wc -l

# 检查文档完整性
cat README.md | grep -E "Usage|Example|API"

# 检查代码规范
grep -r "@Slf4j\|@RequiredArgsConstructor" src/main/java/
```

### 4. 推荐的 Spring Boot 脚手架

| 项目 | 技术栈 | 特点 | 适用场景 | GitHub Stars |
|------|--------|------|----------|--------------|
| **Spring Initializr** | Spring Boot + 任意 | 官方脚手架，灵活配置 | 新项目启动、学习原型 | - |
| **RuoYi-Vue** | Spring Boot + Vue3 + MyBatis-Plus | 前后端分离、权限完善 | 企业后台系统、RBAC权限 | 28k+ |
| **pig** | Spring Cloud + Vue | 微服务架构、OAuth2 | 分布式系统、SaaS平台 | 28k+ |
| **jeecg-boot** | Spring Boot + Ant Design Vue | 低代码平台、代码生成 | 快速开发、CRUD密集型 | 40k+ |

### 5. 脚手架选择决策树

```
开始
  │
  ├─ 是否需要微服务架构？
  │   ├─ 是 → 选择 pig 或 Spring Cloud Alibaba 脚手架
  │   └─ 否 → 继续评估
  │
  ├─ 是否需要快速开发大量 CRUD？
  │   ├─ 是 → 选择 jeecg-boot
  │   └─ 否 → 继续评估
  │
  ├─ 是否需要完善的权限管理？
  │   ├─ 是 → 选择 RuoYi-Vue
  │   └─ 否 → 继续评估
  │
  └─ 是否从零开始学习/验证？
      ├─ 是 → 使用 Spring Initializr
      └─ 否 → 检查现有代码库是否有类似功能可复用
```

### 6. 复制与迭代流程

```bash
# 1. 复制最佳匹配的脚手架结构
git clone <脚手架仓库> my-project
cd my-project

# 2. 清理不必要的代码
rm -rf .git README.md docs/
find . -name "*.md" -type f ! -name "README.md" -delete

# 3. 保留核心结构
# 保留：src/main/java 的分层结构
# 保留：pom.xml 的核心依赖
# 保留：application.yml 的基础配置

# 4. 自定义包名和配置
# 替换包名
find . -name "*.java" -exec sed -i 's/old.package/new.package/g' {} \;

# 5. 在验证过的结构上迭代实现
# 使用本文件定义的模式添加新功能
```

### 7. 脚手架适配建议

根据项目规模和团队情况：

#### 小型项目（1-3人，3个月内）
- 推荐：Spring Initializr + 本文件模式
- 依赖：spring-boot-starter-web, mybatis-plus, mysql
- 结构：Controller → Service → Mapper（三层即可）

#### 中型项目（3-10人，6-12个月）
- 推荐：RuoYi-Vue 或 jeecg-boot
- 依赖：增加 Redis、JWT、Spring Security
- 结构：完整分层 + 权限模块 + 代码生成

#### 大型项目（10+人，12个月+）
- 推荐：pig 或自定义微服务脚手架
- 依赖：Spring Cloud 全家桶 + 分布式组件
- 结构：微服务 + 网关 + 配置中心 + 链路追踪
