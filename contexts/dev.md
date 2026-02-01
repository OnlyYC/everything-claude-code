# 开发上下文

模式：活跃开发
重点：功能实现、代码编写、需求落地

## 行为准则
- 代码先行，解释在后
- 可用方案优于完美方案
- 修改代码后运行单元测试
- 保持提交原子性（一次提交一个功能点）

## 优先级原则
1. 先跑通
2. 再正确
3. 后重构

## 技术栈最佳实践

### 后端架构（Java 21 + Spring Boot 3）
- **分层架构**：Controller → Service → Mapper，职责分明
- **实体类**：使用 Lombok 简化代码，@Data/@Builder/@AllArgsConstructor
- **接口返回**：统一使用 Result<T> 包装响应数据
- **异常处理**：全局异常处理器 @ControllerAdvice + 自定义业务异常

### 持久层规范
- **MyBatis-Plus**：优先使用 BaseMapper 内置方法，减少 XML 编写
- **复杂查询**：多表关联使用 MyBatis XML 映射
- **事务管理**：@Transactional 加在 Service 层方法上
- **数据库**：字段命名使用下划线风格（user_name），实体类使用驼峰（userName）

### 代码规范
- **命名约定**：
  - 类名：大驼峰（UserController）
  - 方法名：小驼峰（getUserList）
  - 常量：全大写下划线（MAX_COUNT）
- **注释规范**：类级注释说明职责，方法级注释说明业务逻辑
- **日志记录**：关键业务节点使用 Slf4j 记录日志

### 推荐工具
- Edit、Write：代码修改
- Bash：运行 Maven 测试/构建、启动 Spring Boot 应用
- Grep、Glob：代码检索与定位

### 常用命令
```bash
# Maven 构建
mvn clean package

# 运行测试
mvn test

# 启动应用
mvn spring-boot:run
```
