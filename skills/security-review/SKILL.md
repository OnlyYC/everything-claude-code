---
name: security-review
description: Use this skill when adding authentication, handling user input, working with secrets, creating API endpoints, or implementing payment/sensitive features. Provides comprehensive security checklist and patterns for Java Spring Boot applications.
---

# 安全性审查技能

此技能确保所有代码遵循安全性最佳实践并识别潜在漏洞。

## 技术栈

- **Java 21** - 使用现代 Java 特性
- **Spring Boot 3.2+** - 安全配置
- **Spring MVC** - Web 安全
- **Spring Security** - 认证授权
- **MyBatis-Plus** - SQL 安全
- **Maven** - 依赖安全

## 何时启用

- 实现认证或授权
- 处理用户输入或文件上传
- 创建新的 REST API 端点
- 处理密钥或凭证
- 实现支付功能
- 存储或传输敏感数据
- 集成第三方 API
- 配置 Spring Security

## 安全性检查清单

### 1. 密钥管理

#### 绝不这样做
```java
// 写死的密钥 - 永远不要这样做！
public class ApiClient {
    private static final String API_KEY = "sk-proj-xxxxx";
    private static final String DB_PASSWORD = "password123";
}
```

#### 总是这样做
```java
// application.yml
spring:
  datasource:
    url: ${DB_URL:jdbc:mysql://localhost:3306/mydb}
    username: ${DB_USERNAME:root}
    password: ${DB_PASSWORD}

# application-prod.yml
spring:
  datasource:
    password: ${DB_PASSWORD}  # 从环境变量读取

external:
  api:
    key: ${EXTERNAL_API_KEY}
    url: ${EXTERNAL_API_URL}
```

```java
@Configuration
@ConfigurationProperties(prefix = "external.api")
@Validated
public record ApiConfig(
    @NotBlank
    String key,

    @NotBlank
    @URL
    String url
) {
    // 启动时验证配置
    @PostConstruct
    public void validate() {
        if (key == null || key.startsWith("sk-proj-")) {
            throw new IllegalStateException("EXTERNAL_API_KEY not configured properly");
        }
    }
}

@Service
@RequiredArgsConstructor
public class ExternalApiService {
    private final ApiConfig apiConfig;

    public void callApi() {
        // 使用配置的密钥
        String response = webClient.get()
            .uri(apiConfig.url())
            .header("Authorization", "Bearer " + apiConfig.key())
            .retrieve()
            .bodyToMono(String.class)
            .block();
    }
}
```

#### 验证步骤
- [ ] 无写死的 API 密钥、Token 或密码
- [ ] 所有密钥在环境变量中
- [ ] `application-*.yml` 中的敏感值使用 `${VAR_NAME}` 占位符
- [ ] `.env` 或敏感配置在 .gitignore 中
- [ ] git 历史中无密钥（`git log -p | grep -i "sk-"`）
- [ ] 生产密钥在环境变量或密钥管理服务中
- [ ] 启动时验证必需的配置存在

### 2. 输入验证

#### 总是验证用户输入
```java
// DTO 使用 JSR-303 验证注解
public record UserCreateRequest(
    @NotBlank(message = "邮箱不能为空")
    @Email(message = "邮箱格式不正确")
    String email,

    @NotBlank(message = "用户名不能为空")
    @Size(min = 2, max = 50, message = "用户名长度必须在 2-50 之间")
    String username,

    @Pattern(regexp = "^(?=.*[A-Za-z])(?=.*\\d)[A-Za-z\\d@$!%*#?&]{8,}$",
             message = "密码必须至少 8 位，包含字母和数字")
    String password,

    @Min(value = 0, message = "年龄不能为负数")
    @Max(value = 150, message = "年龄不能超过 150")
    Integer age
) {}

@RestController
@RequestMapping("/api/users")
@Validated
public class UserController {

    @PostMapping
    public ResponseEntity<ApiResponse<UserDTO>> createUser(
        @Valid @RequestBody UserCreateRequest request
    ) {
        UserDTO user = userService.createUser(request);
        return ResponseEntity.ok(ApiResponse.success(user));
    }

    // 处理验证异常
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ApiResponse<Map<String, String>>> handleValidationException(
        MethodArgumentNotValidException ex
    ) {
        Map<String, String> errors = ex.getBindingResult()
            .getFieldErrors()
            .stream()
            .collect(Collectors.toMap(
                FieldError::getField,
                FieldError::getDefaultMessage
            ));
        return ResponseEntity.badRequest()
            .body(ApiResponse.error("验证失败", errors));
    }
}
```

#### 路径变量验证
```java
@GetMapping("/users/{id}")
public ResponseEntity<UserDTO> getUser(
    @PathVariable @Pattern(regexp = "\\d+", message = "ID 必须为数字") String id
) {
    UserDTO user = userService.getUserById(Long.valueOf(id));
    return ResponseEntity.ok(user);
}

@GetMapping("/products/{slug}")
public ResponseEntity<ProductDTO> getProduct(
    @PathVariable @Slug(message = "无效的 slug 格式") String slug
) {
    // slug 自定义验证注解
}
```

#### 查询参数验证
```java
@GetMapping("/users")
public ResponseEntity<Page<UserDTO>> searchUsers(
    @RequestParam(required = false) @Size(min = 2, max = 50) String keyword,
    @RequestParam(defaultValue = "0") @Min(0) int page,
    @RequestParam(defaultValue = "10") @Min(1) @Max(100) int size
) {
    Pageable pageable = PageRequest.of(page, size);
    return ResponseEntity.ok(userService.searchUsers(keyword, pageable));
}
```

#### 文件上传验证
```java
@RestController
@RequestMapping("/api/files")
public class FileUploadController {

    private static final long MAX_FILE_SIZE = 5 * 1024 * 1024; // 5MB
    private static final Set<String> ALLOWED_TYPES = Set.of(
        "image/jpeg", "image/png", "image/gif", "image/webp"
    );
    private static final Set<String> ALLOWED_EXTENSIONS = Set.of(
        ".jpg", ".jpeg", ".png", ".gif", ".webp"
    );

    @PostMapping
    public ResponseEntity<ApiResponse<String>> uploadFile(
        @RequestParam("file") MultipartFile file
    ) {
        // 文件存在检查
        if (file.isEmpty()) {
            throw new BadRequestException("文件不能为空");
        }

        // 大小检查
        if (file.getSize() > MAX_FILE_SIZE) {
            throw new BadRequestException("文件大小不能超过 5MB");
        }

        // 内容类型检查
        String contentType = file.getContentType();
        if (!ALLOWED_TYPES.contains(contentType)) {
            throw new BadRequestException("不支持的文件类型");
        }

        // 扩展名检查
        String filename = file.getOriginalFilename();
        String extension = filename.substring(filename.lastIndexOf(".")).toLowerCase();
        if (!ALLOWED_EXTENSIONS.contains(extension)) {
            throw new BadRequestException("不支持的文件扩展名");
        }

        // 魔数检查（真实文件类型）
        try (InputStream input = file.getInputStream()) {
            String detectedType = Files.probeContentType(
                Paths.get(Objects.requireNonNull(filename))
            );
            if (!ALLOWED_TYPES.contains(detectedType)) {
                throw new BadRequestException("文件内容与扩展名不匹配");
            }
        } catch (IOException e) {
            throw new ServerException("文件验证失败");
        }

        // 保存文件（使用随机文件名）
        String savedPath = fileService.saveFile(file);
        return ResponseEntity.ok(ApiResponse.success(savedPath));
    }
}
```

#### 验证步骤
- [ ] 所有用户输入使用 @Valid/@Validated 验证
- [ ] DTO 使用 JSR-303 注解（@NotNull, @Size, @Pattern 等）
- [ ] 文件上传受限（大小、类型、扩展名、魔数）
- [ ] 数据库查询使用参数化查询（#{}）
- [ ] 白名单验证（非黑名单）
- [ ] 错误消息不泄露系统内部信息

### 3. SQL 注入预防

#### 绝不拼接 SQL
```java
// 危险 - SQL 注入漏洞！
@Mapper
public interface UserMapper {
    @Select("SELECT * FROM users WHERE email = '${email}'")  // 永远不要用 ${}
    User findByEmail(String email);
}

// 同样危险
@Select("SELECT * FROM users WHERE name = '" + name + "'")
User findByName(String name);
```

#### 总是使用参数化查询
```java
// 安全 - MyBatis-Plus 使用 #{} 参数化
@Mapper
public interface UserMapper extends BaseMapper<User> {

    // 使用 #{} 进行参数绑定
    @Select("SELECT * FROM users WHERE email = #{email}")
    User findByEmail(@Param("email") String email);

    // 多条件查询
    @Select("SELECT * FROM users WHERE email = #{email} AND status = #{status}")
    Optional<User> findByEmailAndStatus(
        @Param("email") String email,
        @Param("status") UserStatus status
    );

    // IN 查询使用 foreach
    @Select("<script>" +
            "SELECT * FROM users WHERE id IN " +
            "<foreach item='id' collection='ids' open='(' separator=',' close=')'>" +
            "#{id}" +
            "</foreach>" +
            "</script>")
    List<User> findByIds(@Param("ids") List<Long> ids);

    // LIKE 查询需要手动拼接 %
    @Select("SELECT * FROM users WHERE name LIKE CONCAT('%', #{keyword}, '%')")
    List<User> searchByName(@Param("keyword") String keyword);
}
```

#### MyBatis-Plus 安全查询
```java
@Service
@RequiredArgsConstructor
public class UserService {

    private final UserMapper userMapper;

    // 使用 LambdaQueryWrapper - 类型安全
    public User findByEmail(String email) {
        return userMapper.selectOne(
            new LambdaQueryWrapper<User>()
                .eq(User::getEmail, email)
        );
    }

    // 动态条件
    public List<User> searchUsers(String keyword, UserStatus status) {
        return userMapper.selectList(
            new LambdaQueryWrapper<User>()
                .like(StringUtils.hasText(keyword), User::getName, keyword)
                .eq(status != null, User::getStatus, status)
        );
    }
}
```

#### MyBatis XML 安全写法
```xml
<!-- UserMapper.xml -->
<mapper namespace="com.example.mapper.UserMapper">

    <!-- 安全 - 使用 #{} -->
    <select id="findByEmail" resultType="User">
        SELECT id, email, username, status
        FROM users
        WHERE email = #{email}
    </select>

    <!-- 动态 SQL 使用 <if> -->
    <select id="searchUsers" resultType="User">
        SELECT id, email, username, status
        FROM users
        <where>
            <if test="keyword != null and keyword != ''">
                AND name LIKE CONCAT('%', #{keyword}, '%')
            </if>
            <if test="status != null">
                AND status = #{status}
            </if>
        </where>
        LIMIT #{offset}, #{limit}
    </select>

    <!-- IN 查询 -->
    <select id="findByIds" resultType="User">
        SELECT * FROM users
        WHERE id IN
        <foreach item="id" collection="ids" open="(" separator="," close=")">
            #{id}
        </foreach>
    </select>

</mapper>
```

#### 验证步骤
- [ ] 所有 SQL 使用 #{} 参数绑定（永远不用 ${} 传用户输入）
- [ ] LIKE 查询使用 CONCAT 拼接通配符
- [ ] IN 查询使用 foreach 标签
- [ ] MyBatis-Plus 使用 LambdaQueryWrapper
- [ ] 表名/列名（需要时）才使用 ${}，且来自白名单

### 4. 认证与授权

#### Spring Security 配置
```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            // CSRF 保护（状态ful API 需要）
            .csrf(csrf -> csrf
                .csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse())
            )
            // CORS 配置
            .cors(cors -> cors.configurationSource(corsConfigurationSource()))
            // Session 管理（无状态，使用 JWT）
            .sessionManagement(session -> session
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS)
            )
            // 异常处理
            .exceptionHandling(exception -> exception
                .authenticationEntryPoint((request, response, authException) -> {
                    response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
                    response.setContentType(MediaType.APPLICATION_JSON_VALUE);
                    response.getWriter().write("{\"error\":\"未认证\"}");
                })
                .accessDeniedHandler((request, response, accessDeniedException) -> {
                    response.setStatus(HttpServletResponse.SC_FORBIDDEN);
                    response.setContentType(MediaType.APPLICATION_JSON_VALUE);
                    response.getWriter().write("{\"error\":\"无权限\"}");
                })
            )
            // JWT 过滤器
            .addFilterBefore(jwtAuthenticationFilter,
                           UsernamePasswordAuthenticationFilter.class)
            // 授权规则
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/auth/**").permitAll()
                .requestMatchers("/api/public/**").permitAll()
                .requestMatchers(HttpMethod.GET, "/api/products/**").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .requestMatchers("/api/user/**").hasAnyRole("USER", "ADMIN")
                .anyRequest().authenticated()
            );

        return http.build();
    }

    @Bean
    public CorsConfigurationSource corsConfigurationSource() {
        CorsConfiguration configuration = new CorsConfiguration();
        configuration.setAllowedOrigins(List.of("https://example.com"));
        configuration.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE"));
        configuration.setAllowedHeaders(List.of("*"));
        configuration.setAllowCredentials(true);

        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/**", configuration);
        return source;
    }
}
```

#### JWT 工具类
```java
@Component
public class JwtTokenProvider {

    @Value("${jwt.secret}")
    private String jwtSecret;

    @Value("${jwt.expiration:86400}") // 默认 24 小时
    private long jwtExpiration;

    public String generateToken(Authentication authentication) {
        UserPrincipal userPrincipal = (UserPrincipal) authentication.getPrincipal();
        Date now = new Date();
        Date expiryDate = new Date(now.getTime() + jwtExpiration * 1000);

        return Jwts.builder()
            .subject(Long.toString(userPrincipal.getId()))
            .claim("email", userPrincipal.getEmail())
            .claim("roles", userPrincipal.getAuthorities())
            .issuedAt(now)
            .expiration(expiryDate)
            .signWith(Keys.hmacShaKeyFor(jwtSecret.getBytes()))
            .compact();
    }

    public Long getUserIdFromToken(String token) {
        Claims claims = Jwts.parser()
            .verifyWith(Keys.hmacShaKeyFor(jwtSecret.getBytes()))
            .build()
            .parseSignedClaims(token)
            .getPayload();

        return Long.parseLong(claims.getSubject());
    }

    public boolean validateToken(String token) {
        try {
            Jwts.parser()
                .verifyWith(Keys.hmacShaKeyFor(jwtSecret.getBytes()))
                .build()
                .parseSignedClaims(token);
            return true;
        } catch (JwtException | IllegalArgumentException e) {
            log.error("Invalid JWT token: {}", e.getMessage());
            return false;
        }
    }
}
```

#### 授权检查
```java
@RestController
@RequestMapping("/api/admin")
public class AdminController {

    // 方法级别授权
    @PreAuthorize("hasRole('ADMIN')")
    @DeleteMapping("/users/{id}")
    public ResponseEntity<Void> deleteUser(@PathVariable Long id) {
        adminService.deleteUser(id);
        return ResponseEntity.noContent().build();
    }

    // 自定义权限检查
    @PreAuthorize("@userService.canAccessUser(#id, authentication)")
    @GetMapping("/users/{id}")
    public ResponseEntity<UserDTO> getUser(@PathVariable Long id) {
        return ResponseEntity.ok(userService.getUserById(id));
    }
}

@Service
public class UserService {

    public boolean canAccessUser(Long userId, Authentication authentication) {
        UserPrincipal principal = (UserPrincipal) authentication.getPrincipal();
        // 管理员可以访问所有用户
        if (principal.getAuthorities().stream()
                .anyMatch(a -> a.getAuthority().equals("ROLE_ADMIN"))) {
            return true;
        }
        // 用户只能访问自己
        return principal.getId().equals(userId);
    }
}
```

#### 数据权限（Row Level Security）
```java
// 使用 MyBatis-Plus 拦截器实现
@Configuration
public class MybatisPlusConfig {

    @Bean
    public MybatisPlusInterceptor mybatisPlusInterceptor() {
        MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();

        // 数据权限插件
        interceptor.addInnerInterceptor(new DataPermissionInterceptor(
            new DataPermissionHandler() {
                @Override
                public void beforeQuery(Executor executor, MappedStatement ms,
                                       Object parameter, RowBounds rowBounds,
                                       ResultHandler resultHandler, BoundSql boundSql) {
                    // 获取当前用户
                    Long currentUserId = SecurityContextHolder.getCurrentUserId();

                    // 修改 SQL，添加数据权限条件
                    // WHERE tenant_id = #{currentUserId}
                }
            }
        ));

        return interceptor;
    }
}
```

#### 验证步骤
- [ ] JWT 存储在 HttpOnly Cookie 或内存（非 localStorage）
- [ ] 敏感操作前有授权检查（@PreAuthorize）
- [ ] 密码使用 BCrypt 加密存储
- [ ] Session 创建策略为 STATELESS
- [ ] CSRF 保护已启用（状态ful API）
- [ ] CORS 正确配置
- [ ] 数据权限过滤已实现

### 5. XSS 预防

#### 输入净化
```java
@Component
public class HtmlSanitizer {

    // 使用 jsoup 净化 HTML
    public String sanitize(String html) {
        if (html == null) return null;

        Whitelist whitelist = Whitelist.none()
            .addTags("b", "i", "em", "strong", "p", "br")
            .addAttributes("a", "href");

        return Jsoup.clean(html, whitelist);
    }

    // 更严格的净化
    public String sanitizeStrict(String html) {
        return Jsoup.clean(html, Whitelist.none());
    }
}
```

#### 输出编码
```java
@RestController
@RequestMapping("/api/content")
public class ContentController {

    @GetMapping("/{id}")
    public ResponseEntity<ContentDTO> getContent(@PathVariable Long id) {
        Content content = contentService.getById(id);

        // JSON 序列化会自动转义特殊字符
        // Jackson 默认处理 XSS 防护
        return ResponseEntity.ok(ContentDTO.from(content));
    }
}

// 确保敏感字段不序列化
public record UserDTO(
    Long id,
    String username,
    String email,
    @JsonIgnore  // 不序列化密码
    String password,

    @JsonInclude(JsonInclude.Include.NON_NULL)
    String phone
) {}
```

#### Spring Security Headers
```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http
        .headers(headers -> headers
            .contentSecurityPolicy(csp -> csp
                .policyDirectives("default-src 'self'; script-src 'self' 'unsafe-inline'; style-src 'self' 'unsafe-inline'; img-src 'self' data: https:")
            )
            .frameOptions(frame -> frame.deny())
            .xssProtection(xss -> xss.enable())
            .httpStrictTransportSecurity(hsts -> hsts
                .includeSubDomains(true)
                .maxAgeInSeconds(31536000)
            )
        );
    return http.build();
}
```

#### 验证步骤
- [ ] 用户输入的 HTML 已净化（使用 jsoup）
- [ ] JSON 响应使用 Jackson 自动转义
- [ ] CSP headers 已配置
- [ ] X-XSS-Protection header 已启用
- [ ] X-Frame-Options 设置为 DENY
- [ ] 敏感字段使用 @JsonIgnore

### 6. CSRF 保护

#### Spring Security CSRF（默认启用）
```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http
        .csrf(csrf -> csrf
            // 存储 token 在 cookie
            .csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse())
            // 排除 API 端点（如果是无状态 JWT）
            .ignoringRequestMatchers("/api/auth/**")
        );
    return http.build();
}
```

#### 前端 CSRF Token 处理
```java
@RestController
@RequestMapping("/api")
public class CsrfController {

    @GetMapping("/csrf")
    public CsrfToken csrf(CsrfToken token) {
        // Spring 自动提供 CsrfToken
        return token;
    }
}
```

#### SameSite Cookies
```yaml
# application.yml
server:
  servlet:
    session:
      cookie:
        http-only: true
        secure: true
        same-site: strict
```

#### 验证步骤
- [ ] 状态变更操作有 CSRF 保护
- [ ] Cookies 设置 SameSite=Strict
- [ ] API 端点根据架构决定是否需要 CSRF
- [ ] 前端正确处理 CSRF token

### 7. 速率限制

#### 使用 Bucket4j
```xml
<dependency>
    <groupId>com.github.vladimir-bukhtoyarov</groupId>
    <artifactId>bucket4j-core</artifactId>
    <version>8.7.0</version>
</dependency>
```

```java
@Component
public class RateLimitingFilter extends OncePerRequestFilter {

    private final Map<String, Bucket> buckets = new ConcurrentHashMap<>();

    private Bucket createBucket() {
        return Bucket.builder()
            .addLimit(Bandwidth.classic(100, Refill.intervally(100, Duration.ofMinutes(15))))
            .build();
    }

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                   HttpServletResponse response,
                                   FilterChain filterChain) throws ServletException, IOException {
        String key = getClientIP(request);
        Bucket bucket = buckets.computeIfAbsent(key, k -> createBucket());

        ConsumptionProbe probe = bucket.tryConsumeAndReturnRemaining(1);
        if (!probe.isConsumed()) {
            response.setStatus(HttpServletResponse.SC_TOO_MANY_REQUESTS);
            response.setHeader("X-Rate-Limit-Remaining", String.valueOf(probe.getRemainingTokens()));
            response.getWriter().write("Too many requests");
            return;
        }

        response.setHeader("X-Rate-Limit-Remaining", String.valueOf(probe.getRemainingTokens()));
        filterChain.doFilter(request, response);
    }

    private String getClientIP(HttpServletRequest request) {
        String xfHeader = request.getHeader("X-Forwarded-For");
        if (xfHeader == null) {
            return request.getRemoteAddr();
        }
        return xfHeader.split(",")[0];
    }
}
```

#### 注解式速率限制
```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface RateLimit {
    int capacity() default 100;
    int refill() default 100;
    Duration duration() default Duration.ofMinutes(15);
}

@Aspect
@Component
public class RateLimitAspect {

    private final Map<String, Bucket> buckets = new ConcurrentHashMap<>();

    @Around("@annotation(rateLimit)")
    public Object rateLimit(ProceedingJoinPoint joinPoint, RateLimit rateLimit) throws Throwable {
        HttpServletRequest request = ((ServletRequestAttributes) RequestContextHolder.currentRequestAttributes()).getRequest();
        String key = request.getRequestURI() + ":" + getClientIP(request);

        Bucket bucket = buckets.computeIfAbsent(key, k ->
            Bucket.builder()
                .addLimit(Bandwidth.classic(rateLimit.refill(),
                    Refill.intervally(rateLimit.refill(), rateLimit.duration())))
                .build()
        );

        if (bucket.tryConsume(1)) {
            return joinPoint.proceed();
        }

        throw new ResponseStatusException(HttpStatus.TOO_MANY_REQUESTS, "Too many requests");
    }
}

// 使用
@RestController
public class SearchController {

    @GetMapping("/search")
    @RateLimit(capacity = 10, refill = 10, duration = Duration.ofMinutes(1))
    public ResponseEntity<List<Result>> search(@RequestParam String q) {
        return ResponseEntity.ok(searchService.search(q));
    }
}
```

#### 验证步骤
- [ ] 所有 API 端点有速率限制
- [ ] 昂贵操作有更严格限制
- [ ] 基于 IP 的速率限制
- [ ] 基于 API Key 的速率限制（已认证用户）
- [ ] 响应包含 X-Rate-Limit-* headers

### 8. 敏感数据暴露

#### 日志配置
```xml
<!-- logback-spring.xml -->
<configuration>
    <!-- 不要记录敏感数据 -->
    <logger name="com.example.filter" level="INFO"/>

    <!-- 生产环境不记录 DEBUG -->
    <springProfile name="prod">
        <root level="INFO"/>
    </springProfile>
</configuration>
```

```java
@Slf4j
@Service
public class UserService {

    // 错误：记录敏感数据
    public void badLogging(User user) {
        log.info("User created: {}", user); // 可能包含密码
    }

    // 正确：只记录必要信息
    public void goodLogging(User user) {
        log.info("User created: id={}, email={}", user.getId(), user.getEmail());
    }
}
```

#### 敏感数据脱敏
```java
@JsonSerialize(using = SensitiveDataSerializer.class)
public record SensitiveData(String value) {
    public static SensitiveData of(String value) {
        return new SensitiveData(value);
    }
}

public class SensitiveDataSerializer extends JsonSerializer<SensitiveData> {
    @Override
    public void serialize(SensitiveData value, JsonGenerator gen, SerializerProvider serializers) {
        String str = value.value();
        if (str == null) {
            gen.writeNull();
        } else if (str.length() <= 4) {
            gen.writeString("****");
        } else {
            gen.writeString(str.substring(0, 2) + "****" + str.substring(str.length() - 2));
        }
    }
}

// 使用
public record UserDTO(
    Long id,
    String email,
    @JsonProperty(access = JsonProperty.Access.WRITE_ONLY)
    String password,
    SensitiveData phone,
    SensitiveData idCard
) {}
```

#### 异常处理
```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    // 生产环境不返回堆栈追踪
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ApiResponse<Void>> handleException(Exception ex) {
        log.error("Unexpected error", ex);  // 服务器日志记录详细错误

        // 用户只看到通用消息
        return ResponseEntity.status(500)
            .body(ApiResponse.error("服务器内部错误，请稍后重试"));
    }

    @ExceptionHandler(AccessDeniedException.class)
    public ResponseEntity<ApiResponse<Void>> handleAccessDenied(AccessDeniedException ex) {
        log.warn("Access denied: {}", ex.getMessage());
        return ResponseEntity.status(403)
            .body(ApiResponse.error("无权限访问此资源"));
    }
}
```

#### 验证步骤
- [ ] 日志中无密码、token 或密钥
- [ ] 用户收到通用错误消息
- [ ] 详细错误只在服务器日志
- [ ] 不向用户暴露堆栈追踪
- [ ] 敏感字段使用 @JsonProperty(access = WRITE_ONLY)
- [ ] 生产环境日志级别为 INFO 或更高

### 9. 支付安全

#### 支付验证
```java
@Service
@RequiredArgsConstructor
public class PaymentService {

    private final PaymentGatewayClient paymentGateway;

    @Transactional
    public PaymentResult processPayment(PaymentRequest request) {
        // 验证金额
        if (request.amount().compareTo(BigDecimal.ZERO) <= 0) {
            throw new InvalidPaymentException("金额必须大于 0");
        }

        if (request.amount().compareTo(new BigDecimal("100000")) > 0) {
            throw new InvalidPaymentException("单笔金额不能超过 100000");
        }

        // 验证用户
        User user = userService.getById(request.userId());
        if (user.getStatus() != UserStatus.ACTIVE) {
            throw new InvalidPaymentException("用户状态异常");
        }

        // 验证订单
        Order order = orderService.getById(request.orderId());
        if (order.getUserId() != request.userId()) {
            throw new InvalidPaymentException("订单不属于当前用户");
        }

        // 防止重复支付
        if (order.getStatus() == OrderStatus.PAID) {
            throw new InvalidPaymentException("订单已支付");
        }

        // 调用支付网关
        PaymentResult result = paymentGateway.charge(request);

        // 记录审计日志
        auditService.logPayment(request, result);

        return result;
    }
}
```

#### 验证步骤
- [ ] 金额范围验证
- [ ] 用户和订单关联验证
- [ ] 防止重复支付
- [ ] 支付失败重试限制
- [ ] 完整的审计日志
- [ ] PCI DSS 合规（如果处理卡信息）

### 10. 依赖安全

#### OWASP Dependency Check
```xml
<plugin>
    <groupId>org.owasp</groupId>
    <artifactId>dependency-check-maven</artifactId>
    <version>9.0.1</version>
    <configuration>
        <failBuildOnCVSS>7</failBuildOnCVSS>
        <suppressionFile>dependency-check-suppressions.xml</suppressionFile>
    </configuration>
    <executions>
        <execution>
            <goals>
                <goal>check</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```

#### 定期更新
```bash
# 检查过时的依赖
mvn versions:display-dependency-updates

# 检查漏洞
mvn org.owasp:dependency-check-maven:check

# 更新依赖
mvn versions:use-latest-versions
```

#### 验证步骤
- [ ] 依赖保持最新
- [ ] 无已知高危漏洞（CVSS >= 7）
- [ ] pom.xml 已 commit
- [ ] GitHub Dependabot 已启用
- [ ] 定期执行依赖更新

## 安全测试

### 自动化安全测试
```java
@SpringBootTest
@AutoConfigureMockMvc
class SecurityTest {

    @Autowired
    private MockMvc mockMvc;

    // 测试未认证访问
    @Test
    void requiresAuthentication() throws Exception {
        mockMvc.perform(get("/api/user/profile"))
            .andExpect(status().isUnauthorized());
    }

    // 测试授权
    @Test
    @WithMockUser(roles = "USER")
    void requiresAdminRole() throws Exception {
        mockMvc.perform(delete("/api/admin/users/1"))
            .andExpect(status().isForbidden());
    }

    // 测试输入验证
    @Test
    void rejectsInvalidInput() throws Exception {
        String invalidBody = """
            {
                "email": "not-an-email",
                "password": "123"
            }
            """;

        mockMvc.perform(post("/api/users")
                .contentType(MediaType.APPLICATION_JSON)
                .content(invalidBody))
            .andExpect(status().isBadRequest());
    }

    // 测试 SQL 注入防护
    @Test
    void preventsSQLInjection() throws Exception {
        String maliciousInput = "admin' OR '1'='1";

        mockMvc.perform(get("/api/users")
                .param("email", maliciousInput))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.data").isEmpty());
    }

    // 测试 XSS 防护
    @Test
    void sanitizesXSS() throws Exception {
        String xssPayload = "<script>alert('xss')</script>";

        mockMvc.perform(post("/api/content")
                .contentType(MediaType.APPLICATION_JSON)
                .content("{\"content\":\"" + xssPayload + "\"}"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.data.content").value(not(containsString("<script>"))));
    }
}
```

## 部署前安全检查清单

任何生产部署前：

- [ ] **密钥**：无写死密钥，全在环境变量中
- [ ] **输入验证**：所有用户输入已验证（@Valid）
- [ ] **SQL 注入**：所有查询使用 #{} 参数化
- [ ] **XSS**：用户输入已净化，CSP 已配置
- [ ] **CSRF**：保护已启用（状态ful API）
- [ ] **认证**：JWT 正确配置，密码 BCrypt 加密
- [ ] **授权**：@PreAuthorize 检查已就位
- [ ] **速率限制**：所有端点已启用
- [ ] **HTTPS**：生产环境强制使用，HSTS 已启用
- [ ] **安全标头**：CSP、X-Frame-Options、X-XSS-Protection
- [ ] **错误处理**：错误中无敏感数据
- [ ] **日志记录**：无敏感数据，生产级别 INFO
- [ ] **依赖**：最新，无高危漏洞
- [ ] **CORS**：正确配置白名单
- [ ] **文件上传**：已验证（大小、类型、魔数）
- [ ] **密码策略**：强度要求、加密存储

## 资源

- [OWASP Top 10](https://owasp.org/www-project-top-ten/)
- [Spring Security Reference](https://docs.spring.io/spring-security/reference/)
- [MyBatis Security](https://mybatis.org/mybatis-3/sqlmap-xml.html#Parameters)
- [Web Security Academy](https://portswigger.net/web-security)

---

**记住**：安全性不是可选的。一个漏洞可能危及整个平台。有疑虑时，选择谨慎的做法。
