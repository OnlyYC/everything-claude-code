---
description: Generate and run integration tests with JUnit 5, RestAssured, and TestContainers. Creates API integration tests, runs full stack tests, captures logs, and generates reports.
---

# 集成测试指令

此指令调用 **integration-test-runner** Agent 来产生、维护和执行使用 JUnit 5 + RestAssured + TestContainers 的集成测试。

## 此指令的功能

1. **生成集成测试用例** - 为 API 接口建立 JUnit 5 测试
2. **执行集成测试** - 使用真实数据库环境执行测试
3. **捕获测试产物** - 失败时的日志、SQL 语句、堆栈跟踪
4. **生成测试报告** - Surefire HTML 报告和 JUnit XML
5. **识别不稳定测试** - 隔离不稳定的测试

## 何时使用

在以下情况使用 `/e2e`：
- 测试关键业务流程（用户注册、下单、支付）
- 验证多层级集成（Controller -> Service -> Mapper -> DB）
- 测试 API 接口契约和响应格式
- 验证前后端集成
- 为生产环境部署做准备

## 运作方式

integration-test-runner Agent 会：

1. **分析 API 接口**并识别测试场景
2. **生成 JUnit 5 测试**使用 AAA 模式（Arrange-Act-Assert）
3. **启动真实环境**（TestContainers MySQL + Redis）
4. **执行测试**并捕获失败信息
5. **生成报告**包含结果和产物
6. **识别不稳定测试**并建议修复

## 技术栈

- **JUnit 5** - 测试框架
- **RestAssured** - REST API 测试
- **TestContainers** - 容器化测试环境（MySQL、Redis）
- **Mockito** - Mock 外部服务
- **WireMock** - Mock 第三方 API
- **Jacoco** - 覆盖率统计

## 测试产物

测试执行时，会捕获以下产物：

**所有测试：**
- Surefire HTML 报告
- JUnit XML 用于 CI 集成

**仅在失败时：**
- 应用日志（logs/application.log）
- SQL 执行日志
- HTTP 请求/响应日志
- 堆栈跟踪

## 查看产物

```bash
# 在浏览器查看 Surefire 报告
open target/site/surefire-report.html

# 查看 Jacoco 覆盖率报告
open target/site/jacoco/index.html

# 查看测试日志
cat target/logs/application.log
```

## 最佳实践

**应该做：**
- 使用 TestContainers 启动真实数据库
- 使用 AAA 模式组织测试代码
- 测试 API 接口而非实现细节
- 测试正常流程和异常流程
- 使用 @Transactional 回滚测试数据
- 使用随机端口避免端口冲突

**不应该做：**
- 直接测试 Service 层（应通过 API 测试）
- 对生产环境执行测试
- 忽略不稳定的测试
- 用集成测试覆盖所有边界情况（使用单元测试）
- 硬编码测试数据（使用测试数据构建器）

## 快速指令

```bash
# 执行所有集成测试
mvn verify

# 执行特定测试类
mvn test -Dtest=UserControllerIntegrationTest

# 执行特定测试方法
mvn test -Dtest=UserControllerIntegrationTest#testCreateUser

# 跳过集成测试
mvn package -DskipITs

# 只执行集成测试
mvn verify -DskipUnitTests

# 生成覆盖率报告
mvn verify jacoco:report
```

## 测试示例

```java
@SpringBootTest(webEnvironment = WebEnvironment.RANDOM_PORT)
@Testcontainers
class UserControllerIntegrationTest {

    @Container
    static MySQLContainer<?> mysql = new MySQLContainer<>("mysql:8.0");

    @Container
    static GenericContainer<?> redis = new GenericContainer<>("redis:7-alpine")
            .withExposedPorts(6379);

    @Autowired
    private TestRestTemplate restTemplate;

    @Test
    @DisplayName("创建用户 - 成功")
    void createUser_Success() {
        // Arrange
        CreateUserRequest request = CreateUserRequest.builder()
                .username("testuser")
                .password("Password123")
                .email("test@example.com")
                .build();

        // Act
        ResponseEntity<Result<UserVO>> response = restTemplate.postForEntity(
                "/api/users",
                request,
                new ParameterizedTypeReference<>() {}
        );

        // Assert
        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.OK);
        assertThat(response.getBody().getCode()).isEqualTo(200);
        assertThat(response.getBody().getData().getUsername()).isEqualTo("testuser");
    }
}
```

## 与其他指令的整合

- 使用 `/plan` 识别要测试的关键业务流程
- 使用 `/tdd` 进行单元测试（更快、更细粒度）
- 使用 `/code-review` 验证测试质量
- 使用 `/test-coverage` 验证整体覆盖率

## 相关 Agent

此指令调用位于以下位置的 `integration-test-runner` Agent：
`~/.claude/agents/integration-test-runner.md`
