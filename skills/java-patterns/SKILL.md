---
name: java-patterns
description: Java 开发强制规则：基于《阿里巴巴 Java 开发手册》，涵盖强制性编程规约、异常处理、并发安全、MySQL 规约。适配 Java 21 + Spring Boot 3 技术栈。
version: 1.1.0
tech_stack: [Java 21, Spring Boot 3, MyBatis-Plus, MySQL]
tools: Read, Write, Edit, Bash, Grep, Glob
related_skills:
  java-coding-standards: "本 skill 负责 Alibaba 强制性规则；java-coding-standards 负责 Java 21 编码风格与语法"
  java-testing: "通用测试框架指南"
  mysql-patterns: "MySQL 数据库优化"
---

# Java 开发强制规则

基于《阿里巴巴 Java 开发手册（嵩山版）》整理的**强制性规则**和**最佳实践**。适配 Java 21 + Spring Boot 3 + MyBatis-Plus + MySQL 技术栈。

## 技能职责划分

| 技能 | 职责 | 内容范围 |
|------|------|---------|
| `java-patterns` | **强制性规则** | Alibaba 手册的强制规约、安全规范、性能要求 |
| `java-coding-standards` | **编码风格** | Java 21 语法特性、命名约定、代码格式 |

## 核心原则

- **强制优先** - Alibaba 手册的强制规约必须遵守
- **安全第一** - SQL 注入、XSS、密钥泄露等安全问题零容忍
- **性能意识** - N+1 查询、内存泄漏、线程安全问题必须避免
- **可维护性** - 代码被阅读的次数远多于被撰写的次数

## 强制性命名规约

### 禁止使用拼音

```java
// 错误 - 强制禁止
String zifu = "abc";
boolean youxiao = true;

// 正确
String character = "abc";
boolean valid = true;
```

### 布尔变量命名

```java
// 错误 - 布尔变量不要以 is 开头（数据类型会混淆）
private boolean isSuccess;  // getter 变成 isIsSuccess()

// 正确
private boolean success;
private boolean deleted;
private boolean enabled;
```

### 包命名规范

```java
// 强制 - 全部小写，域名反写
package com.alibaba.fastjson;
package org.springframework.web;
package com.example.app.controller;

// 错误
package com.example.app.Controller;  // 大写错误
```

## OOP 强制规约

### 包装类比较 - 强制使用 equals

```java
// 错误 - [强制] 所有包装类对象之间值的比较禁止使用 ==
Integer a = 1;
Integer b = 1;
if (a == b) {}  // 错误！-128~127 之外的对象会失败

// 正确
if (a.equals(b)) {}
if (Objects.equals(a, b)) {}  // 推荐 - 避免 NPE
```

### equals 和 hashCode - 强制同步重写

```java
// [强制] 重写 equals 必须重写 hashCode
@Data  // Lombok 自动生成
public class User {
    private Long id;
    private String name;
}
```

### 金额计算 - 强制使用 BigDecimal

```java
// [强制] 禁止使用构造方法 BigDecimal(double)
// 错误
BigDecimal b1 = new BigDecimal(0.1);  // 精度丢失

// 正确
BigDecimal b1 = new BigDecimal("0.1");
BigDecimal b2 = BigDecimal.valueOf(0.1);

// 错误 - 禁止 float/double 计算金额
double amount = 0.1 + 0.2;  // 0.30000000000000004
```

### long 类型赋值 - 强制大写 L

```java
// [强制] long 赋值必须使用大写 L
// 错误
long num = 1l;  // 小写 l 容易和数字 1 混淆

// 正确
long num = 1L;
```

## 集合处理强制规约

### subList 使用 - 强制不可强转

```java
// [强制] subList 返回的是内部类，不可强转成 ArrayList
// 错误
List<Integer> subList = list.subList(0, 5);
ArrayList<Integer> sub = (ArrayList<Integer>) subList;  // ClassCastException

// 正确 - 使用 new ArrayList 包装
List<Integer> subList = new ArrayList<>(list.subList(0, 5));
```

### 数组转集合 - 强制注意可变性

```java
// [强制] Arrays.asList 返回的不可修改
// 错误
String[] array = {"a", "b", "c"};
List<String> list = Arrays.asList(array);
list.add("d");  // UnsupportedOperationException

// 正确 - 使用 ArrayList 包装
List<String> list = new ArrayList<>(Arrays.asList(array));
```

### foreach 中禁止 remove/add

```java
// [强制] foreach 循环中禁止 remove/add 元素
// 错误
for (String item : list) {
    if ("remove".equals(item)) {
        list.remove(item);  // ConcurrentModificationException
    }
}

// 正确 - 使用 removeIf（推荐）
list.removeIf("remove"::equals);
```

## 并发处理强制规约

### SimpleDateFormat - 强制使用线程安全版本

```java
// [强制] SimpleDateFormat 是线程不安全的，禁止定义为 static
// 错误
private static final SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd");

// 正确 - 使用 DateTimeFormatter（Java 8+）
private static final DateTimeFormatter formatter =
    DateTimeFormatter.ofPattern("yyyy-MM-dd");
String dateStr = LocalDateTime.now().format(formatter);
```

### 线程池创建 - 强制使用 ThreadPoolExecutor

```java
// [强制] 禁止使用 Executors 创建线程池（队列无界可能导致 OOM）
// 错误
ExecutorService executor = Executors.newFixedThreadPool(10);
// 问题：队列长度 Integer.MAX_VALUE，可能导致 OOM

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

### Random - 强制使用 ThreadLocalRandom

```java
// [强制] Random 实例多线程使用有性能问题
// 错误
private static final Random random = new Random();

// 正确 - 使用 ThreadLocalRandom
private static final ThreadLocalRandom random = ThreadLocalRandom.current();
```

### volatile 不能保证原子性

```java
// 错误 - volatile 不能解决原子性问题
private volatile int count = 0;
count++;  // 非原子操作

// 正确 - 使用 AtomicInteger
private AtomicInteger count = new AtomicInteger(0);
count.incrementAndGet();
```

## 控制语句强制规约

### 必须使用大括号

```java
// [强制] if/else/for/while 必须使用大括号
// 错误
if (condition)
    doSomething();

// 正确
if (condition) {
    doSomething();
}
```

### switch fall-through - 强制注释说明

```java
// [强制] 每个 case 必须通过 break/return 终止或注释说明
// 正确
switch (type) {
    case 1:
        doSomething();
        break;
    case 2:
        doSomethingElse();
        // fall-through（继续执行 case 3）
    case 3:
        doAnotherThing();
        break;
    default:
        log.warn("Unknown type: {}", type);
        break;
}
```

### 避免深层嵌套

```java
// [强制] 嵌套不超过 3 层
// 错误
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

## 异常处理强制规约

### 禁止捕获 Throwable

```java
// [强制] 禁止捕获 Throwable
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
}
```

### 禁止空 catch 块

```java
// [强制] 禁止使用空的 catch 块
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
// [强制] 不要丢弃原始异常
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
// [强制] finally 中禁止使用 return
// 错误 - finally 中的 return 会覆盖 try 中的返回值
try {
    return 1;
} finally {
    return 2;  // 永远返回 2
}
```

## 日志强制规约

### 使用 SLF4J 门面

```java
// [强制] 禁止直接使用 Log4j/Logback
// 错误
import org.apache.log4j.Logger;
Logger logger = Logger.getLogger(getClass());

// 正确 - 使用 SLF4J
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
private static final Logger log = LoggerFactory.getLogger(UserService.class);
```

### 占位符方式

```java
// [强制] 禁止字符串拼接
// 错误
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

### 禁止输出敏感信息

```java
// [强制] 日志中禁止输出敏感信息
// 错误
log.info("User login: username={}, password={}", username, password);

// 正确
log.info("User login: username={}", username);
```

## 安全强制规约

### SQL 注入防护

```java
// [强制] 禁止字符串拼接 SQL
// 错误
String sql = "SELECT * FROM user WHERE name = '" + name + "'";

// 正确 - 使用参数化查询（MyBatis）
@Select("SELECT * FROM user WHERE name = #{name}")
User findByName(@Param("name") String name);

// 或使用 LambdaQueryWrapper（MyBatis-Plus）
LambdaQueryWrapper<User> wrapper = Wrappers.lambdaQuery();
wrapper.eq(User::getName, name);
```

### 密码安全

```java
// [强制] 密码必须使用 BCrypt 哈希存储
String hashedPassword = BCrypt.hashpw(password, BCrypt.gensalt());

// 验证密码
boolean isMatch = BCrypt.checkpw(password, hashedPassword);
```

**记住**：Alibaba 手册的强制规约必须严格遵守。这些规则能避免 80% 的常见 Bug 和安全问题。
