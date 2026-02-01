# 更新文档

从单一真相来源同步文档：

1. 读取 `pom.xml`
   - 生成 Maven 命令参考表
   - 提取依赖和插件信息

2. 读取 `application.yml` / `application.properties`
   - 提取所有配置项
   - 记录用途和默认值

3. 读取 `README.md`
   - 检查项目描述是否与 pom.xml 一致

4. 生成 `docs/CONTRIB.md`，包含：
   - 开发工作流程
   - 可用的 Maven 命令
   - 环境配置
   - 测试程序

5. 生成 `docs/RUNBOOK.md`，包含：
   - 部署程序
   - 监控和警报
   - 常见问题和修复
   - 回滚程序

6. 识别过时的文档：
   - 找出 90 天以上未修改的文档
   - 列出供手动审查

7. 显示差异摘要

## Maven pom.xml 配置

### 常用 Maven 命令

| 命令 | 说明 |
|------|------|
| `mvn clean compile` | 清理并编译 |
| `mvn clean package` | 打包 |
| `mvn clean install` | 安装到本地仓库 |
| `mvn test` | 执行单元测试 |
| `mvn verify` | 执行集成测试 |
| `mvn spring-boot:run` | 启动应用 |

### 依赖管理

```xml
<!-- 在 pom.xml 中提取 -->
<dependencies>
    <!-- Spring Boot Starter -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <!-- MyBatis-Plus -->
    <dependency>
        <groupId>com.baomidou</groupId>
        <artifactId>mybatis-plus-boot-starter</artifactId>
    </dependency>

    <!-- MySQL Driver -->
    <dependency>
        <groupId>com.mysql</groupId>
        <artifactId>mysql-connector-j</artifactId>
        <scope>runtime</scope>
    </dependency>
</dependencies>
```

## application.yml 配置

### 常用配置项

| 配置项 | 说明 | 默认值 |
|--------|------|--------|
| `server.port` | 服务端口 | 8080 |
| `spring.datasource.url` | 数据库 URL | - |
| `spring.datasource.username` | 数据库用户名 | - |
| `spring.datasource.password` | 数据库密码 | - |
| `mybatis-plus.mapper-locations` | Mapper XML 位置 | classpath*:mapper/**/*.xml |
| `logging.level.root` | 日志级别 | INFO |

### 多环境配置

```
src/main/resources/
├── application.yml           # 通用配置
├── application-dev.yml       # 开发环境
├── application-test.yml      # 测试环境
└── application-prod.yml      # 生产环境
```

## 文档模板

### CONTRIBUTION.md 模板

```markdown
# 贡献指南

## 环境要求
- JDK 21+
- Maven 3.9+
- MySQL 8.0+

## 快速开始
\`\`\`bash
git clone <repo>
cd <project>
mvn clean install
mvn spring-boot:run
\`\`\`

## 开发流程
1. 创建功能分支
2. 编写测试（TDD）
3. 实现功能
4. 运行测试
5. 提交代码
```

### RUNBOOK.md 模板

```markdown
# 运维手册

## 部署流程
\`\`\`bash
mvn clean package
java -jar target/app.jar --spring.profiles.active=prod
\`\`\`

## 常见问题
### 应用启动失败
- 检查 MySQL 是否运行
- 检查端口是否被占用
- 检查配置文件是否正确
```

单一真相来源：pom.xml 和 application.yml
