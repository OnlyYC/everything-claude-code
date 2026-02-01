# 安全指南

> **适用范围**：以下安全指南适用于 **Java / Spring Boot** 项目

## 强制安全检查

任何代码提交前，必须通过以下检查：

- [ ] 没有硬编码的密钥（API 密钥、密码、Token）
- [ ] 所有用户输入已校验
- [ ] SQL 注入防护（MyBatis 参数化查询）
- [ ] XSS 防护（输出转义）
- [ ] 已启用 CSRF 保护
- [ ] 已验证认证/授权
- [ ] 所有接口都有限流控制
- [ ] 错误信息不泄露敏感数据
- [ ] 敏感数据已加密存储
- [ ] 依赖无已知漏洞

## 密钥管理

### 禁止硬编码密钥

```java
// ❌ 绝对禁止：硬编码密钥
public class PayService {
    private static final String API_KEY = "sk-proj-xxxxx";  // 危险！
}

// ❌ 禁止：配置文件中明文密钥
// application.yml
pay:
  api-key: sk-proj-xxxxx  // 不要提交到版本控制！
```

### 推荐方式

```java
// ✓ 推荐：使用配置中心（不提交到版本控制）
@Configuration
public class PayConfig {
    @Value("${pay.api-key}")
    private String apiKey;

    @Value("${pay.api-secret}")
    private String apiSecret;

    @Bean
    public PayClient payClient() {
        if (StringUtils.isBlank(apiKey)) {
            throw new IllegalStateException("pay.api-key 未配置");
        }
        return new PayClient(apiKey, apiSecret);
    }
}

// ✓ 推荐：使用 Nacos/Apollo 配置中心
@RefreshScope
@Configuration
public class DynamicConfig {
    @NacosValue(value = "${pay.api-key}", autoRefreshed = true)
    private String apiKey;
}

// ✓ 推荐：使用环境变量
@Configuration
public class EnvConfig {
    @Value("${PAY_API_KEY:#{null}}")
    private String apiKey;
}
```

### 密钥配置隔离

```bash
# 环境隔离
config/
├── application-dev.yml      # 开发环境（可包含测试密钥）
├── application-test.yml     # 测试环境
├── application-prod.yml     # 生产环境（从外部注入）
└── .gitignore               # 忽略敏感配置

# .gitignore
application-prod.yml
*.secret
```

## 认证与授权

### Spring Security 配置

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity(prePostEnabled = true)
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        return http
            // 禁用 CSRF（API 项目）
            .csrf(csrf -> csrf.disable())

            // CORS 配置
            .cors(cors -> cors.configurationSource(corsConfigurationSource()))

            // 无状态会话
            .sessionManagement(session -> session
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS)
            )

            // 授权规则
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/public/**").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            )

            // JWT 过滤器
            .addFilterBefore(jwtAuthenticationFilter(),
                           UsernamePasswordAuthenticationFilter.class)

            // 异常处理
            .exceptionHandling(exception -> exception
                .authenticationEntryPoint((request, response, authException) -> {
                    response.setStatus(401);
                    response.setContentType("application/json");
                    response.getWriter().write("{\"code\":401,\"message\":\"未授权\"}");
                })
                .accessDeniedHandler((request, response, accessDeniedException) -> {
                    response.setStatus(403);
                    response.setContentType("application/json");
                    response.getWriter().write("{\"code\":403,\"message\":\"无权限\"}");
                })
            )

            .build();
    }

    @Bean
    public CorsConfigurationSource corsConfigurationSource() {
        CorsConfiguration configuration = new CorsConfiguration();
        configuration.setAllowedOriginPatterns(Arrays.asList("*"));
        configuration.setAllowedMethods(Arrays.asList("GET", "POST", "PUT", "DELETE"));
        configuration.setAllowedHeaders(Arrays.asList("*"));
        configuration.setAllowCredentials(true);

        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/**", configuration);
        return source;
    }
}
```

### 方法级权限控制

```java
@Service
public class UserService {

    @PreAuthorize("hasRole('ADMIN')")
    public void deleteUser(Long userId) {
        // 仅管理员可删除用户
    }

    @PreAuthorize("#userId == authentication.principal.id or hasRole('ADMIN')")
    public User getUserProfile(Long userId) {
        // 只能访问自己的信息或管理员
    }

    @PreAuthorize("hasAuthority('user:read')")
    public List<User> listUsers() {
        // 需要特定权限
    }

    @PostFilter("filterObject.email == authentication.principal.username or hasRole('ADMIN')")
    public List<User> findAllUsers() {
        // 过滤返回结果
    }
}
```

## SQL 注入防护

### MyBatis 参数化查询

```java
// ❌ 危险：SQL 拼接（禁止！）
@Select("SELECT * FROM t_user WHERE username = '${username}'")
User findByUsername(String username);

// ❌ 危险：字符串拼接
String sql = "SELECT * FROM t_user WHERE username = '" + username + "'";

// ✓ 正确：参数化查询（#{}）
@Select("SELECT * FROM t_user WHERE username = #{username}")
User findByUsername(String username);

// ✓ 推荐：MyBatis-Plus LambdaQueryWrapper（类型安全）
public User findByUsername(String username) {
    LambdaQueryWrapper<User> wrapper = new LambdaQueryWrapper<>();
    wrapper.eq(User::getUsername, username);
    return selectOne(wrapper);
}

// ✓ 注意：${} 用于表名/列名（谨慎使用）
@Select("SELECT * FROM ${tableName} WHERE id = #{id}")
User findByIdFromTable(@Param("tableName") String tableName, @Param("id") Long id);
```

### 动态表名验证

```java
public List<User> findByTable(String tableName) {
    // 白名单验证
    Set<String> allowedTables = Set.of("t_user", "t_admin", "t_guest");
    if (!allowedTables.contains(tableName)) {
        throw new SecurityException("非法表名");
    }

    LambdaQueryWrapper<User> wrapper = new LambdaQueryWrapper<>();
    // 安全使用
    return baseMapper.selectList(wrapper);
}
```

## XSS 防护

### 输入过滤

```java
@Configuration
public class XssFilterConfig {

    @Bean
    public FilterRegistrationBean<XssFilter> xssFilter() {
        FilterRegistrationBean<XssFilter> registration = new FilterRegistrationBean<>();
        registration.setFilter(new XssFilter());
        registration.addUrlPatterns("/*");
        registration.setName("xssFilter");
        registration.setOrder(1);
        return registration;
    }
}

// XSS Filter 实现
public class XssFilter implements Filter {

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) {
        XssHttpServletRequestWrapper wrappedRequest = new XssHttpServletRequestWrapper((HttpServletRequest) request);
        chain.doFilter(wrappedRequest, response);
    }
}

public class XssHttpServletRequestWrapper extends HttpServletRequestWrapper {

    @Override
    public String getParameter(String name) {
        String value = super.getParameter(name);
        return cleanXss(value);
    }

    @Override
    public String[] getParameterValues(String name) {
        String[] values = super.getParameterValues(name);
        if (values == null) return null;
        String[] cleanValues = new String[values.length];
        for (int i = 0; i < values.length; i++) {
            cleanValues[i] = cleanXss(values[i]);
        }
        return cleanValues;
    }

    private String cleanXss(String value) {
        if (value == null) return null;
        // 推荐使用 OWASP Java Encoder
        // Maven 依赖: org.webjars.npm:owasp-java-html-sanitizer
        return Encode.forHtml(value);
    }
}
```

### 输出转义

```java
// 在 VO 层自动转义
@Data
public class UserRespVO {
    private Long id;
    private String username;
    private String bio;  // 用户输入的富文本

    public String getBio() {
        // OWASP Java Encoder
        return Encode.forHtml(bio);
    }
}
```

## CSRF 防护

```java
// 启用 CSRF（Web 应用）
@Configuration
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        return http
            .csrf(csrf -> csrf
                .csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse())
            )
            // ...
            .build();
    }
}
```

## 接口限流

### Redis 限流实现

```java
@Component
public class RateLimitFilter implements Filter {

    private final RedisTemplate<String, String> redisTemplate;
    private static final String RATE_LIMIT_KEY = "rate_limit:";

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) {
        HttpServletRequest req = (HttpServletRequest) request;
        String ip = getClientIp(req);
        String key = RATE_LIMIT_KEY + ip;

        // 计数器
        Long count = redisTemplate.opsForValue().increment(key);
        if (count == 1) {
            redisTemplate.expire(key, 1, TimeUnit.MINUTES);
        }

        // 限流规则
        if (count > 100) {  // 每分钟100次
            throw new RateLimitException("访问过于频繁");
        }

        chain.doFilter(request, response);
    }

    private String getClientIp(HttpServletRequest request) {
        String ip = request.getHeader("X-Forwarded-For");
        if (ip == null || ip.isEmpty()) {
            ip = request.getRemoteAddr();
        }
        return ip;
    }
}
```

### 注解式限流

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface RateLimit {
    int limit() default 100;
    int period() default 60;  // 秒
}

@Aspect
@Component
public class RateLimitAspect {

    @Around("@annotation(rateLimit)")
    public Object around(ProceedingJoinPoint point, RateLimit rateLimit) throws Throwable {
        String key = generateKey(point);
        // 实现限流逻辑
        // ...
        return point.proceed();
    }
}

// 使用
@GetMapping("/sensitive-api")
@RateLimit(limit = 10, period = 60)
public Result<Void> sensitiveApi() {
    // ...
}
```

## 敏感数据处理

### 日志脱敏

```java
@Slf4j
public class UserService {

    public void createUser(UserCreateReq req) {
        log.info("创建用户: username={}, email={}",
            req.getUsername(),
            maskEmail(req.getEmail()));
    }

    private String maskEmail(String email) {
        if (email == null) return null;
        int at = email.indexOf('@');
        return email.charAt(0) + "***" + email.substring(at);
    }

    private String maskPhone(String phone) {
        if (phone == null) return null;
        return phone.replaceAll("(\\d{3})\\d{4}(\\d{4})", "$1****$2");
    }
}
```

### 响应数据脱敏

```java
@Data
public class UserRespVO {
    private Long id;
    private String username;
    private String phone;
    private String idCard;

    public String getPhone() {
        return phone != null ? phone.replaceAll("(\\d{3})\\d{4}(\\d{4})", "$1****$2") : null;
    }

    public String getIdCard() {
        return idCard != null ? idCard.replaceAll("(\\d{6})\\d{8}(\\d{4})", "$1********$2") : null;
    }
}
```

## 依赖供应链安全

### 依赖签名验证

```xml
<!-- pom.xml: 启用依赖验证 -->
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-enforcer-plugin</artifactId>
    <version>3.4.1</version>
    <executions>
        <execution>
            <id>enforce-dependency-convergence</id>
            <goals>
                <goal>enforce</goal>
            </goals>
            <configuration>
                <rules>
                    <dependencyConvergence/>
                    <requirePluginDeps>
                        <message>最佳实践：始终声明插件依赖</message>
                    </requirePluginDeps>
                </rules>
            </configuration>
        </execution>
    </executions>
</plugin>
```

### 依赖版本管理

```bash
# 使用 Maven BOM 统一管理版本
# dependency-bom/pom.xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-dependencies</artifactId>
            <version>3.2.0</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

### 依赖安全扫描

### OWASP Dependency-Check

```xml
<!-- pom.xml -->
<plugin>
    <groupId>org.owasp</groupId>
    <artifactId>dependency-check-maven</artifactId>
    <version>9.0.0</version>
    <executions>
        <execution>
            <goals>
                <goal>check</goal>
            </goals>
        </execution>
    </executions>
    <configuration>
        <failBuildOnCVSS>8</failBuildOnCVSS>
        <suppressionFiles>
            <suppressionFile>dependency-check-suppressions.xml</suppressionFile>
        </suppressionFiles>
    </configuration>
</plugin>
```

### Snyk 依赖扫描

```bash
# 安装 Snyk
npm install -g snyk

# 扫描 Maven 项目
snyk test --file=pom.xml

# 监控项目
snyk monitor
```

### Maven 依赖检查

```bash
# 查看依赖树
mvn dependency:tree

# 检查过时依赖
mvn versions:display-dependency-updates

# 检查插件更新
mvn versions:display-plugin-updates
```

## 安全响应协议

### 问题分类

| 级别 | CVSS | 示例 | 响应时间 |
|------|------|------|----------|
| **严重** | 9.0-10.0 | SQL 注入、RCE | 立即（1小时内） |
| **高** | 7.0-8.9 | 认证绕过、敏感数据泄露 | 当天（24小时内） |
| **中** | 4.0-6.9 | XSS、CSRF | 本周内（7天内） |
| **低** | 0.1-3.9 | 信息泄露、配置问题 | 下个迭代 |

### 响应流程

```
1. 立即停止相关代码变更
2. 识别并分类问题
3. 评估影响范围
4. 修复关键问题后再继续
5. 轮换任何泄露的密钥
6. 使用 Grep 搜索整个代码库查找类似问题
7. 编写安全事件报告
8. 更新安全检查清单
```

### 全库安全扫描

```bash
# 搜索硬编码密钥模式
grep -rE "(password|secret|key|token)\s*=\s*['\"]\w+" --include="*.java" .

# 搜索 SQL 拼接
grep -rE "(\+.*?(username|password|id))" --include="*.java" .

# 搜索敏感配置
grep -rE "(api[_-]?key|access[_-]?token)\s*:" --include="*.yml" .
```

## 安全审查清单

提交代码前，使用 **java-reviewer** Agent 进行主动安全审查：

| 检查项 | 说明 |
|--------|------|
| **密钥管理** | 无硬编码密钥，使用配置中心 |
| **输入验证** | 所有用户输入已校验 |
| **SQL 注入** | 使用参数化查询 |
| **XSS 防护** | 输入过滤 + 输出转义 |
| **CSRF 防护** | 已启用 CSRF 保护 |
| **认证授权** | Spring Security 配置正确 |
| **接口限流** | 所有 API 有限流保护 |
| **错误处理** | 不泄露敏感信息 |
| **数据加密** | 敏感数据加密存储 |
| **依赖漏洞** | 无已知高危漏洞 |
