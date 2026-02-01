# 用户级 CLAUDE.md 示例

这是一个用户级 CLAUDE.md 文件示例，请放置于 `~/.claude/CLAUDE.md`。

用户级配置全局应用于所有项目，用于：
- 个人编码偏好
- 必须强制执行的全局规则
- 模块化规则的链接

---

## 核心理念

你是 Claude Code。我使用专门的 agent 和 skill 来处理复杂任务。

**核心原则：**
1. **Agent 优先**：将复杂工作委托给专门的 agent
2. **并行执行**：尽可能使用 Task 工具启动多个 agent
3. **先计划后执行**：复杂操作使用计划模式
4. **测试驱动**：先写测试，后写实现
5. **安全优先**：绝不妥协安全性

---

## 模块化规则

详细指南位于 `~/.claude/rules/`：

| 规则文件 | 内容 |
|-----------|----------|
| security.md | 安全检查、密钥管理、JWT 最佳实践 |
| coding-style.md | Java 代码风格、Lombok 使用、不可变模式 |
| testing.md | JUnit 5 + Mockito、Testcontainers、80% 覆盖率 |
| git-workflow.md | 提交格式、PR 工作流 |
| agents.md | Agent 编排、何时使用哪个 agent |
| patterns.md | Spring Boot 模式、Service/Repository 模式 |
| performance.md | JVM 优化、缓存策略 |
| hooks.md | 钩子系统 |

---

## 可用 Agent

位于 `~/.claude/agents/`：

| Agent | 用途 |
|-------|---------|
| planner | 功能实现规划 |
| architect | 系统设计与架构 |
| tdd-guide | 测试驱动开发（JUnit 5） |
| code-reviewer | Java 代码审查（Checkstyle、SpotBugs） |
| security-reviewer | 安全漏洞分析 |
| build-error-resolver | Maven 构建错误解决 |
| e2e-runner | Spring Boot 集成测试 |
| refactor-cleaner | 无用代码清理、Java 重构 |
| doc-updater | Javadoc 和 API 文档更新 |

---

## 个人偏好

### 隐私安全
- 日志输出必须脱敏；禁止粘贴密钥（API Key/Token/密码/JWT）
- 分享前先审查输出 - 移除敏感数据
- 禁止提交 `application-local.yml` 或 `.env` 文件

### 代码风格
- 代码、注释、文档中禁止使用 emoji
- 使用 Lombok 简化样板代码
- 优先使用不可变对象 - 使用 `record`、`@Value`、`final` 字段
- 宁可多写小文件，不写大文件
- 单文件 200-400 行为宜，上限 800 行
- 严格遵循 Java 命名规范

### Java 特定规范
- 类型明显时使用 `var`
- 禁止返回 `null` - 使用 `Optional` 或空集合
- 使用 `@RequiredArgsConstructor` 进行依赖注入
- 使用 `@Slf4j` 代替手动创建 logger
- 使用 Stream API 处理集合
- Service 层方法优先使用 `@Transactional`

### Spring Boot 特定规范
- Controller 要瘦，Service 要胖
- DTO 用于接口契约，Entity 用于数据库映射
- 使用 `@Valid` + Jakarta Validation 校验入参
- 使用 `@ControllerAdvice` 处理异常
- 分离环境配置：dev、test、prod

### Git 规范
- 提交规范：`feat:` 新功能、`fix:` 修复、`refactor:` 重构、`docs:` 文档、`test:` 测试、`chore:` 构建
- 提交前本地测试：`mvn clean test`
- 小而聚焦的提交
- 禁止直接提交到 main 分支

### 测试规范
- 测试驱动开发（TDD）：先写测试
- 代码覆盖率最低 80%
- 使用 JUnit 5 + Mockito 编写单元测试
- 使用 `@SpringBootTest` 编写集成测试
- 使用 Testcontainers 进行真实 DB/Redis 测试

---

## 编辑器集成

我使用 IntelliJ IDEA 作为主力编辑器：
- 自动生成 Javadoc
- 保存时自动优化导入、格式化代码
- Maven 工具窗口管理依赖
- HTTP Client 进行接口测试

---

## 常用 Java 模式

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
        return userMapper.selectById(id)
            .map(UserVO::fromEntity)
            .orElseThrow(() -> new NotFoundException("用户不存在"));
    }
}
```

### DTO 模式

```java
public record UserCreateRequest(
    @NotBlank String username,
    @NotBlank @Size(min = 6) String password,
    @Email String email
) {}

public record UserVO(
    Long id,
    String username,
    LocalDateTime createdAt
) {
    public static UserVO fromEntity(User user) {
        return new UserVO(user.getId(), user.getUsername(), user.getCreatedAt());
    }
}
```

---

## 成功标准

当满足以下条件时，你就成功了：
- 所有测试通过（覆盖率 80%+）
- 无安全漏洞（OWASP、依赖扫描）
- 代码符合 Spring Boot 最佳实践
- Maven 构建成功：`mvn clean verify`
- Checkstyle 和 SpotBugs 检查通过
- 满足用户需求

---

**理念**：Agent 优先设计、并行执行、谋定后动、测试先行、安全第一。
