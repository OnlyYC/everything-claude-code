# 安全指南

## 强制安全检查

任何提交前：
- [ ] 没有硬编码的密钥（API 密钥、密码、Token）
- [ ] 所有用户输入已校验
- [ ] SQL 注入防护（MyBatis 参数化查询）
- [ ] XSS 防护（输出转义）
- [ ] 已启用 CSRF 保护
- [ ] 已验证认证/授权
- [ ] 所有接口都有限流控制
- [ ] 错误信息不泄露敏感数据

## 密钥管理

```java
// 绝不：硬编码密钥
public class PayService {
    private static final String API_KEY = "sk-proj-xxxxx";  // 危险！
}

// 推荐：使用配置中心
@Configuration
public class PayConfig {
    @Value("${pay.api-key}")
    private String apiKey;

    @Value("${pay.api-secret}")
    private String apiSecret;

    // 或使用 @ConfigurationProperties
    @Bean
    public PayClient payClient() {
        if (StringUtils.isBlank(apiKey)) {
            throw new IllegalStateException("pay.api-key 未配置");
        }
        return new PayClient(apiKey, apiSecret);
    }
}

// 推荐：使用 Nacos/Apollo 配置中心
@RefreshScope
@Configuration
public class DynamicConfig {
    @NacosValue(value = "${pay.api-key}", autoRefreshed = true)
    private String apiKey;
}
```

## SQL 注入防护（MyBatis）

```java
// 危险：SQL 拼接
@Select("SELECT * FROM t_user WHERE username = '${username}'")
User findByUsername(String username);  // 禁止！

// 正确：参数化查询
@Select("SELECT * FROM t_user WHERE username = #{username}")
User findByUsername(String username);

// 推荐：使用 MyBatis-Plus LambdaQueryWrapper
public User findByUsername(String username) {
    LambdaQueryWrapper<User> wrapper = new LambdaQueryWrapper<>();
    wrapper.eq(User::getUsername, username);
    return selectOne(wrapper);
}
```


## XSS 防护

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

// 使用工具类转义输出
public class HtmlUtils {
    public static String escape(String input) {
        return StringEscapeUtils.escapeHtml4(input);
    }
}
```

## 接口限流

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

        Long count = redisTemplate.opsForValue().increment(key);
        if (count == 1) {
            redisTemplate.expire(key, 1, TimeUnit.MINUTES);
        }
        if (count > 100) {  // 每分钟100次
            throw new RateLimitException("访问过于频繁");
        }

        chain.doFilter(request, response);
    }
}
```

## 安全响应协议

如果发现安全问题：
1. 立即停止
2. 使用 **security-reviewer** Agent
3. 在继续前修复关键问题
4. 轮换任何泄露的密钥
5. 审查整个代码库是否存在类似问题
