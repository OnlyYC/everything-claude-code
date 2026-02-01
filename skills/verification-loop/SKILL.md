# 验证循环技能

Claude Code 会话的完整验证系统。

## 何时使用

在以下情况调用此技能：
- 完成功能或重大代码变更后
- 创建 PR 前
- 想确保质量门槛通过时
- 重构后
- 添加新 API 端点或 Service 后

## 验证阶段

### 阶段 1：构建验证
```bash
# 编译项目
mvn clean compile 2>&1 | tail -30

# 完整打包（可选，用于验证完整构建）
mvn clean package -DskipTests 2>&1 | tail -30
```

如果构建失败，停止并在继续前修复。

常见构建失败原因：
- 依赖版本冲突
- Java 版本不匹配（需要 Java 21）
- MyBatis Mapper XML 配置错误
- Spring Boot 配置问题

### 阶段 2：静态代码分析
```bash
# Checkstyle 检查代码风格
mvn checkstyle:check 2>&1 | head -40

# SpotBugs 检查潜在 Bug
mvn spotbugs:check 2>&1 | head -40

# PMD 检查代码质量
mvn pmd:check 2>&1 | head -40
```

报告所有静态分析问题。继续前修复关键问题。

### 阶段 3：依赖安全检查
```bash
# OWASP Dependency Check
mvn org.owasp:dependency-check-maven:check 2>&1 | head -40

# 或使用 Snyk
snyk test 2>&1 | head -30
```

### 阶段 4：测试套件
```bash
# 执行单元测试
mvn test 2>&1 | tail -50

# 执行带覆盖率的测试
mvn clean test jacoco:report 2>&1 | tail -50

# 检查覆盖率报告
cat target/site/jacoco/index.html | grep -o "Total[0-9]*%" | head -5
```

报告：
- 总测试数：X
- 通过：X
- 失败：X
- 错误：X
- 跳过：X
- 覆盖率：X%（目标：最低 80%）

### 阶段 5：安全扫描
```bash
# 检查硬编码密钥
grep -rn "sk-" --include="*.java" --include="*.yml" --include="*.properties" src/ 2>/dev/null | head -10
grep -rn "api_key\|apiKey\|API_KEY" --include="*.java" src/ 2>/dev/null | head -10
grep -rn "password.*=.*[^${}]" --include="*.java" --include="*.yml" src/ 2>/dev/null | head -10

# 检查 SQL 注入风险（字符串拼接 SQL）
grep -rn "\".*SELECT.*\" + " --include="*.java" src/ 2>/dev/null | head -10
grep -rn "\".*INSERT.*\" + " --include="*.java" src/ 2>/dev/null | head -10

# 检查 System.out.println（应该使用日志）
grep -rn "System\.out\.println" --include="*.java" src/ 2>/dev/null | head -10
grep -rn "printStackTrace" --include="*.java" src/ 2>/dev/null | head -10

# 检查 @Autowired 在字段上（应该使用构造器注入）
grep -rn "@Autowired" --include="*.java" -A 2 src/ 2>/dev/null | grep -v "constructor" | head -10

# 检查过度使用 @Transactional
grep -rn "@Transactional" --include="*.java" src/ 2>/dev/null | wc -l
```

### 阶段 6：MyBatis/SQL 验证
```bash
# 验证 Mapper XML 语法
find src/main/resources -name "*.xml" -exec xmllint --noout {} \; 2>&1 | head -20

# 检查未使用的 Mapper 方法（需要手动审查）
grep -rn "interface.*Mapper" --include="*.java" src/main/java/mapper/

# 检查硬编码表名
grep -rn "\".*FROM.*\"" --include="*.xml" src/main/resources/ 2>/dev/null | head -10
```

### 阶段 7：差异审查
```bash
# 显示变更统计
git diff --stat

# 显示变更文件列表
git diff HEAD~1 --name-only

# 显示具体变更
git diff

# 或使用 pager
git diff | less -R
```

审查每个变更的文件：
- 非预期变更
- 缺少错误处理
- 潜在边界案例
- Spring 注解使用是否正确
- MyBatis-Plus 使用是否规范

## 输出格式

执行所有阶段后，生成验证报告：

```
验证报告
==================

构建：       [PASS/FAIL]
静态分析：   [PASS/FAIL]（Checkstyle: X, SpotBugs: Y, PMD: Z）
依赖安全：   [PASS/FAIL]（X 个漏洞）
测试：       [PASS/FAIL]（X 通过，Y 失败，Z 覆盖率）
代码安全：   [PASS/FAIL]（X 个问题）
SQL 验证：   [PASS/FAIL]（X 个问题）
差异：       [X 个文件变更，+Y 行，-Z 行]

整体：       [READY/NOT READY] for PR

待修复问题：
1. [高] UserService.java:45 - 硬编码密码
2. [中] ProductMapper.xml:12 - SQL 拼接风险
3. [低] Checkstyle - 缺少 Javadoc
...
```

## pom.xml 配置参考

完整的验证工具配置：

```xml
<properties>
    <java.version>21</java.version>
    <maven.compiler.source>21</maven.compiler.source>
    <maven.compiler.target>21</maven.compiler.target>
</properties>

<dependencies>
    <!-- Spring Boot Starter -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <!-- MyBatis-Plus -->
    <dependency>
        <groupId>com.baomidou</groupId>
        <artifactId>mybatis-plus-spring-boot3-starter</artifactId>
        <version>3.5.5</version>
    </dependency>

    <!-- Validation -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-validation</artifactId>
    </dependency>

    <!-- Test Dependencies -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>

<build>
    <plugins>
        <!-- Spring Boot Plugin -->
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
        </plugin>

        <!-- Compiler Plugin - Java 21 -->
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-compiler-plugin</artifactId>
            <version>3.12.1</version>
            <configuration>
                <source>21</source>
                <target>21</target>
                <release>21</release>
                <compilerArgs>
                    <arg>-Xlint:all</arg>
                    <arg>-Werror</arg>
                </compilerArgs>
            </configuration>
        </plugin>

        <!-- Checkstyle -->
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-checkstyle-plugin</artifactId>
            <version>3.3.1</version>
            <configuration>
                <configLocation>checkstyle.xml</configLocation>
                <consoleOutput>true</consoleOutput>
                <failsOnError>true</failsOnError>
                <violationSeverity>warning</violationSeverity>
            </configuration>
            <executions>
                <execution>
                    <id>validate</id>
                    <phase>validate</phase>
                    <goals>
                        <goal>check</goal>
                    </goals>
                </execution>
            </executions>
        </plugin>

        <!-- SpotBugs -->
        <plugin>
            <groupId>com.github.spotbugs</groupId>
            <artifactId>spotbugs-maven-plugin</artifactId>
            <version>4.8.3.0</version>
            <configuration>
                <effort>Max</effort>
                <threshold>Low</threshold>
                <failOnError>true</failOnError>
            </configuration>
        </plugin>

        <!-- JaCoCo Coverage -->
        <plugin>
            <groupId>org.jacoco</groupId>
            <artifactId>jacoco-maven-plugin</artifactId>
            <version>0.8.11</version>
            <executions>
                <execution>
                    <goals>
                        <goal>prepare-agent</goal>
                    </goals>
                </execution>
                <execution>
                    <id>report</id>
                    <phase>test</phase>
                    <goals>
                        <goal>report</goal>
                    </goals>
                </execution>
                <execution>
                    <id>check</id>
                    <goals>
                        <goal>check</goal>
                    </goals>
                    <configuration>
                        <rules>
                            <rule>
                                <element>CLASS</element>
                                <limits>
                                    <limit>
                                        <counter>LINE</counter>
                                        <value>COVEREDRATIO</value>
                                        <minimum>0.80</minimum>
                                    </limit>
                                </limits>
                            </rule>
                        </rules>
                    </configuration>
                </execution>
            </executions>
        </plugin>

        <!-- Surefire - Unit Tests -->
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-surefire-plugin</artifactId>
            <version>3.2.2</version>
            <configuration>
                <includes>
                    <include>**/*Test.java</include>
                    <include>**/*Tests.java</include>
                </includes>
            </configuration>
        </plugin>

        <!-- OWASP Dependency Check -->
        <plugin>
            <groupId>org.owasp</groupId>
            <artifactId>dependency-check-maven</artifactId>
            <version>9.0.1</version>
            <configuration>
                <failBuildOnCVSS>7</failBuildOnCVSS>
            </configuration>
        </plugin>
    </plugins>
</build>
```

## 持续模式

对于长时会话，每 15 分钟或重大变更后执行验证：

```markdown
设置心理检查点：
- 完成每个 Service 方法后
- 完成 Controller 后
- 完成 Mapper 后
- 移至下一个任务前

执行：/verify
```

## 常见问题检查清单

### Spring Boot 最佳实践
- [ ] 使用构造器注入而非 @Autowired 字段注入
- [ ] Controller 使用 @Validated 进行参数验证
- [ ] 异常使用 @ControllerAdvice 统一处理
- [ ] 配置使用 @ConfigurationProperties 而非 @Value
- [ ] 日志使用 SLF4J 而非 System.out

### MyBatis-Plus 最佳实践
- [ ] 使用 @TableName 指定表名
- [ ] 使用 @TableId 指定主键策略
- [ ] 逻辑删除使用 @TableLogic
- [ ] 复杂查询使用自定义 Mapper XML
- [ ] 避免使用 select *（指定具体字段）

### Java 21 最佳实践
- [ ] 使用 Record 作为不可变 DTO
- [ ] 使用 Pattern Matching 简化类型检查
- [ ] 使用 Text Blocks 构建多行字符串/SQL
- [ ] 使用 Switch Expressions
- [ ] 使用 var 简化类型声明

### 安全检查
- [ ] 无硬编码密钥或密码
- [ ] 无 SQL 字符串拼接
- [ ] 敏感数据使用 @JsonIgnore
- [ ] 输入验证使用 @Valid/@Validated
- [ ] 敏感操作添加审计日志

## 快速验证命令

一键执行所有验证：

```bash
mvn clean compile checkstyle:check spotbugs:check test jacoco:report
```

或创建验证脚本：

```bash
#!/bin/bash
# verify.sh

set -e

echo "=== 构建验证 ==="
mvn clean compile

echo "=== 静态分析 ==="
mvn checkstyle:check
mvn spotbugs:check

echo "=== 测试执行 ==="
mvn test

echo "=== 覆盖率报告 ==="
mvn jacoco:report

echo "=== 安全扫描 ==="
grep -rn "sk-" src/ || echo "无密钥泄露"

echo "=== 验证完成 ==="
```

## 与 Hooks 集成

此技能补充 PostToolUse hooks 但提供更深入的验证。
Hooks 立即捕捉问题；此技能提供全面审查。

推荐 Pre-Commit Hook：

```bash
#!/bin/bash
# .git/hooks/pre-commit

mvn test && mvn checkstyle:check
```

推荐 Pre-Push Hook：

```bash
#!/bin/bash
# .git/hooks/pre-push

mvn clean test jacoco:check
```
