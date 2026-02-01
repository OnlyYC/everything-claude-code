# 项目指南技能（示例）

这是项目特定技能的示例。使用此作为你自己项目的模板。

基于企业级 Java 后端应用程序最佳实践。

---

## 何时使用

在处理项目特定设计时参考此技能。项目技能包含：
- 架构概览
- 文件结构
- 代码模式
- 测试要求
- 部署工作流程

---

## 架构概览

**技术栈：**
- **JDK**：Java 21（虚拟线程、记录类、模式匹配）
- **框架**：Spring Boot 3.3+、Spring MVC、Spring Security 6
- **持久层**：MyBatis 3、MyBatis-Plus 3.5+
- **数据库**：MySQL 8.0+
- **构建工具**：Maven 3.9+
- **缓存**：Spring Cache + Caffeine（本地）/ Redis（分布式）
- **测试**：JUnit 5、Mockito、Testcontainers
- **部署**：Docker、Kubernetes

**分层架构：**
```
┌─────────────────────────────────────────────────────────────┐
│                      表现层 (Controller)                     │
│  Spring MVC REST API + Jackson JSON                        │
│  请求验证、统一响应、异常处理                                 │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                      业务层 (Service)                        │
│  Spring Transaction + AOP                                  │
│  业务逻辑、事务管理、缓存                                    │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    持久层 (Mapper/DAO)                       │
│  MyBatis + MyBatis-Plus                                     │
│  数据库操作、ORM 映射                                        │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
                        ┌──────────┐
                        │  MySQL   │
                        │ Database │
                        └──────────┘
```

---

## 文件结构

```
project/
├── pom.xml                               # Maven 配置
├── src/
│   ├── main/
│   │   ├── java/com/example/project/
│   │   │   ├── ProjectApplication.java  # Spring Boot 启动类
│   │   │   │
│   │   │   ├── controller/              # REST 控制器
│   │   │   │   ├── UserController.java
│   │   │   │   └── advice/              # 全局异常处理
│   │   │   │       ├── GlobalExceptionHandler.java
│   │   │   │       └── BusinessException.java
│   │   │   │
│   │   │   ├── dto/                     # 数据传输对象
│   │   │   │   ├── request/             # 请求 DTO
│   │   │   │   │   └── UserCreateRequest.java
│   │   │   │   ├── response/            # 响应 DTO
│   │   │   │   │   └── UserResponse.java
│   │   │   │   └── ApiResponse.java     # 统一响应封装
│   │   │   │
│   │   │   ├── service/                 # 业务服务接口
│   │   │   │   ├── UserService.java
│   │   │   │   └── impl/                # 业务服务实现
│   │   │   │       └── UserServiceImpl.java
│   │   │   │
│   │   │   ├── mapper/                  # MyBatis Mapper 接口
│   │   │   │   ├── UserMapper.java
│   │   │   │   └── BaseMapper.java      # 通用 Mapper 基类
│   │   │   │
│   │   │   ├── entity/                  # 数据库实体
│   │   │   │   ├── User.java
│   │   │   │   └── BaseEntity.java      # 通用实体基类
│   │   │   │
│   │   │   ├── config/                  # 配置类
│   │   │   │   ├── MybatisPlusConfig.java
│   │   │   │   ├── WebMvcConfig.java
│   │   │   │   └── SecurityConfig.java
│   │   │   │
│   │   │   ├── common/                  # 通用组件
│   │   │   │   ├── constant/            # 常量定义
│   │   │   │   ├── enums/               # 枚举定义
│   │   │   │   ├── exception/           # 自定义异常
│   │   │   │   └── util/                # 工具类
│   │   │   │
│   │   │   └── security/                # 安全认证
│   │   │       ├── JwtTokenProvider.java
│   │   │       └── UserDetailsServiceImpl.java
│   │   │
│   │   └── resources/
│   │       ├── application.yml          # 主配置文件
│   │       ├── application-dev.yml      # 开发环境配置
│   │       ├── application-prod.yml     # 生产环境配置
│   │       ├── mapper/                  # MyBatis XML 映射文件
│   │       │   └── UserMapper.xml
│   │       └── db/
│   │           └── migration/           # 数据库迁移脚本
│   │               └── V1__init_schema.sql
│   │
│   └── test/
│       └── java/com/example/project/
│           ├── controller/              # 控制器测试
│           ├── service/                 # 服务测试
│           └── mapper/                  # Mapper 测试
│
├── docker/
│   ├── Dockerfile
│   └── docker-compose.yml
├── docs/                                # 项目文档
└── scripts/                             # 构建和部署脚本
```

---

## 代码模式

### 统一响应封装

```java
package com.example.project.dto;

import com.fasterxml.jackson.annotation.JsonInclude;
import io.swagger.v3.oas.annotations.media.Schema;

@Schema(description = "统一API响应")
@JsonInclude(JsonInclude.Include.NON_NULL)
public record ApiResponse<T>(
    @Schema(description = "是否成功")
    Boolean success,

    @Schema(description = "响应数据")
    T data,

    @Schema(description = "错误信息")
    String error,

    @Schema(description = "响应代码")
    String code
) {
    public static <T> ApiResponse<T> ok(T data) {
        return new ApiResponse<>(true, data, null, "0000");
    }

    public static <T> ApiResponse<T> fail(String error) {
        return new ApiResponse<>(false, null, error, "9999");
    }

    public static <T> ApiResponse<T> fail(String code, String error) {
        return new ApiResponse<>(false, null, error, code);
    }
}
```

### 实体基类（使用记录模式和 MyBatis-Plus）

```java
package com.example.project.entity;

import com.baomidou.mybatisplus.annotation.*;
import io.swagger.v3.oas.annotations.media.Schema;
import lombok.Data;

import java.time.LocalDateTime;

@Data
@Schema(description = "基础实体")
public abstract class BaseEntity {

    @TableId(type = IdType.ASSIGN_ID)
    @Schema(description = "主键ID")
    private Long id;

    @TableField(fill = FieldFill.INSERT)
    @Schema(description = "创建时间")
    private LocalDateTime createTime;

    @TableField(fill = FieldFill.INSERT_UPDATE)
    @Schema(description = "更新时间")
    private LocalDateTime updateTime;

    @TableLogic
    @Schema(description = "逻辑删除标识")
    private Boolean deleted;
}
```

### MyBatis-Plus Mapper 接口

```java
package com.example.project.mapper;

import com.baomidou.mybatisplus.core.mapper.BaseMapper;
import com.example.project.entity.User;
import org.apache.ibatis.annotations.Mapper;
import org.apache.ibatis.annotations.Param;
import org.apache.ibatis.annotations.Select;

import java.util.List;
import java.util.Optional;

@Mapper
public interface UserMapper extends BaseMapper<User> {

    @Select("SELECT * FROM user WHERE status = #{status}")
    List<User> findByStatus(@Param("status") Integer status);

    default Optional<User> findByIdOpt(Long id) {
        return Optional.ofNullable(selectById(id));
    }
}
```

### Service 层实现

```java
package com.example.project.service.impl;

import com.baomidou.mybatisplus.core.conditions.query.LambdaQueryWrapper;
import com.baomidou.mybatisplus.extension.service.impl.ServiceImpl;
import com.example.project.dto.request.UserCreateRequest;
import com.example.project.dto.response.UserResponse;
import com.example.project.entity.User;
import com.example.project.enums.UserStatus;
import com.example.project.exception.BusinessException;
import com.example.project.mapper.UserMapper;
import com.example.project.service.UserService;
import lombok.RequiredArgsConstructor;
import org.springframework.cache.annotation.CacheEvict;
import org.springframework.cache.annotation.Cacheable;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;

@Service
@RequiredArgsConstructor
public class UserServiceImpl
        extends ServiceImpl<UserMapper, User>
        implements UserService {

    @Override
    @Transactional(rollbackFor = Exception.class)
    @CacheEvict(value = "users", allEntries = true)
    public UserResponse create(UserCreateRequest request) {
        // 检查用户名是否已存在
        if (existsByUsername(request.username())) {
            throw new BusinessException("USER_EXISTS", "用户名已存在");
        }

        User user = new User();
        user.setUsername(request.username());
        user.setEmail(request.email());
        user.setStatus(UserStatus.ACTIVE.getCode());

        save(user);
        return UserResponse.from(user);
    }

    @Override
    @Cacheable(value = "users", key = "#id")
    public UserResponse getById(Long id) {
        User user = baseMapper.findByIdOpt(id)
                .orElseThrow(() -> new BusinessException("USER_NOT_FOUND", "用户不存在"));
        return UserResponse.from(user);
    }

    @Override
    public List<UserResponse> listActiveUsers() {
        LambdaQueryWrapper<User> wrapper = new LambdaQueryWrapper<>();
        wrapper.eq(User::getStatus, UserStatus.ACTIVE.getCode());
        return list(wrapper).stream()
                .map(UserResponse::from)
                .toList();
    }

    private boolean existsByUsername(String username) {
        LambdaQueryWrapper<User> wrapper = new LambdaQueryWrapper<>();
        wrapper.eq(User::getUsername, username);
        return count(wrapper) > 0;
    }
}
```

### REST 控制器

```java
package com.example.project.controller;

import com.example.project.dto.ApiResponse;
import com.example.project.dto.request.UserCreateRequest;
import com.example.project.dto.response.UserResponse;
import com.example.project.service.UserService;
import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.tags.Tag;
import jakarta.validation.Valid;
import lombok.RequiredArgsConstructor;
import org.springframework.web.bind.annotation.*;

import java.util.List;

@Tag(name = "用户管理", description = "用户相关接口")
@RestController
@RequestMapping("/api/users")
@RequiredArgsConstructor
public class UserController {

    private final UserService userService;

    @Operation(summary = "创建用户")
    @PostMapping
    public ApiResponse<UserResponse> create(
            @Valid @RequestBody UserCreateRequest request) {
        return ApiResponse.ok(userService.create(request));
    }

    @Operation(summary = "获取用户详情")
    @GetMapping("/{id}")
    public ApiResponse<UserResponse> getById(@PathVariable Long id) {
        return ApiResponse.ok(userService.getById(id));
    }

    @Operation(summary = "获取活跃用户列表")
    @GetMapping("/active")
    public ApiResponse<List<UserResponse>> listActive() {
        return ApiResponse.ok(userService.listActiveUsers());
    }
}
```

### 全局异常处理

```java
package com.example.project.controller.advice;

import com.example.project.dto.ApiResponse;
import com.example.project.exception.BusinessException;
import lombok.extern.slf4j.Slf4j;
import org.springframework.http.HttpStatus;
import org.springframework.validation.BindException;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.ResponseStatus;
import org.springframework.web.bind.annotation.RestControllerAdvice;

@Slf4j
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(BusinessException.class)
    @ResponseStatus(HttpStatus.OK)
    public ApiResponse<?> handleBusinessException(BusinessException e) {
        log.warn("Business exception: code={}, message={}",
                e.getCode(), e.getMessage());
        return ApiResponse.fail(e.getCode(), e.getMessage());
    }

    @ExceptionHandler(BindException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ApiResponse<?> handleValidationException(BindException e) {
        String message = e.getBindingResult().getAllErrors().get(0)
                .getDefaultMessage();
        return ApiResponse.fail("VALIDATION_ERROR", message);
    }

    @ExceptionHandler(Exception.class)
    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    public ApiResponse<?> handleException(Exception e) {
        log.error("Unexpected error", e);
        return ApiResponse.fail("SYSTEM_ERROR", "系统错误，请稍后重试");
    }
}
```

### 自定义业务异常

```java
package com.example.project.exception;

import lombok.Getter;

import java.io.Serializable;

@Getter
public class BusinessException extends RuntimeException {

    private final String code;

    public BusinessException(String code, String message) {
        super(message);
        this.code = code;
    }

    public BusinessException(String message) {
        this("BUSINESS_ERROR", message);
    }
}
```

### DTO 请求示例

```java
package com.example.project.dto.request;

import io.swagger.v3.oas.annotations.media.Schema;
import jakarta.validation.constraints.Email;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.Size;

@Schema(description = "创建用户请求")
public record UserCreateRequest(

    @Schema(description = "用户名", required = true)
    @NotBlank(message = "用户名不能为空")
    @Size(min = 3, max = 20, message = "用户名长度为3-20个字符")
    String username,

    @Schema(description = "邮箱", required = true)
    @NotBlank(message = "邮箱不能为空")
    @Email(message = "邮箱格式不正确")
    String email

) {}
```

### DTO 响应示例

```java
package com.example.project.dto.response;

import com.example.project.entity.User;
import io.swagger.v3.oas.annotations.media.Schema;

import java.time.LocalDateTime;

@Schema(description = "用户响应")
public record UserResponse(

    @Schema(description = "用户ID")
    Long id,

    @Schema(description = "用户名")
    String username,

    @Schema(description = "邮箱")
    String email,

    @Schema(description = "状态")
    Integer status,

    @Schema(description = "创建时间")
    LocalDateTime createTime

) {
    public static UserResponse from(User user) {
        return new UserResponse(
            user.getId(),
            user.getUsername(),
            user.getEmail(),
            user.getStatus(),
            user.getCreateTime()
        );
    }
}
```

### 配置类示例

```java
package com.example.project.config;

import com.baomidou.mybatisplus.annotation.DbType;
import com.baomidou.mybatisplus.extension.plugins.MybatisPlusInterceptor;
import com.baomidou.mybatisplus.extension.plugins.inner.PaginationInnerInterceptor;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class MybatisPlusConfig {

    @Bean
    public MybatisPlusInterceptor mybatisPlusInterceptor() {
        MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();
        // 分页插件
        interceptor.addInnerInterceptor(
            new PaginationInnerInterceptor(DbType.MYSQL)
        );
        return interceptor;
    }
}
```

### application.yml 配置

```yaml
spring:
  application:
    name: project-api
  profiles:
    active: dev

  # 数据源配置
  datasource:
    driver-class-name: com.mysql.cj.jdbc.Driver
    url: jdbc:mysql://localhost:3306/project?useUnicode=true&characterEncoding=utf8&useSSL=false&serverTimezone=Asia/Shanghai&allowPublicKeyRetrieval=true
    username: root
    password: ${DB_PASSWORD:}
    hikari:
      maximum-pool-size: 20
      minimum-idle: 5
      connection-timeout: 30000
      idle-timeout: 600000
      max-lifetime: 1800000

  # 缓存配置
  cache:
    type: caffeine
    caffeine:
      spec: maximumSize=1000,expireAfterWrite=10m

# MyBatis-Plus 配置
mybatis-plus:
  configuration:
    map-underscore-to-camel-case: true
    log-impl: org.apache.ibatis.logging.slf4j.Slf4jImpl
  global-config:
    db-config:
      logic-delete-field: deleted
      logic-delete-value: 1
      logic-not-delete-value: 0
      id-type: assign_id
  mapper-locations: classpath*:/mapper/**/*.xml

# 服务器配置
server:
  port: 8080
  shutdown: graceful

# 日志配置
logging:
  level:
    com.example.project.mapper: debug
    com.example.project: info

# Swagger 配置
springdoc:
  api-docs:
    enabled: true
  swagger-ui:
    enabled: true
    path: /swagger-ui.html
```

---

## 测试要求

### 单元测试（JUnit 5 + Mockito）

```bash
# 执行所有测试
mvn test

# 执行带覆盖率的测试
mvn clean test jacoco:report

# 执行特定测试类
mvn test -Dtest=UserServiceTest
```

**Service 测试示例：**
```java
package com.example.project.service;

import com.example.project.dto.request.UserCreateRequest;
import com.example.project.dto.response.UserResponse;
import com.example.project.entity.User;
import com.example.project.enums.UserStatus;
import com.example.project.mapper.UserMapper;
import com.example.project.service.impl.UserServiceImpl;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import static org.assertj.core.api.Assertions.*;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.*;

@ExtendWith(MockitoExtension.class)
@DisplayName("用户服务测试")
class UserServiceTest {

    @Mock
    private UserMapper userMapper;

    @InjectMocks
    private UserServiceImpl userService;

    @Test
    @DisplayName("创建用户 - 成功")
    void create_Success() {
        // Given
        UserCreateRequest request = new UserCreateRequest(
            "testuser", "test@example.com"
        );

        User savedUser = new User();
        savedUser.setId(1L);
        savedUser.setUsername("testuser");
        savedUser.setEmail("test@example.com");
        savedUser.setStatus(UserStatus.ACTIVE.getCode());

        when(userMapper.existsByUsername("testuser")).thenReturn(false);
        when(userMapper.insert(any(User.class))).thenReturn(1);

        // When
        UserResponse response = userService.create(request);

        // Then
        assertThat(response.username()).isEqualTo("testuser");
        assertThat(response.email()).isEqualTo("test@example.com");
        verify(userMapper).insert(any(User.class));
    }

    @Test
    @DisplayName("创建用户 - 用户名已存在时抛出异常")
    void create_UsernameExists_ThrowsException() {
        // Given
        UserCreateRequest request = new UserCreateRequest(
            "existing", "test@example.com"
        );
        when(userMapper.existsByUsername("existing")).thenReturn(true);

        // When & Then
        assertThatThrownBy(() -> userService.create(request))
                .isInstanceOf(BusinessException.class)
                .hasMessageContaining("用户名已存在");
    }
}
```

### 集成测试（Testcontainers）

```java
package com.example.project.integration;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.web.client.TestRestTemplate;
import org.springframework.test.context.DynamicPropertyRegistry;
import org.springframework.test.context.DynamicPropertySource;
import org.testcontainers.containers.MySQLContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;

import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@Testcontainers
class UserIntegrationTest {

    @Container
    static MySQLContainer<?> mysql = new MySQLContainer<>(
        "mysql:8.0"
    );

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", mysql::getJdbcUrl);
        registry.add("spring.datasource.username", mysql::getUsername);
        registry.add("spring.datasource.password", mysql::getPassword);
    }

    @Autowired
    private TestRestTemplate restTemplate;

    @Test
    void shouldCreateUser() {
        // Given
        UserCreateRequest request = new UserCreateRequest(
            "integration", "integration@test.com"
        );

        // When
        var response = restTemplate.postForEntity(
            "/api/users", request, ApiResponse.class
        );

        // Then
        assertThat(response.getStatusCode().value()).isEqualTo(200);
        assertThat(response.getBody().success()).isTrue();
    }
}
```

### Controller 测试

```java
package com.example.project.controller;

import com.example.project.dto.request.UserCreateRequest;
import com.example.project.dto.response.UserResponse;
import com.example.project.service.UserService;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
import org.springframework.boot.test.mock.mockito.MockBean;
import org.springframework.http.MediaType;
import org.springframework.test.web.servlet.MockMvc;

import static org.mockito.Mockito.when;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@WebMvcTest(UserController.class)
class UserControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @Autowired
    private ObjectMapper objectMapper;

    @MockBean
    private UserService userService;

    @Test
    void create_ReturnsSuccess() throws Exception {
        // Given
        UserCreateRequest request = new UserCreateRequest(
            "test", "test@example.com"
        );
        UserResponse response = new UserResponse(
            1L, "test", "test@example.com", 1, null
        );

        when(userService.create(request)).thenReturn(response);

        // When & Then
        mockMvc.perform(post("/api/users")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(request)))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.success").value(true))
                .andExpect(jsonPath("$.data.username").value("test"));
    }
}
```

---

## 部署工作流程

### 部署前检查清单

- [ ] `mvn clean test` 测试全部通过
- [ ] 代码覆盖率 >= 80%
- [ ] `mvn checkstyle:check` 代码规范检查通过
- [ ] 无写死密钥，敏感配置使用环境变量
- [ ] 数据库迁移脚本已准备
- [ ] Swagger API 文档已更新

### Maven 构建命令

```bash
# 清理编译
mvn clean compile

# 打包（跳过测试）
mvn clean package -DskipTests

# 打包并运行测试
mvn clean package

# 指定环境打包
mvn clean package -P prod

# 生成依赖分析报告
mvn dependency:tree
```

### Docker 部署

**Dockerfile：**
```dockerfile
FROM eclipse-temurin:21-jre-alpine

WORKDIR /app

COPY target/project-api-*.jar app.jar

EXPOSE 8080

ENTRYPOINT ["java", \
    "-XX:+UseZGC", \
    "-XX:+UseStringDeduplication", \
    "-Dspring.profiles.active=prod", \
    "-jar", "app.jar"]
```

**docker-compose.yml：**
```yaml
version: '3.8'

services:
  api:
    build: .
    ports:
      - "8080:8080"
    environment:
      - SPRING_PROFILES_ACTIVE=prod
      - DB_PASSWORD=${DB_PASSWORD}
    depends_on:
      - mysql
      - redis

  mysql:
    image: mysql:8.0
    environment:
      - MYSQL_ROOT_PASSWORD=${DB_PASSWORD}
      - MYSQL_DATABASE=project
    volumes:
      - mysql-data:/var/lib/mysql

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

volumes:
  mysql-data:
```

### 环境变量

```bash
# 生产环境 (.env.prod)
SPRING_PROFILES_ACTIVE=prod
DB_HOST=mysql-prod.example.com
DB_PORT=3306
DB_NAME=project
DB_USERNAME=app_user
DB_PASSWORD=your_secure_password

REDIS_HOST=redis-prod.example.com
REDIS_PORT=6379
REDIS_PASSWORD=

JWT_SECRET=your_jwt_secret_key
JWT_EXPIRATION=86400000

# 监控配置
MANAGEMENT_ENDPOINTS_WEB_EXPOSURE_INCLUDE=health,info,metrics
MANAGEMENT_METRICS_EXPORT_PROMETHEUS_ENABLED=true
```

---

## 关键规则

1. **Java 21 特性优先** - 优先使用记录类、模式匹配、虚拟线程
2. **不可变性** - DTO 使用 record，避免可变状态
3. **TDD** - 实现前先编写测试
4. **80% 覆盖率** 最低要求
5. **单一职责** - 每个类只负责一个功能
6. **依赖注入** - 使用构造器注入（@RequiredArgsConstructor）
7. **异常处理** - 统一使用 BusinessException + 全局处理器
8. **事务管理** - Service 方法添加 @Transactional 注解
9. **日志规范** - 使用 Slf4j，生产环境禁用 DEBUG 日志
10. **代码规范** - 遵循 Alibaba Java Coding Guidelines

---

## 相关技能

- `java-coding-standards` - Java 编码最佳实践
- `springboot-patterns` - Spring Boot 设计模式
- `springboot-tdd` - Spring Boot TDD 方法论
- `java-testing` - Java 测试指南
- `backend-patterns` - 后端通用模式
