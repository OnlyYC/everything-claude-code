---
name: security
description: Java/Spring Boot 安全指南
priority: must
tags: [java, spring-boot, security, spring-security, jwt]
tech-stack: [Spring Security, JWT, OWASP]
---

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

### 禁止硬编码

```java
// ❌ 绝对禁止
public class PayService {
    private static final String API_KEY = "sk-proj-xxxxx";
}

// ✓ 推荐：使用配置中心
@Configuration
public class PayConfig {
    @Value("${pay.api-key}")
    private String apiKey;

    @Bean
    public PayClient payClient() {
        if (StringUtils.isBlank(apiKey)) {
            throw new IllegalStateException("pay.api-key 未配置");
        }
        return new PayClient(apiKey);
    }
}

// ✓ 推荐：使用环境变量
@Value("${PAY_API_KEY:#{null}}")
private String apiKey;
```

### 密钥配置隔离

```bash
config/
├── application-dev.yml      # 开发环境
├── application-test.yml     # 测试环境
├── application-prod.yml     # 生产环境（从外部注入）
└── .gitignore               # 忽略敏感配置
```

## Spring Security 配置

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity(prePostEnabled = true)
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        return http
            .csrf(csrf -> csrf.disable())  // API 项目禁用 CSRF
            .cors(cors -> cors.configurationSource(corsConfigurationSource()))
            .sessionManagement(session -> session
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS)
            )
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/public/**").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            )
            .addFilterBefore(jwtAuthenticationFilter(),
                           UsernamePasswordAuthenticationFilter.class)
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
}
```

## SQL 注入防护

```java
// ❌ 危险：SQL 拼接（禁止！）
@Select("SELECT * FROM t_user WHERE username = '${username}'")
User findByUsername(String username);

// ✓ 正确：参数化查询（#{}）
@Select("SELECT * FROM t_user WHERE username = #{username}")
User findByUsername(String username);

// ✓ 推荐：MyBatis-Plus LambdaQueryWrapper（类型安全）
public User findByUsername(String username) {
    LambdaQueryWrapper<User> wrapper = new LambdaQueryWrapper<>();
    wrapper.eq(User::getUsername, username);
    return selectOne(wrapper);
}

// ✓ 注意：${} 用于表名/列名（谨慎使用，需白名单验证）
@Select("SELECT * FROM ${tableName} WHERE id = #{id}")
User findByIdFromTable(@Param("tableName") String tableName, @Param("id") Long id);
```

## XSS 防护

### 输入过滤 + 输出转义

```java
// 推荐：使用 OWASP Java Encoder
// Maven 依赖: org.webjars.npm:owasp-java-html-sanitizer

public class XssHttpServletRequestWrapper extends HttpServletRequestWrapper {

    @Override
    public String getParameter(String name) {
        String value = super.getParameter(name);
        return cleanXss(value);
    }

    private String cleanXss(String value) {
        if (value == null) return null;
        return Encode.forHtml(value);
    }
}

// 在 VO 层自动转义
@Data
public class UserRespVO {
    private String bio;

    public String getBio() {
        return Encode.forHtml(bio);
    }
}
```

## CSRF 防护

```java
// Web 应用启用 CSRF
@Configuration
public class SecurityConfig {
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        return http
            .csrf(csrf -> csrf
                .csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse())
            )
            .build();
    }
}
```

## 接口限流

```java
@Component
public class RateLimitFilter implements Filter {

    private final RedisTemplate<String, String> redisTemplate;

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) {
        HttpServletRequest req = (HttpServletRequest) request;
        String ip = getClientIp(req);
        String key = "rate_limit:" + ip;

        Long count = redisTemplate.opsForValue().increment(key);
        if (count == 1) {
            redisTemplate.expire(key, 1, TimeUnit.MINUTES);
        }

        if (count > 100) {
            throw new RateLimitException("访问过于频繁");
        }
        chain.doFilter(request, response);
    }
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
    private String phone;
    private String idCard;

    public String getPhone() {
        return phone != null ?
            phone.replaceAll("(\\d{3})\\d{4}(\\d{4})", "$1****$2") : null;
    }

    public String getIdCard() {
        return idCard != null ?
            idCard.replaceAll("(\\d{6})\\d{8}(\\d{4})", "$1********$2") : null;
    }
}
```

## 依赖安全

### 依赖扫描

```bash
# OWASP Dependency-Check
mvn org.owasp:dependency-check-maven:check

# Snyk 扫描
snyk test --file=pom.xml

# Maven 依赖检查
mvn dependency:tree
mvn versions:display-dependency-updates
```

### 依赖签名验证

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-enforcer-plugin</artifactId>
    <version>3.4.1</version>
    <executions>
        <execution>
            <id>enforce-dependency-convergence</id>
            <goals><goal>enforce</goal></goals>
        </execution>
    </executions>
</plugin>
```

## HTTPS 强制

```java
@Configuration
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        return http
            .requiresChannel(channel -> channel
                .requestMatchers(r -> r.getRequestURI().startsWith("/api"))
                .requiresSecure()
            )
            .build();
    }
}
```

```yaml
# application-prod.yml
server:
  ssl:
    enabled: true
    key-store: classpath:keystore.p12
    key-store-password: ${SSL_KEY_STORE_PASSWORD}
  port: 8443
```

## JWT Token 管理

```java
@Service
@RequiredArgsConstructor
public class JwtTokenService {

    private final RedisTemplate<String, String> redisTemplate;
    private static final long ACCESS_TOKEN_EXPIRATION = 3600;
    private static final long REFRESH_TOKEN_EXPIRATION = 604800;

    public String generateAccessToken(String username) {
        Date expiryDate = new Date(System.currentTimeMillis() + ACCESS_TOKEN_EXPIRATION * 1000);
        return Jwts.builder()
                .setSubject(username)
                .setExpiration(expiryDate)
                .signWith(getSigningKey())
                .compact();
    }

    public void revokeToken(String username) {
        String key = "refresh_token:" + username;
        redisTemplate.delete(key);
    }
}
```

## 安全响应协议

| 级别 | CVSS | 示例 | 响应时间 |
|------|------|------|----------|
| **严重** | 9.0-10.0 | SQL 注入、RCE | 立即（1小时内） |
| **高** | 7.0-8.9 | 认证绕过、敏感数据泄露 | 当天（24小时内） |
| **中** | 4.0-6.9 | XSS、CSRF | 本周内（7天内） |
| **低** | 0.1-3.9 | 信息泄露 | 下个迭代 |

```bash
# 全库安全扫描
grep -rE "(password|secret|key|token)\s*=\s*['\"]\w+" --include="*.java" .
grep -rE "api[_-]?key|access[_-]?token" --include="*.yml" .
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
