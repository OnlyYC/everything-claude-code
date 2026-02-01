# 常见模式

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

    public static <T> Result<T> fail(ErrorCode errorCode) {
        return Result.<T>builder()
                .code(errorCode.getCode())
                .message(errorCode.getMessage())
                .timestamp(System.currentTimeMillis())
                .build();
    }
}

// 分页响应
@Data
@Builder
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
}

// 响应 VO
@Data
@Builder
public class UserRespVO {
    private Long id;
    private String username;
    private String email;
    private LocalDateTime createTime;
}

// 转换工具
@Component
public class UserConverter {
    public UserRespVO toVO(User user) {
        return UserRespVO.builder()
                .id(user.getId())
                .username(user.getUsername())
                .email(user.getEmail())
                .createTime(user.getCreateTime())
                .build();
    }
}
```

## Mapper 模式（MyBatis-Plus）

```java
// Entity 实体
@Data
@TableName("t_user")
public class User {
    @TableId(type = IdType.AUTO)
    private Long id;

    private String username;
    private String email;

    @TableField(fill = FieldFill.INSERT)
    private LocalDateTime createTime;

    @TableField(fill = FieldFill.INSERT_UPDATE)
    private LocalDateTime updateTime;

    @TableLogic
    private Integer deleted;
}

// Mapper 接口
public interface UserMapper extends BaseMapper<User> {
    // 继承 BaseMapper 获得 CRUD 方法
    // 自定义方法可添加 @Select 注解或 XML
}

// Service 层
public interface UserService extends IService<User> {
    UserRespVO getById(Long id);
    PageResult<UserRespVO> page(PageReq pageReq);
}

@Service
public class UserServiceImpl extends ServiceImpl<UserMapper, User> implements UserService {

    @Autowired
    private UserConverter userConverter;

    @Override
    public UserRespVO getById(Long id) {
        User user = super.getById(id);
        return userConverter.toVO(user);
    }

    @Override
    public PageResult<UserRespVO> page(PageReq pageReq) {
        IPage<User> page = new Page<>(pageReq.getCurrent(), pageReq.getSize());
        LambdaQueryWrapper<User> wrapper = new LambdaQueryWrapper<>();
        // 添加查询条件...
        IPage<User> result = super.page(page, wrapper);
        return PageResult.of(result);
    }
}
```

## Controller 模式（RESTful）

```java
@RestController
@RequestMapping("/api/users")
@RequiredArgsConstructor
public class UserController {

    private final UserService userService;

    @PostMapping
    public Result<Void> create(@Valid @RequestBody UserCreateReq req) {
        userService.create(req);
        return Result.ok();
    }

    @GetMapping("/{id}")
    public Result<UserRespVO> getById(@PathVariable Long id) {
        return Result.ok(userService.getById(id));
    }

    @GetMapping
    public Result<PageResult<UserRespVO>> page(PageReq pageReq) {
        return Result.ok(userService.page(pageReq));
    }

    @PutMapping("/{id}")
    public Result<Void> update(@PathVariable Long id, @Valid @RequestBody UserUpdateReq req) {
        userService.update(id, req);
        return Result.ok();
    }

    @DeleteMapping("/{id}")
    public Result<Void> delete(@PathVariable Long id) {
        userService.delete(id);
        return Result.ok();
    }
}
```

## 脚手架项目

实现新功能时：
1. 搜索经过实战验证的脚手架项目
2. 使用并行 agents 评估选项：
   - 安全性评估
   - 可扩展性分析
   - 相关性评分
   - 实现规划
3. 复制最佳匹配作为基础
4. 在验证过的结构上迭代
