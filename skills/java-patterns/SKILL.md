---
name: java-patterns
description: Java 开发模式：基于阿里巴巴 Java 开发手册，适配 Java 21 + Spring Boot 3 + MyBatis-Plus + MySQL 技术栈。涵盖命名规范、代码格式、集合处理、并发处理、异常日志、分层架构等最佳实践。
---

# Java 开发模式

基于《阿里巴巴 Java 开发手册（嵩山版）》整理，适配 Java 21 + Spring Boot 3 + MyBatis-Plus + MySQL 技术栈。

## 核心原则

- **清晰优于聪明** - 代码被阅读的次数远多于被撰写的次数
- **简单优先** - 能用简单方案就不引入复杂依赖
- **约定优于配置** - 遵循 Spring Boot 约定和团队规范
- **快速失败** - 抛出有意义的异常，便于定位问题

## 命名规范

### 包命名

```java
// 正确 - 全部小写，域名反写
package com.alibaba.fastjson;
package org.springframework.web;
package com.example.app.controller;
```

### 类命名（大驼峰 PascalCase）

```java
// 服务实现类
public class UserServiceImpl {}
public class OrderServiceImpl {}

// 抽象类
public abstract class AbstractOrder {}
public abstract class BaseService {}

// 异常类
public class BusinessException {}
public class SystemException {}

// 测试类
public class UserServiceTest {}
public class OrderControllerTest {}

// 实体类
public class UserEntity {}
public class OrderEntity {}

// DTO/VO
public class UserDTO {}
public class UserVO {}
```

### 方法/变量命名（小驼峰 camelCase）

```java
// 正确
private String userName;
private boolean success;  // 布尔变量不以 is 开头
private Integer count;

public String getUserName() {}
public void setPassword(String password) {}
public void calculateTotal() {}
public boolean isValid() {}  // boolean 返回值可用 is 前缀
```

### 常量命名（全大写下划线分隔）

```java
// 正确
private static final int MAX_COUNT = 100;
private static final String DEFAULT_TIMEOUT = "30000";
private static final double TAX_RATE = 0.13;
```

### 禁止使用拼音

```java
// 错误
String zifu = "abc";
boolean youxiao = true;

// 正确
String character = "abc";
boolean valid = true;
```

## 代码格式

### 大括号风格

```java
// 正确 - 左大括号不换行
public void method() {
    if (condition) {
        doSomething();
    }
}
```

### 缩进与空格

```java
// 正确 - 缩进使用 4 个空格，运算符左右保留空格
int a = 1 + 2;
if (a == b) {
    doSomething();
}
```

### 方法参数对齐

```java
// 正确
public void method(String param1, String param2, String param3) {}

method("value1",
       "value2",
       "value3");
```

## OOP 规约

### 包装类比较

```java
// 错误 - 使用 == 比较
Integer a = 1;
Integer b = 1;
if (a == b) {}

// 正确 - 使用 equals
if (a.equals(b)) {}

// 或使用 Objects.equals（避免空指针）
if (Objects.equals(a, b)) {}
```

### equals 和 hashCode

```java
// 正确 - 重写 equals 必须重写 hashCode
@Data
public class User {
    private Long id;
    private String name;

    @Override
    public boolean equals(Object obj) {
        if (this == obj) return true;
        if (obj == null || getClass() != obj.getClass()) return false;
        User user = (User) obj;
        return Objects.equals(id, user.id);
    }

    @Override
    public int hashCode() {
        return Objects.hash(id);
    }
}

// 推荐：使用 Lombok @Data 自动生成
@Data
public class User {
    private Long id;
    private String name;
}
```

### 金额计算使用 BigDecimal

```java
// 错误
double amount = 0.1 + 0.2;  // 0.30000000000000004

// 正确
BigDecimal amount = new BigDecimal("0.1").add(new BigDecimal("0.2"));

// 禁止使用构造方法 BigDecimal(double)
// 错误
BigDecimal b1 = new BigDecimal(0.1);

// 正确
BigDecimal b1 = new BigDecimal("0.1");
BigDecimal b2 = BigDecimal.valueOf(0.1);
```

### long 类型赋值

```java
// 错误
long num = 1l;  // 小写 l 容易和数字 1 混淆

// 正确
long num = 1L;
```

## 集合处理

### ArrayList 的 subList

```java
// 错误 - subList 结果不可强转成 ArrayList
List<Integer> subList = list.subList(0, 5);
ArrayList<Integer> sub = (ArrayList<Integer>) subList; // ClassCastException

// 正确 - 使用 new ArrayList 包装
List<Integer> subList = new ArrayList<>(list.subList(0, 5));
```

### 数组转集合

```java
// 错误 - Arrays.asList 返回的不可修改
String[] array = {"a", "b", "c"};
List<String> list = Arrays.asList(array);
list.add("d"); // UnsupportedOperationException

// 正确 - 使用 ArrayList 包装
List<String> list = new ArrayList<>(Arrays.asList(array));
```

### foreach 中禁止 remove/add

```java
// 错误
for (String item : list) {
    if ("remove".equals(item)) {
        list.remove(item); // ConcurrentModificationException
    }
}

// 正确 - 使用迭代器
Iterator<String> iterator = list.iterator();
while (iterator.hasNext()) {
    String item = iterator.next();
    if ("remove".equals(item)) {
        iterator.remove();
    }
}

// 或使用 removeIf（推荐）
list.removeIf("remove"::equals);
```

### 泛型通配符

```java
// <? extends T> 生产者 - 不可使用 add
List<? extends Number> list1 = new ArrayList<>();
list1.add(1); // 编译错误

// <? super T> 消费者 - 不可使用 get 获取具体类型
List<? super Number> list2 = new ArrayList<>();
Number num = list2.get(0); // 编译错误

// PECS 原则：Producer Extends, Consumer Super
```

## 并发处理

### 单例模式

```java
// 正确 - 双重检查锁（DCL）
public class Singleton {
    private static volatile Singleton instance;

    private Singleton() {}

    public static Singleton getInstance() {
        if (instance == null) {
            synchronized (Singleton.class) {
                if (instance == null) {
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }
}

// 或使用枚举（推荐）
public enum Singleton {
    INSTANCE;
}
```

### SimpleDateFormat 线程安全

```java
// 错误 - SimpleDateFormat 是线程不安全的
private static final SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd");

// 正确 - 使用 DateTimeFormatter（Java 8+）
private static final DateTimeFormatter formatter =
    DateTimeFormatter.ofPattern("yyyy-MM-dd");

String dateStr = LocalDateTime.now().format(formatter);
```

### 线程池创建

```java
// 错误 - 禁止使用 Executors 创建线程池
ExecutorService executor = Executors.newFixedThreadPool(10);
// 问题：允许的请求队列长度为 Integer.MAX_VALUE，可能导致 OOM

// 正确 - 使用 ThreadPoolExecutor
ThreadPoolExecutor executor = new ThreadPoolExecutor(
    5,                          // corePoolSize
    10,                         // maximumPoolSize
    60L,                        // keepAliveTime
    TimeUnit.SECONDS,
    new LinkedBlockingQueue<>(100),  // 有界队列
    new ThreadFactoryBuilder().setNameFormat("pool-%d").build(),
    new ThreadPoolExecutor.CallerRunsPolicy()  // 拒绝策略
);
```

### Random 线程安全

```java
// 错误 - Random 实例被多线程同时使用会有性能问题
private static final Random random = new Random();

// 正确 - 使用 ThreadLocalRandom
private static final ThreadLocalRandom random = ThreadLocalRandom.current();
```

### volatile 不能保证原子性

```java
// 错误 - volatile 不能解决原子性问题
private volatile int count = 0;
count++; // 非原子操作

// 正确 - 使用 AtomicInteger
private AtomicInteger count = new AtomicInteger(0);
count.incrementAndGet();
```

## 控制语句

### 必须使用大括号

```java
// 错误
if (condition)
    doSomething();

// 正确
if (condition) {
    doSomething();
}
```

### switch 语句

```java
// 正确 - 每个 case 必须通过 break/return 终止或注释说明
switch (type) {
    case 1:
        doSomething();
        break;
    case 2:
        doSomethingElse();
        // 继续执行 case 3（fall-through）
    case 3:
        doAnotherThing();
        break;
    default:
        log.warn("Unknown type: {}", type);
        break;
}

// Java 21+ 使用 switch 表达式（更简洁）
String result = switch (type) {
    case 1 -> "one";
    case 2 -> "two";
    case 3 -> "three";
    default -> "unknown";
};
```

### 避免深层嵌套

```java
// 错误 - 嵌套超过 3 层
if (a) {
    if (b) {
        if (c) {
            doSomething();
        }
    }
}

// 正确 - 提前返回
if (!a) return;
if (!b) return;
if (!c) return;
doSomething();
```

## 异常处理

### 禁止捕获 Throwable

```java
// 错误
try {
    doSomething();
} catch (Throwable t) {
    // 处理
}

// 正确 - 捕获具体异常
try {
    doSomething();
} catch (IllegalArgumentException | NullPointerException e) {
    log.error("Parameter error", e);
    throw new BusinessException(ErrorCode.PARAM_ERROR, e.getMessage());
} catch (BusinessException e) {
    log.error("Business error", e);
    throw e;
}
```

### 禁止空 catch 块

```java
// 错误
try {
    doSomething();
} catch (Exception e) {}

// 正确
try {
    doSomething();
} catch (Exception e) {
    log.error("Operation failed", e);
    throw new BusinessException(ErrorCode.SYSTEM_ERROR, "操作失败");
}
```

### 保留原始异常信息

```java
// 错误 - 丢失原始异常
try {
    doSomething();
} catch (Exception e) {
    throw new BusinessException("Failed");
}

// 正确 - 传递原始异常
try {
    doSomething();
} catch (Exception e) {
    throw new BusinessException("Failed", e);
}
```

### 禁止在 finally 中使用 return

```java
// 错误 - finally 中的 return 会覆盖 try 中的返回值
try {
    return 1;
} finally {
    return 2; // 永远返回 2
}
```

## 日志规范

### 使用 SLF4J 门面

```java
// 错误 - 直接使用日志系统
import org.apache.log4j.Logger;
Logger logger = Logger.getLogger(getClass());

// 正确 - 使用 SLF4J
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
private static final Logger log = LoggerFactory.getLogger(UserService.class);
```

### 占位符方式

```java
// 错误 - 字符串拼接
logger.debug("Processing item: " + item);

// 正确 - 占位符
logger.debug("Processing item: {}", item);
```

### 日志级别规范

| 级别 | 使用场景 |
|------|---------|
| ERROR | 影响系统正常运行、当前请求正常运行的异常情况 |
| WARN | 不影响系统运行、但需要注意的异常情况 |
| INFO | 关键业务流程、核心系统状态变化 |
| DEBUG | 调试信息、问题排查 |

### 日志输出禁止敏感信息

```java
// 错误
log.info("User login: username={}, password={}", username, password);

// 正确
log.info("User login: username={}", username);
```

## MySQL 数据库规约

### 表名/字段名

```sql
-- 正确 - 使用小写字母或数字
CREATE TABLE user_order (
  id BIGINT PRIMARY KEY,
  user_id BIGINT NOT NULL COMMENT '用户ID',
  order_no VARCHAR(32) NOT NULL COMMENT '订单号',
  is_deleted TINYINT(1) DEFAULT 0 COMMENT '是否删除',
  create_time DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  update_time DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  UNIQUE KEY uk_order_no (order_no),
  KEY idx_user_id (user_id)
) COMMENT='用户订单表';
```

### 索引命名

```sql
-- 主键索引：pk_字段名
PRIMARY KEY (`id`)

-- 唯一索引：uk_字段名
UNIQUE KEY `uk_user_id` (`user_id`)

-- 普通索引：idx_字段名
KEY `idx_create_time` (`create_time`)
```

### 数据类型选择

```sql
-- 表达是与否概念的字段，必须使用 is_xxx
is_deleted TINYINT(1) DEFAULT 0,
is_enabled TINYINT(1) DEFAULT 1

-- 金额、精度要求高的场景使用 decimal
amount DECIMAL(10, 2)

-- 固定长度字符串
mobile_number CHAR(11)

-- 可变长字符串
remark VARCHAR(500)
```

### SQL 规约

```sql
-- 正确 - 使用 COUNT(*)
SELECT COUNT(*) FROM user;

-- 正确 - 使用 ISNULL 判断
SELECT * FROM user WHERE name IS NULL;

-- 错误 - 禁止左模糊
SELECT * FROM user WHERE name LIKE '%test%';

-- 正确 - 右模糊可以使用索引
SELECT * FROM user WHERE name LIKE 'test%';

-- 错误 - WHERE 条件中索引列不能参与计算
SELECT * FROM user WHERE age + 1 = 20;

-- 正确
SELECT * FROM user WHERE age = 19;

-- 更新时使用 LIMIT 限制行数
UPDATE user SET status = 1 WHERE id = 1 LIMIT 1;
```

## 工程结构

### 分层架构

```
┌─────────────────────────────────────┐
│        Controller 层                  │  - 接口暴露、参数接收
│        (@RestController)              │
├─────────────────────────────────────┤
│        Service 层                     │  - 业务逻辑编排
│        (@Service)                     │
├─────────────────────────────────────┤
│        Mapper/Repository 层          │  - 数据访问（MyBatis）
│        (@Mapper)                      │
└─────────────────────────────────────┘
```

### 包结构

```
com.example.app/
├── controller/        # 控制器层
├── service/           # 业务逻辑层
│   └── impl/         # 服务实现
├── mapper/            # MyBatis Mapper 接口
├── entity/            # 数据库实体
├── domain/            # 领域模型
├── dto/               # 数据传输对象（接收请求）
├── vo/                # 视图对象（返回响应）
├── enums/             # 枚举类
├── exception/         # 自定义异常
├── config/            # 配置类
├── util/              # 工具类
└── common/            # 公共类
```

### 依赖注入

```java
// 正确 - 使用 @RequiredArgsConstructor（Lombok）
@Service
@RequiredArgsConstructor
public class UserService {
    private final UserMapper userMapper;
    private final OrderService orderService;
}
```

## 安全规范

### SQL 注入防护

```java
// 错误 - 字符串拼接
String sql = "SELECT * FROM user WHERE name = '" + name + "'";

// 正确 - 使用参数化查询（MyBatis）
@Select("SELECT * FROM user WHERE name = #{name}")
User findByName(@Param("name") String name);

// 或使用 LambdaQueryWrapper（MyBatis-Plus）
LambdaQueryWrapper<User> wrapper = Wrappers.lambdaQuery();
wrapper.eq(User::getName, name);
```

### XSS 防护

```java
// 正确 - 对用户输入进行转义
String escapedInput = StringEscapeUtils.escapeHtml4(userInput);
```

### 密码安全

```java
// 正确 - 使用 BCrypt 哈希存储密码
String hashedPassword = BCrypt.hashpw(password, BCrypt.gensalt());

// 验证密码
boolean isMatch = BCrypt.checkpw(password, hashedPassword);
```

## 快速参考

### 常用注解

```java
// Spring 注解
@RestController      // 组合注解：@Controller + @ResponseBody
@RequestMapping     // 映射 HTTP 请求
@GetMapping          // 映射 GET 请求
@PostMapping         // 映射 POST 请求
@PutMapping          // 映射 PUT 请求
@DeleteMapping       // 映射 DELETE 请求

@Service            // 服务层组件
@Repository          // 数据访问层组件（MyBatis Mapper 可省略）
@Configuration       // 配置类

@Autowired          // 自动装配（推荐构造器注入）
@Value              // 注入配置值

@Component          // 通用组件
@Bean               // 定义 Bean

@Valid              // 参数校验
@Validated          // 分组校验

@Transactional      // 事务

// MyBatis-Plus 注解
@Mapper             // Mapper 接口
@TableName          // 表名映射
@TableId            // 主键映射
@TableField         // 字段映射
@TableLogic         // 逻辑删除

// Lombok 注解
@Data               // getter/setter/equals/hashCode/toString
@Builder            // 建造者模式
@AllArgsConstructor // 全参构造器
@NoArgsConstructor  // 无参构造器
@RequiredArgsConstructor // 必填参数构造器
@Slf4j              // 日志对象

// Validation 注解
@NotNull            // 不能为 null
@NotBlank           // 不能为空字符串
@NotEmpty           // 不能为空
@Size               // 大小范围
@Min/@Max           // 数值范围
@Email              // 邮箱格式
@Pattern            // 正则表达式

// Jackson 注解
@JsonFormat          // 日期格式化
@JsonProperty       // 属性名映射
@JsonIgnore         // 忽略序列化
```

### 常用工具类

```java
// Guava
Lists.newArrayList();
Maps.newHashMap();
Sets.newHashSet();

// Apache Commons Lang
StringUtils.isEmpty();
ObjectUtils.isEmpty();
BooleanUtils.isTrue();

// Hutool（国内常用）
CollUtil.newArrayList();
StrUtil.isEmpty();
DateUtil.now();
```

### 日期时间（Java 8+）

```java
// 获取当前时间
LocalDateTime now = LocalDateTime.now();

// 格式化
DateTimeFormatter formatter = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss");
String dateStr = now.format(formatter);

// 解析
LocalDateTime dateTime = LocalDateTime.parse(dateStr, formatter);

// 时间计算
LocalDateTime tomorrow = now.plusDays(1);
LocalDateTime nextMonth = now.plusMonths(1);
```

### Lambda 表达式和 Stream

```java
// 过滤
List<String> filtered = list.stream()
    .filter(s -> s.length() > 3)
    .collect(Collectors.toList());

// 映射
List<String> names = users.stream()
    .map(User::getName)
    .collect(Collectors.toList());

// 分组
Map<String, List<User>> grouped = users.stream()
    .collect(Collectors.groupingBy(User::getDepartment));

// 去重
List<String> distinct = list.stream()
    .distinct()
    .collect(Collectors.toList());
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

**记住**：遵循《阿里巴巴 Java 开发手册》，保持代码简洁、清晰、可维护。优先考虑可维护性而非微优化。
