---
name: java-coding-standards
description: Java 21 编码风格：命名规范、不可变性、Optional 使用、Stream 流、异常处理、泛型、项目结构。适配 Spring Boot 3 + Java 21 技术栈。
version: 1.1.0
tech_stack: [Java 21, Spring Boot 3, MyBatis-Plus]
tools: Read, Write, Edit, Bash, Grep, Glob
related_skills:
  java-patterns: "本 skill 负责 Java 21 编码风格与语法；java-patterns 负责 Alibaba 强制性规则"
  backend-patterns: "通用后端架构模式"
---

# Java 21 编码风格

Spring Boot 服务中可读、可维护的 Java 21+ 代码风格指南。

## 技能职责划分

| 技能 | 职责 | 内容范围 |
|------|------|---------|
| `java-coding-standards` | **编码风格** | Java 21 语法特性、命名约定、代码格式 |
| `java-patterns` | **强制规则** | Alibaba 手册的强制规约、安全规范 |

## 核心原则

- **清晰优于聪明** - 代码被阅读的次数远多于被撰写的次数
- **默认不可变** - 最小化共享可变状态
- **快速失败** - 抛出有意义的异常
- **保持一致** - 命名和包结构统一

## 命名规范

### 类命名（大驼峰 PascalCase）

```java
// 服务实现类 - 以 Service 结尾
public class UserService {}
public class OrderService {}

// 控制器 - 以 Controller 结尾
public class UserController {}
public class OrderController {}

// 实体类 - 以 Entity 结尾
public class UserEntity {}
public class OrderEntity {}

// DTO/VO - 清晰标注用途
public class CreateUserDTO {}
public class UserVO {}
public class UserQuery {}

// 异常类 - 以 Exception 结尾
public class BusinessException {}
public class UserNotFoundException {}

// 工具类 - 以 Util/Helper 结尾
public class DateUtil {}
public class JsonHelper {}
```

### 方法命名（小驼峰 camelCase）

```java
// 查询方法 - 以 get/find/query/list 开头
public User getUserById(Long id) {}
public User findByName(String name) {}
public List<User> listActiveUsers() {}
public Page<User> queryUsers(UserQuery query) {}

// 创建/保存 - 以 create/save 开头
public User createUser(CreateUserDTO dto) {}
public void saveUser(User user) {}

// 更新 - 以 update 开头
public User updateUser(UpdateUserDTO dto) {}

// 删除 - 以 delete/remove 开头
public void deleteUser(Long id) {}
public void removeUser(Long id) {}

// 判断 - 以 is/has/can/should 开头
public boolean isValid() {}
public boolean hasPermission(String permission) {}
public boolean canDelete(Long userId) {}

// 转换 - 以 to/convert/parse 开头
public UserVO toVO(User user) {}
public UserDTO convertToDTO(User user) {}
public LocalDate parseDate(String dateStr) {}
```

### 变量命名

```java
// 布尔变量 - 以 is/has/can/should 开头
private boolean enabled;
private boolean hasPermission;
private boolean canDelete;
private boolean shouldRefresh;

// 集合变量 - 使用复数形式
private List<User> users;
private Map<Long, User> userMap;
private Set<String> permissions;

// 常量 - 全大写下划线分隔
private static final int MAX_PAGE_SIZE = 100;
private static final String DEFAULT_CHARSET = "UTF-8";
private static final long CACHE_TTL_SECONDS = 300;
```

## 不可变性

### 使用记录类（Record）

```java
// 正确 - 使用 record 创建不可变数据载体
public record UserDTO(
    Long id,
    String name,
    String email,
    LocalDateTime createdAt
) {}

// record 自动实现：
// - 全部字段 private final
// - 自动生成构造器、getter、equals、hashCode、toString
```

### 使用 final 字段

```java
// 正确 - 类字段使用 final
@Service
@RequiredArgsConstructor
public class UserService {
    private final UserRepository userRepository;
    private final EmailService emailService;
    // 依赖不可变
}

// 正确 - 值对象使用 final
public final class Money {
    private final BigDecimal amount;
    private final Currency currency;

    public Money(BigDecimal amount, Currency currency) {
        this.amount = amount;
        this.currency = currency;
    }

    // 只提供 getter，不提供 setter
}
```

### 避免可变集合暴露

```java
// 错误 - 直接暴露可变集合
public class Order {
    private List<OrderItem> items = new ArrayList<>();

    public List<OrderItem> getItems() {
        return items;  // 调用者可以修改
    }
}

// 正确 - 返回不可变视图或副本
public class Order {
    private List<OrderItem> items = new ArrayList<>();

    public List<OrderItem> getItems() {
        return Collections.unmodifiableList(items);
    }

    // 或返回副本
    public List<OrderItem> getItemsCopy() {
        return new ArrayList<>(items);
    }
}

// 正确 - 使用 of() 创建不可变集合
private static final List<String> ALLOWED_STATUSES = List.of(
    "PENDING", "PROCESSING", "COMPLETED"
);

private static final Map<String, String> STATUS_MAP = Map.of(
    "PENDING", "待处理",
    "COMPLETED", "已完成"
);
```

## Optional 使用规范

### 返回 Optional

```java
// 正确 - 查询方法返回 Optional
public Optional<User> findById(Long id) {
    return userRepository.findById(id);
}

// 错误 - 返回 null 表示不存在
public User findById(Long id) {
    return userRepository.findById(id).orElse(null);
}
```

### 链式操作

```java
// 正确 - 使用 map/flatMap/orElse
public UserVO getUserById(Long id) {
    return userRepository.findById(id)
        .map(this::toVO)
        .orElseThrow(() -> new UserNotFoundException(id));
}

// 正确 - 嵌套 Optional 使用 flatMap
public Optional<String> getUserEmail(Long id) {
    return userRepository.findById(id)
        .flatMap(user -> Optional.ofNullable(user.getEmail()));
}

// 错误 - 使用 get()
Optional<User> user = findById(1L);
if (user.isPresent()) {
    return user.get();  // 不推荐
}
```

### 判断与默认值

```java
// 正确 - orElse 提供默认值
User user = findById(1L).orElse(User.guest());

// 正确 - orElseGet 延迟计算（推荐用于复杂操作）
User user = findById(1L).orElseGet(() -> createGuestUser());

// 错误 - orElse 中有耗时操作
User user = findById(1L).orElse(fetchFromRemote());  // 总是执行

// 正确 - orElseThrow 抛出异常
User user = findById(1L).orElseThrow(() -> new UserNotFoundException(1L));

// 正确 - ifPresent 有值时执行操作
findById(1L).ifPresent(user -> sendWelcomeEmail(user));

// 正确 - ifPresentOrElse Java 9+
findById(1L).ifPresentOrElse(
    user -> sendWelcomeEmail(user),
    () -> log.warn("User not found")
);
```

### 避免的用法

```java
// 错误 - Optional 作为字段
private Optional<String> name;  // 不要这样做

// 错误 - Optional 作为方法参数
public void setName(Optional<String> name) {}  // 不要这样做

// 正确 - 直接使用 nullable 字段
private String name;

// 正确 - 直接使用参数或重载
public void setName(String name) {}
public void setName() { this.name = "default"; }
```

## Stream 流最佳实践

### 简单转换

```java
// 正确 - 简单的映射
List<String> names = users.stream()
    .map(User::getName)
    .toList();

// 正确 - 过滤
List<User> activeUsers = users.stream()
    .filter(user -> user.getStatus() == UserStatus.ACTIVE)
    .toList();

// 正确 - 过滤 + 映射
List<String> activeNames = users.stream()
    .filter(user -> user.getStatus() == UserStatus.ACTIVE)
    .map(User::getName)
    .toList();
```

### 避免复杂嵌套

```java
// 错误 - 复杂的嵌套 Stream
List<String> result = orders.stream()
    .filter(order -> order.getStatus() == OrderStatus.COMPLETED)
    .flatMap(order -> order.getItems().stream())
    .filter(item -> item.getProduct().getCategory() == "ELECTRONICS")
    .map(item -> item.getProduct().getName())
    .distinct()
    .toList();

// 正确 - 使用循环提高可读性
Set<String> productNames = new HashSet<>();
for (Order order : orders) {
    if (order.getStatus() != OrderStatus.COMPLETED) continue;
    for (OrderItem item : order.getItems()) {
        if (item.getProduct().getCategory().equals("ELECTRONICS")) {
            productNames.add(item.getProduct().getName());
        }
    }
}
```

### 收集器使用

```java
// 转换为 List
List<User> list = users.stream().toList();

// 转换为 Set
Set<String> uniqueNames = users.stream()
    .map(User::getName)
    .collect(Collectors.toSet());

// 分组
Map<UserStatus, List<User>> byStatus = users.stream()
    .collect(Collectors.groupingBy(User::getStatus));

// 分组 + 统计
Map<UserStatus, Long> countByStatus = users.stream()
    .collect(Collectors.groupingBy(User::getStatus, Collectors.counting()));

// 转换为 Map（处理重复键）
Map<Long, User> userMap = users.stream()
    .collect(Collectors.toMap(
        User::getId,
        Function.identity(),
        (existing, replacement) -> existing  // 保留已存在的
    ));

// 分区（分为两组）
Map<Boolean, List<User>> partitioned = users.stream()
    .collect(Collectors.partitioningBy(u -> u.getAge() >= 18));
```

### 短路操作

```java
// anyMatch - 任意匹配
boolean hasActiveUser = users.stream()
    .anyMatch(u -> u.getStatus() == UserStatus.ACTIVE);

// allMatch - 全部匹配
boolean allAdults = users.stream()
    .allMatch(u -> u.getAge() >= 18);

// noneMatch - 全部不匹配
boolean noneDeleted = users.stream()
    .noneMatch(u -> u.isDeleted());

// findFirst - 找到第一个
Optional<User> firstActive = users.stream()
    .filter(u -> u.getStatus() == UserStatus.ACTIVE)
    .findFirst();

// findAny - 找到任意一个（并行时更快）
Optional<User> anyActive = users.stream()
    .filter(u -> u.getStatus() == UserStatus.ACTIVE)
    .findAny();
```

## 异常处理规范

### 异常类型选择

```java
// 领域错误使用非受检异常
public class UserNotFoundException extends RuntimeException {
    public UserNotFoundException(Long id) {
        super("User not found: " + id);
    }
}

public class InsufficientStockException extends RuntimeException {
    public InsufficientStockException(Long productId, int requested, int available) {
        super(String.format("Insufficient stock for product %d: requested=%d, available=%d",
            productId, requested, available));
    }
}

// 系统错误用上下文包装
try {
    externalApi.call();
} catch (Exception e) {
    throw new SystemException("External API call failed", e);
}
```

### 异常抛出

```java
// 正确 - 快速失败
public void withdraw(Long accountId, BigDecimal amount) {
    if (amount.compareTo(BigDecimal.ZERO) <= 0) {
        throw new IllegalArgumentException("Amount must be positive");
    }
    Account account = findById(accountId)
        .orElseThrow(() -> new AccountNotFoundException(accountId));
    if (account.getBalance().compareTo(amount) < 0) {
        throw new InsufficientBalanceException(accountId, amount, account.getBalance());
    }
    // 业务逻辑
}

// 错误 - 吞掉异常
try {
    doSomething();
} catch (Exception e) {
    // 静默忽略
}
```

### 异常捕获

```java
// 正确 - 捕获具体异常
try {
    processPayment(order);
} catch (PaymentException e) {
    log.error("Payment failed for order {}", order.getId(), e);
    throw new OrderCreationException("Payment failed", e);
}

// 错误 - 捕获过于宽泛
try {
    doSomething();
} catch (Exception e) {  // 不要这样做
    log.error("Error", e);
}

// 正确 - 集中重新抛出/记录
@ExceptionHandler
public Result<Void> handleException(Exception ex) {
    log.error("Unhandled exception", ex);
    return Result.fail(ErrorCode.SYSTEM_ERROR, "系统繁忙");
}
```

### Try-with-resources

```java
// 正确 - 自动关闭资源
try (InputStream is = new FileInputStream(file);
     BufferedReader reader = new BufferedReader(new InputStreamReader(is))) {
    // 使用资源
}

// 正确 - 多个资源
try (Connection conn = dataSource.getConnection();
     PreparedStatement stmt = conn.prepareStatement(sql)) {
    // 使用资源
}
```

## 泛型和类型安全

### 避免原始类型

```java
// 错误 - 使用原始类型
List list = new ArrayList();
list.add("string");
list.add(123);  // 编译通过但运行时可能出错

// 正确 - 声明泛型参数
List<String> list = new ArrayList<>();
// list.add(123);  // 编译错误
```

### 泛型方法

```java
// 正确 - 泛型方法
public <T> List<T> filter(List<T> list, Predicate<T> predicate) {
    return list.stream()
        .filter(predicate)
        .toList();
}

// 正确 - 有界泛型
public <T extends Comparable<T>> T max(List<T> list) {
    return list.stream().max(Comparable::compareTo).orElseThrow();
}

// 正确 - 多个泛型参数
public <K, V> Map<K, V> toMap(List<V> list, Function<V, K> keyExtractor) {
    return list.stream()
        .collect(Collectors.toMap(keyExtractor, Function.identity()));
}
```

### 通配符使用

```java
// PECS: Producer Extends, Consumer Super

// 生产者 - 读取数据
public void processAll(List<? extends Number> numbers) {
    for (Number n : numbers) {
        System.out.println(n.doubleValue());
    }
    // numbers.add(1);  // 编译错误 - 不能写入
}

// 消费者 - 写入数据
public void addAll(List<? super Integer> list) {
    list.add(1);
    list.add(2);
    // Integer i = list.get(0);  // 编译错误 - 读取类型不明确
}
```

## 代码格式规范

### 缩进与对齐

```java
// 正确 - 4 个空格缩进
public void method() {
    if (condition) {
        doSomething();
    }
}

// 方法参数对齐
public void method(
    String param1,
    String param2,
    String param3
) {
    // 方法体
}

// 链式调用对齐
List<User> users = users.stream()
    .filter(User::isActive)
    .sorted(Comparator.comparing(User::getName))
    .toList();
```

### 成员顺序

```java
public class UserService {
    // 1. 常量
    private static final int MAX_RETRY = 3;

    // 2. 静态变量
    private static final Logger log = LoggerFactory.getLogger(UserService.class);

    // 3. 实例变量（按访问修饰符：public -> protected -> private）
    private final UserRepository userRepository;
    private final EmailService emailService;

    // 4. 构造器（使用 Lombok 自动生成）
    // @RequiredArgsConstructor 生成全参构造器

    // 5. 公共方法
    public User createUser(CreateUserDTO dto) {}

    // 6. 保护方法
    protected void validateUser(User user) {}

    // 7. 私有方法
    private void sendWelcomeEmail(User user) {}

    // 8. 内部类
    private static class UserValidator {}
}
```

### 方法长度

```java
// 错误 - 方法过长（>50行）
public void processOrder(OrderDTO dto) {
    // 100 行代码...
}

// 正确 - 提取为小方法
public void processOrder(OrderDTO dto) {
    validateOrder(dto);
    User user = findOrCreateUser(dto);
    Order order = createOrder(dto, user);
    processPayment(order);
    sendConfirmation(order);
}
```

## 日志规范

### 日志级别使用

```java
// ERROR - 影响系统正常运行
log.error("Database connection failed", exception);

// WARN - 不影响运行但需要注意
log.warn("Cache miss for key: {}", key);

// INFO - 关键业务流程
log.info("Order created: orderId={}, userId={}", order.getId(), order.getUserId());

// DEBUG - 调试信息
log.debug("Query parameters: {}", params);

// TRACE - 更详细的调试信息
log.trace("Processing item: {}", item);
```

### 占位符

```java
// 正确 - 使用占位符
log.info("User created: id={}, name={}", user.getId(), user.getName());

// 错误 - 字符串拼接（性能差）
log.info("User created: id=" + user.getId() + ", name=" + user.getName());
```

### 异常日志

```java
// 正确 - 记录异常堆栈
try {
    doSomething();
} catch (Exception e) {
    log.error("Operation failed: orderId={}", orderId, e);
    throw e;
}

// 错误 - 不记录堆栈
log.error("Operation failed: {}", e.getMessage());  // 丢失堆栈信息
```

### 敏感信息保护

```java
// 错误 - 输出敏感信息
log.info("User login: username={}, password={}", username, password);

// 正确 - 脱敏处理
log.info("User login: username={}", maskUsername(username));
```

## 空值处理

### 参数校验

```java
// 使用 Bean Validation
public class CreateUserDTO {
    @NotNull(message = "姓名不能为空")
    @Size(min = 2, max = 50, message = "姓名长度2-50")
    private String name;

    @NotBlank(message = "邮箱不能为空")
    @Email(message = "邮箱格式不正确")
    private String email;
}

// 方法参数校验
@Service
@Validated
public class UserService {
    public void createUser(@Valid CreateUserDTO dto) {}
}

// 编程式校验
public void setUsername(String username) {
    if (StringUtils.isBlank(username)) {
        throw new IllegalArgumentException("Username cannot be blank");
    }
    this.username = username;
}
```

### 使用 Optional 避免 Null

```java
// 返回 Optional
public Optional<User> findById(Long id) {
    return userRepository.findById(id);
}

// 使用 Optional 链式操作
public String getUserName(Long id) {
    return findById(id)
        .map(User::getName)
        .orElse("Unknown");
}
```

## 项目结构

### 标准包结构

```
com.example.app/
├── config/              # 配置类
│   ├── SecurityConfig.java
│   └── CacheConfig.java
├── controller/          # 控制器层
│   └── UserController.java
├── service/             # 业务逻辑层
│   ├── UserService.java
│   └── impl/
│       └── UserServiceImpl.java
├── mapper/              # MyBatis Mapper 接口
│   └── UserMapper.java
├── entity/              # 数据库实体
│   └── UserEntity.java
├── domain/              # 领域模型
│   └── User.java
├── dto/                 # 数据传输对象（接收请求）
│   ├── CreateUserDTO.java
│   └── UpdateUserDTO.java
├── vo/                  # 视图对象（返回响应）
│   └── UserVO.java
├── query/               # 查询对象
│   └── UserQuery.java
├── enums/               # 枚举类
│   └── UserStatus.java
├── exception/           # 自定义异常
│   ├── BusinessException.java
│   └── UserNotFoundException.java
├── util/                # 工具类
│   └── DateUtil.java
├── common/              # 公共类
│   ├── Result.java
│   └── ErrorCode.java
└── constants/           # 常量定义
    └── CacheConstants.java
```

## 代码异味检测

| 异味 | 说明 | 正确做法 |
|------|------|---------|
| 长方法 | 方法超过 50 行 | 拆分为多个小方法 |
| 深层嵌套 | 嵌套超过 3 层 | 提前返回、提取方法 |
| 魔法数字 | 无解释的数字 | 使用命名常量 |
| 重复代码 | 相同代码出现多次 | 提取为公共方法 |
| 过长参数列表 | 参数超过 5 个 | 使用 DTO 对象 |
| 上帝类 | 类承担过多职责 | 拆分职责 |
| 特征依恋 | 方法过度依赖其他类 | 重构方法位置 |

**记住**：保持代码有意图、有类型、可观察。优先考虑可维护性而非微优化。

## 相关技能

- `java-patterns` - 基于 Alibaba Java 开发手册
- `java-testing` - JUnit 5 + Mockito 测试指南
- `springboot-patterns` - Spring Boot 架构模式
- `springboot-tdd` - Spring Boot TDD 工作流程
