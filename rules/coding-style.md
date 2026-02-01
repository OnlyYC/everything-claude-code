# 代码风格

## 不可变性（关键）

始终创建新对象，绝不修改：

```java
// 错误：修改现有对象
public void updateUser(User user, String name) {
    user.setName(name);  // 修改！
}

// 正确：不可变性
public User updateUser(User user, String name) {
    return user.toBuilder()
            .name(name)
            .build();
}

// 推荐：使用 Record（Java 21+）
public record User(String id, String name, String email) {}
```

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

## 异常处理

使用 Spring 全局异常处理：

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

## 代码质量检查清单

在标记工作完成前：
- [ ] 代码可读且命名规范（驼峰、常量大写下划线）
- [ ] 方法简短（<50 行）
- [ ] 类职责单一（<500 行）
- [ ] 没有深层嵌套（>4 层，使用卫语句提前返回）
- [ ] 统一的异常处理
- [ ] 没有调试用的 System.out.println
- [ ] 没有硬编码的配置（使用配置文件或常量类）
- [ ] 对象不可变（使用 Record、Builder、Lombok）
