# 代码风格

> **适用范围**：以下风格指南适用于 **Java / Spring Boot / MyBatis-Plus** 项目

## 不可变性

**核心原则**：优先创建新对象，避免不必要的可变状态。

```java
// 推荐：DTO/VO 使用 Record（Java 21+）
public record User(String id, String name, String email) {}

// 推荐：DTO 使用 Lombok 不可变注解
@Value  // @Value = @Getter + @FieldDefaults(makeFinal = true, level = PRIVATE)
public class UserRespVO {
    String username;
    String email;
}

// Entity 可以保持可变（符合 JPA 规范）
@Data  // Entity 需要可变 setter
@Entity
public class User {
    private Long id;
    private String name;
    private String email;
}

// Service 层：避免修改传入的 DTO，返回新对象
public UserRespVO getUserById(Long id) {
    User user = userMapper.selectById(id);
    return UserConverter.toVO(user);  // 转换为新的 VO 对象
}
```

**原则**：
- DTO/VO 对象应优先考虑不可变性（使用 Record 或 @Value）
- Entity 对象可以保持可变（符合 JPA/MyBatis 规范）
- 跨层传递时返回新对象，避免修改输入参数
- 使用转换器（Converter）在不同层之间创建新对象

## 分层架构

遵循标准三层架构：

```
┌─────────────────────────────────────┐
│         Controller 层               │  入口、路由、参数校验
├─────────────────────────────────────┤
│          Service 层                 │  业务逻辑、事务控制
├─────────────────────────────────────┤
│         Mapper 层                   │  数据访问（MyBatis-Plus）
└─────────────────────────────────────┘
```

**包结构规范**（按领域组织）：

```
com.company.module
├── controller    // 控制器
├── service       // 业务逻辑
│   └── impl      // 实现类
├── mapper        // MyBatis Mapper
├── entity        // 数据库实体
├── dto           // 数据传输对象
│   ├── req       // 请求 DTO
│   └── resp      // 响应 DTO
└── vo            // 视图对象
```

## 文件组织

多小文件 > 少大文件：
- 高内聚、低耦合
- 单个类通常 150-300 行，最多 500 行
- 单个方法不超过 50 行
- 按功能/领域组织，而非按类型

## 命名规范

```java
// 类名：大驼峰
public class UserService {}

// 方法名：小驼峰
public User getUserById(Long id) {}

// 常量：全大写下划线
public static final int MAX_RETRY_COUNT = 3;

// 包名：全小写点分隔
package com.company.service.user;
```

## 依赖注入

**强制使用构造器注入**（禁止字段注入）：

```java
// 推荐：构造器注入 + Lombok @RequiredArgsConstructor
@RequiredArgsConstructor
@Service
public class UserService {
    private final UserMapper userMapper;
    private final EmailService emailService;
}

// 推荐：不使用 Lombok 时显式声明构造器
@Service
public class UserService {
    private final UserMapper userMapper;
    private final EmailService emailService;

    public UserService(UserMapper userMapper, EmailService emailService) {
        this.userMapper = userMapper;
        this.emailService = emailService;
    }
}

// 推荐：需要可选依赖时使用 @Autowired（仅限构造器）
@Service
public class UserService {
    private final UserMapper userMapper;
    private final EmailService emailService;

    @Autowired
    public UserService(UserMapper userMapper,
                       @Autowired(required = false) EmailService emailService) {
        this.userMapper = userMapper;
        this.emailService = emailService;
    }
}
```

**为什么构造器注入优于字段注入：**

| 特性 | 构造器注入 | 字段注入 |
|------|-----------|----------|
| 不可变性 | ✓ final 字段 | ✗ 可变 |
| 单元测试友好 | ✓ 直接传入构造参数 | ✗ 需要 Spring 容器 |
| 依赖明确 | ✓ 构造器签名可见 | ✗ 隐藏在字段中 |
| 循环依赖检测 | ✓ 启动时失败 | ✗ 运行时失败 |
| Spring 官方推荐 | ✓ | ✗ |

## 异常处理

### 核心原则

**异常分类处理：**

| 异常类型 | 处理策略 | 示例 |
|----------|----------|------|
| **业务异常** | 抛出 BusinessException，自定义错误码 | 用户不存在、余额不足 |
| **参数异常** | 使用 @Valid 自动校验，抛出 MethodArgumentNotValidException | 必填字段为空 |
| **系统异常** | 捕获并记录日志，返回通用错误码 | 数据库连接失败 |
| **第三方异常** | 降级处理，返回友好提示 | 支付网关超时 |

**禁止事项：**
- 禁止吞没异常（空 catch 块）
- 禁止使用 `printStackTrace()` 记录异常
- 禁止在循环中抛出异常（影响性能）

### Spring 全局异常处理

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(BusinessException.class)
    public Result<Void> handleBusinessException(BusinessException e) {
        log.error("业务异常: {}", e.getMessage());
        return Result.fail(e.getCode(), e.getMessage());
    }

    @ExceptionHandler(Exception.class)
    public Result<Void> handleException(Exception e) {
        log.error("系统异常", e);
        return Result.fail(ErrorCode.SYSTEM_ERROR);
    }
}
```

## 参数校验

使用 Jakarta Validation：

```java
@Data
@Validated
public class UserCreateReq {

    @NotBlank(message = "用户名不能为空")
    @Size(min = 2, max = 20, message = "用户名长度2-20个字符")
    private String username;

    @NotBlank(message = "邮箱不能为空")
    @Email(message = "邮箱格式不正确")
    private String email;

    @Min(value = 0, message = "年龄不能小于0")
    @Max(value = 150, message = "年龄不能大于150")
    private Integer age;
}

// Controller 使用
@PostMapping("/users")
public Result<Void> create(@Valid @RequestBody UserCreateReq req) {
    userService.create(req);
    return Result.ok();
}
```

## 日志规范

```java
@Slf4j
@Service
public class UserService {

    public void createUser(UserCreateReq req) {
        log.info("创建用户: username={}", req.getUsername());
        // 业务逻辑
        log.debug("用户创建成功: id={}", user.getId());
    }

    public void deleteUser(Long id) {
        log.warn("删除用户: id={}, operator={}", id, SecurityUtils.getUserId());
        // 删除逻辑
    }
}
```

**日志级别使用指南：**

| 级别 | 使用场景 |
|------|----------|
| **ERROR** | 系统错误、需要立即处理的问题 |
| **WARN** | 业务异常、潜在问题 |
| **INFO** | 关键业务流程、状态变更 |
| **DEBUG** | 调试信息、详细执行流程 |

## Lombok 注解推荐

```java
// Service 层
@RequiredArgsConstructor  // 生成构造器（推荐）
@Slf4j                    // 生成日志对象
@Service

// Entity/DTO
@Data                     // getter/setter/toString/equals/hashCode
@Builder                  // 构建器模式
@NoArgsConstructor        // 无参构造器（JSON 反序列化需要）
@AllArgsConstructor       // 全参构造器

// Controller
@RequiredArgsConstructor
@RestController

// 示例组合
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class UserRespVO {
    private Long id;
    private String username;
}
```

## Controller 规范

```java
@RestController
@RequestMapping("/api/users")
@RequiredArgsConstructor
@Tag(name = "用户管理", description = "用户相关接口")
public class UserController {

    private final UserService userService;

    @PostMapping
    @Operation(summary = "创建用户")
    public Result<Void> create(@Valid @RequestBody UserCreateReq req) {
        userService.create(req);
        return Result.ok();
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

## Service 规范

```java
public interface UserService extends IService<User> {
    UserRespVO getById(Long id);
    PageResult<UserRespVO> page(PageReq pageReq);
    Long create(UserCreateReq req);
}

@Slf4j
@Service
@RequiredArgsConstructor
public class UserServiceImpl extends ServiceImpl<UserMapper, User> implements UserService {

    private final UserMapper userMapper;
    private final UserConverter userConverter;

    @Override
    @Transactional(rollbackFor = Exception.class)
    public Long create(UserCreateReq req) {
        // 参数校验
        validateUser(req);

        // 业务逻辑
        User user = buildUser(req);
        userMapper.insert(user);

        // 后置处理
        sendWelcomeEmail(user);

        return user.getId();
    }

    private void validateUser(UserCreateReq req) {
        if (userMapper.existsByUsername(req.getUsername())) {
            throw new BusinessException(ErrorCode.USER_ALREADY_EXISTS);
        }
    }

    private User buildUser(UserCreateReq req) {
        return User.builder()
                .username(req.getUsername())
                .email(req.getEmail())
                .build();
    }

    private void sendWelcomeEmail(User user) {
        // 异步发送欢迎邮件
    }
}
```

## 代码质量检查清单

在标记工作完成前：

- [ ] 代码可读且命名规范（驼峰、常量大写下划线）
- [ ] 方法简短（<50 行）
- [ ] 类职责单一（<500 行）
- [ ] 没有深层嵌套（>4 层，使用卫语句提前返回）
- [ ] 统一的异常处理
- [ ] 没有调试用的 System.out.println（使用日志）
- [ ] 没有硬编码的配置（使用配置文件或常量类）
- [ ] DTO/VO 使用不可变对象（Record 或 @Value）
- [ ] 跨层传递时返回新对象，不修改输入参数
- [ ] 使用构造器注入而非字段注入
- [ ] 依赖声明为 final
- [ ] 事务方法标注 @Transactional
- [ ] 敏感日志已脱敏
