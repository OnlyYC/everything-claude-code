# 测试规范

## 最低测试覆盖率：80%

测试类型（全部必需）：
1. **单元测试** - 单个方法、工具类、Service 层
2. **集成测试** - API 接口、数据库操作
3. **E2E 测试** - 关键用户流程（RestAssured）

## 单元测试（JUnit 5 + Mockito）

```java
@ExtendWith(MockitoExtension.class)
class UserServiceTest {

    @Mock
    private UserMapper userMapper;

    @InjectMocks
    private UserServiceImpl userService;

    @Test
    @DisplayName("根据ID查询用户 - 成功")
    void getUserById_Success() {
        // Given
        Long userId = 1L;
        User user = new User(userId, "张三", "zhangsan@example.com");
        when(userMapper.selectById(userId)).thenReturn(user);

        // When
        User result = userService.getById(userId);

        // Then
        assertNotNull(result);
        assertEquals("张三", result.getUsername());
        verify(userMapper, times(1)).selectById(userId);
    }

    @Test
    @DisplayName("根据ID查询用户 - 不存在")
    void getUserById_NotFound() {
        // Given
        Long userId = 999L;
        when(userMapper.selectById(userId)).thenReturn(null);

        // When & Then
        assertThrows(BusinessException.class, () -> userService.getById(userId));
    }
}
```

## 集成测试（@SpringBootTest）

```java
@SpringBootTest
@AutoConfigureMockMvc
@Transactional
class UserControllerIntegrationTest {

    @Autowired
    private MockMvc mockMvc;

    @Autowired
    private ObjectMapper objectMapper;

    @Test
    @DisplayName("创建用户 - 成功")
    void createUser_Success() throws Exception {
        // Given
        UserCreateReq req = new UserCreateReq("张三", "zhangsan@example.com");

        // When & Then
        mockMvc.perform(post("/api/users")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(req)))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.code").value(200))
                .andExpect(jsonPath("$.message").value("success"));
    }
}
```

## MyBatis 测试

```java
@DataMybatisTest
class UserMapperTest {

    @Autowired
    private UserMapper userMapper;

    @Test
    @DisplayName("插入用户")
    void insertUser() {
        // Given
        User user = new User(null, "李四", "lisi@example.com");

        // When
        int rows = userMapper.insert(user);

        // Then
        assertEquals(1, rows);
        assertNotNull(user.getId());
    }

    @Test
    @DisplayName("根据用户名查询")
    void selectByUsername() {
        // Given
        String username = "王五";

        // When
        User user = userMapper.selectOne(
            new LambdaQueryWrapper<User>()
                .eq(User::getUsername, username)
        );

        // Then
        assertNotNull(user);
        assertEquals(username, user.getUsername());
    }
}
```

## 测试驱动开发（TDD）

强制工作流程：
1. 先编写测试（RED）
2. 运行测试 - 应该失败
3. 编写最小实现（GREEN）
4. 运行测试 - 应该通过
5. 重构优化（IMPROVE）
6. 验证覆盖率（80%+）

## 测试失败排查

1. 使用 **tdd-guide** Agent
2. 检查测试隔离性
3. 验证 mock 是否正确
4. 修复实现代码，而非测试代码（除非测试本身有问题）

## Agent 支持

- **tdd-guide** - 主动用于新功能，强制先编写测试
- **e2e-runner** - RestAssured E2E 测试专家
