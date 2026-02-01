---
name: java-coding-standards
description: Java 编码规范：命名规范、不可变性、Optional 使用、Stream 流、异常处理、泛型、项目结构。适配 Spring Boot 3 + Java 21。
---

# Java 编码规范

Spring Boot 服务中可读、可维护的 Java 21+ 代码规范。

## 核心原则

- 清晰优于聪明
- 默认不可变；最小化共享可变状态
- 快速失败，抛出有意义的异常
- 保持一致的命名和包结构

## 命名规范

```java
// 类/记录：大驼峰命名法（PascalCase）
public class MarketService {}
public record Money(BigDecimal amount, Currency currency) {}

// 方法/字段：小驼峰命名法（camelCase）
private final MarketRepository marketRepository;
public Market findBySlug(String slug) {}

// 常量：全大写下划线分隔（UPPER_SNAKE_CASE）
private static final int MAX_PAGE_SIZE = 100;
```

## 不可变性

```java
// 优先使用记录类和 final 字段
public record MarketDto(Long id, String name, MarketStatus status) {}

public class Market {
    private final Long id;
    private final String name;
    // 只提供 getter，不提供 setter
}
```

## Optional 使用

```java
// 查询方法返回 Optional
Optional<Market> market = marketRepository.findBySlug(slug);

// 使用 map/flatMap 代替 get()
return market
    .map(MarketResponse::from)
    .orElseThrow(() -> new EntityNotFoundException("市场不存在"));
```

## Stream 流最佳实践

```java
// 使用 Stream 进行转换，保持管道简短
List<String> names = markets.stream()
    .map(Market::getName)
    .filter(Objects::nonNull)
    .toList();

// 避免复杂的嵌套 Stream；为清晰起见使用循环
```

## 异常处理

- 对领域错误使用非受检异常；用上下文包装技术异常
- 创建领域特定异常（如 `MarketNotFoundException`）
- 避免广泛的 `catch (Exception ex)`，除非是集中重新抛出/记录

```java
throw new MarketNotFoundException(slug);
```

## 泛型和类型安全

- 避免原始类型；声明泛型参数
- 对可复用工具类优先使用有界泛型

```java
public <T extends Identifiable> Map<Long, T> indexById(Collection<T> items) { ... }
```

## 项目结构（Maven/Gradle）

```
src/main/java/com/example/app/
  config/          # 配置类
  controller/      # 控制器层
  service/         # 业务逻辑层
  mapper/          # MyBatis Mapper 接口
  entity/          # 实体类
  domain/          # 领域模型
  dto/             # 数据传输对象
  vo/              # 视图对象
  enums/           # 枚举类
  exception/       # 自定义异常
  util/            # 工具类
  common/          # 公共类
src/main/resources/
  application.yml           # 主配置文件
  application-dev.yml       # 开发环境配置
  application-prod.yml      # 生产环境配置
  mapper/                   # MyBatis XML 映射文件
src/test/java/...           # 测试代码（镜像 main 结构）
```

## 代码格式和风格

- 统一使用 2 或 4 个空格（项目标准）
- 每个文件一个公共顶层类型
- 保持方法简短专注；提取辅助方法
- 成员顺序：常量、字段、构造器、公共方法、保护方法、私有方法

## 代码异味避免

- 长参数列表 → 使用 DTO/Builder 模式
- 深层嵌套 → 提前返回
- 魔法数字 → 命名常量
- 静态可变状态 → 优先依赖注入
- 静默 catch 块 → 记录并处理或重新抛出

## 日志规范

```java
private static final Logger log = LoggerFactory.getLogger(MarketService.class);
log.info("查询市场 slug={}", slug);
log.error("查询市场失败 slug={}", slug, ex);
```

## 空值处理

- 仅在不可避免时接受 `@Nullable`；否则使用 `@NonNull`
- 在输入上使用 Bean Validation（`@NotNull`、`@NotBlank`）

## 测试规范

- JUnit 5 + AssertJ 进行流式断言
- Mockito 进行 Mock；尽可能避免部分 Mock
- 优先确定性测试；无隐藏延迟

**记住**：保持代码有意图、有类型、可观察。除非证明有必要，否则优先考虑可维护性而非微优化。
