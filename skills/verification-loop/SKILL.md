---
name: verification-loop
description: 完整的 Spring Boot 项目验证系统：构建验证、静态分析、测试覆盖率、安全扫描、代码审查、Docker 验证、性能验证。适配 Java 21 + Spring Boot 3 + MyBatis-Plus + MySQL 技术栈。
version: 1.1.0
tech_stack: [Java 21, Spring Boot 3.2, MyBatis-Plus, Docker, Maven]
tools: Read, Write, Edit, Bash, Grep, Glob
related_skills: [springboot-verification, springboot-tdd, eval-harness, security-review]
---

# 验证循环技能

Claude Code 会话的完整验证系统。

## 何时使用

在以下情况调用此技能：
- 完成功能或重大代码变更后
- 创建 PR 前
- 想确保质量门槛通过时
- 重构后
- 添加新 API 端点或 Service 后

## 与其他技能的职责划分

| 技能 | 职责 |
|------|------|
| `verification-loop` | 完整项目验证：构建、静态分析、安全扫描、Docker、性能 |
| `tdd-workflow` | 通用 TDD 方法论：红-绿-重构、测试设计原则 |
| `springboot-tdd` | Spring Boot 特定测试：@WebMvcTest、@DataJpaTest、@MockBean |
| `security-review` | 安全审查：认证授权、输入验证、密钥管理 |
| `eval-harness` | Eval 驱动开发：能力评估、回归测试 |

**核心区别**：
- 本技能聚焦 "部署前的全面质量验证"（包括构建、安全、性能、Docker）
- `tdd-workflow` 聚焦 "开发阶段的测试实践"（红-绿-重构循环）
- `security-review` 聚焦 "安全最佳实践"（认证、授权、漏洞防护）
- 使用顺序：`tdd-workflow`（开发） → `security-review`（安全检查） → `verification-loop`（全面验证）

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

---

## 阶段 8：Docker 验证

### Dockerfile 检查

```bash
# 检查 Dockerfile 语法
docker build --check -f Dockerfile .

# 分析 Dockerfile 最佳实践
hadolint Dockerfile
```

### Docker 镜像构建验证

```bash
# 构建镜像
docker build -t myapp:latest .

# 检查镜像大小
docker images myapp:latest

# 验证镜像可以启动
docker run --rm myapp:latest java -version

# 检查镜像层
docker history myapp:latest
```

### Docker 容器运行验证

```bash
# 启动容器
docker run -d --name test-container -p 8080:8080 myapp:latest

# 等待应用启动
sleep 10

# 健康检查
curl -f http://localhost:8080/actuator/health

# 执行应用自检
docker exec test-container curl -f http://localhost:8080/actuator/health

# 检查日志
docker logs test-container

# 清理
docker stop test-container
docker rm test-container
```

### Docker Compose 验证

```bash
# 启动完整环境
docker-compose -f docker-compose.yml -f docker-compose.test.yml up -d

# 等待服务就绪
docker-compose logs -f app

# 执行健康检查
curl http://localhost:8080/actuator/health

# 执行集成测试
docker-compose exec app mvn test

# 清理
docker-compose down -v
```

### Docker 安全扫描

```bash
# 使用 Trivy 扫描镜像漏洞
trivy image myapp:latest

# 使用 Snyk 扫描
snyk container test myapp:latest

# 检查镜像基础镜像
docker inspect myapp:latest | grep -A 10 "Layers"
```

---

## 阶段 9：性能验证

### 应用启动性能

```bash
# 测量应用启动时间
time java -jar target/app.jar

# 使用 Spring Boot Actuator
curl http://localhost:8080/actuator/metrics/jvm.memory.used
curl http://localhost:8080/actuator/metrics/process.uptime
```

### 内存使用分析

```bash
# 启动应用并监控内存
java -Xmx512m -Xms256m -jar target/app.jar &
PID=$!

# 等待应用启动
sleep 10

# 检查内存使用
jps -l | grep app
jmap -heap $PID

# 使用 VisualVM 连接分析
jvisualvm --openpid $PID

# 清理
kill $PID
```

### API 性能测试

```bash
# 使用 Apache Bench 进行简单压测
ab -n 1000 -c 10 http://localhost:8080/api/users

# 使用 wrk（更高级）
wrk -t4 -c100 -d30s http://localhost:8080/api/users

# 使用 curl 测试响应时间
time curl http://localhost:8080/api/users/1
curl -w "@curl-format.txt" -o /dev/null -s http://localhost:8080/api/users/1

# curl-format.txt
# time_namelookup: %{time_namelookup}\n
# time_connect: %{time_connect}\n
# time_appconnect: %{time_appconnect}\n
# time_pretransfer: %{time_pretransfer}\n
# time_starttransfer: %{time_starttransfer}\n
# time_total: %{time_total}\n
# http_code: %{http_code}\n
```

### 数据库性能验证

```bash
# 检查慢查询
curl http://localhost:8080/actuator/metrics/jdbc.connections.active

# 使用 JMX 监控连接池
jconsole

# 检查数据库连接数
mysql -e "SHOW PROCESSLIST" | wc -l
```

### JMeter 性能测试

```xml
<!-- user_test.jmx -->
<?xml version="1.0" encoding="UTF-8"?>
<jmeterTestPlan version="1.2">
  <hashTree>
    <TestPlan>
      <elementProp name="TestPlan.user_defined_variables" elementType="Arguments">
        <collectionProp name="Arguments.arguments">
          <elementProp name="BASE_URL" elementType="Argument">
            <stringProp name="Argument.name">BASE_URL</stringProp>
            <stringProp name="Argument.value">http://localhost:8080</stringProp>
          </elementProp>
        </collectionProp>
      </elementProp>
    </TestPlan>
    <hashTree>
      <ThreadGroup>
        <stringProp name="ThreadGroup.num_threads">100</stringProp>
        <stringProp name="ThreadGroup.ramp_time">10</stringProp>
        <longProp name="ThreadGroup.duration">60</longProp>
      </ThreadGroup>
      <hashTree>
        <HTTPSamplerProxy>
          <stringProp name="HTTPSampler.domain">${BASE_URL}</stringProp>
          <stringProp name="HTTPSampler.path">/api/users</stringProp>
          <stringProp name="HTTPSampler.method">GET</stringProp>
        </HTTPSamplerProxy>
      </hashTree>
    </hashTree>
  </hashTree>
</jmeterTestPlan>
```

```bash
# 运行 JMeter 测试
jmeter -n -t user_test.jmx -l result.jtl -e -o report/

# 检查结果
cat result.jtl | grep -c "true"
```

---

## CI/CD 平台集成

> **注意**：以下为各平台完整配置参考。实际项目应根据需求选择相应平台并调整配置。

### GitHub Actions 完整配置

```yaml
name: Spring Boot 验证

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  verify:
    runs-on: ubuntu-latest

    services:
      mysql:
        image: mysql:8.0
        env:
          MYSQL_ROOT_PASSWORD: test
          MYSQL_DATABASE: testdb
        options: >-
          --health-cmd="mysqladmin ping -h localhost"
          --health-interval=10s
          --health-timeout=5s
          --health-retries=5
        ports:
          - 3306:3306

    steps:
      - name: Checkout 代码
        uses: actions/checkout@v4

      - name: 设置 JDK 21
        uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
          cache: 'maven'

      - name: 编译项目
        run: mvn -T 4 clean compile

      - name: 运行测试
        run: mvn -T 4 test

      - name: 生成覆盖率报告
        run: mvn jacoco:report

      - name: 上传覆盖率报告
        uses: codecov/codecov-action@v3
        with:
          files: target/site/jacoco/jacoco.xml

      - name: 上传测试结果
        uses: actions/upload-artifact@v3
        if: always()
        with:
          name: test-results
          path: target/surefire-reports/
```

### GitLab CI 完整配置

```yaml
# .gitlab-ci.yml

image: maven:3.9-eclipse-temurin-21

variables:
  MAVEN_OPTS: "-Dmaven.repo.local=$CI_PROJECT_DIR/.m2/repository"
  DOCKER_DRIVER: overlay2
  DOCKER_TLS_CERTDIR: "/certs"

stages:
  - build
  - test
  - verify
  - security
  - docker
  - deploy

build:
  stage: build
  script:
    - mvn -T 4 clean compile
  artifacts:
    paths:
      - target/
    expire_in: 1 hour

unit-test:
  stage: test
  services:
    - mysql:8.0
  variables:
    MYSQL_DATABASE: testdb
    MYSQL_ROOT_PASSWORD: test
  script:
    - mvn -T 4 test
  artifacts:
    paths:
      - target/surefire-reports/
      - target/site/jacoco/
    expire_in: 1 week
  coverage: '/Lines: *(\d+\.\d+)%/'

static-analysis:
  stage: verify
  script:
    - mvn checkstyle:check
    - mvn spotbugs:check
  dependencies:
    - build

coverage-check:
  stage: verify
  script:
    - mvn jacoco:check
  dependencies:
    - unit-test

dependency-check:
  stage: security
  script:
    - mvn org.owasp:dependency-check-maven:check
  allow_failure: true

docker-build:
  stage: docker
  image: docker:24
  services:
    - docker:24-dind
  script:
    - docker build -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA .
    - docker build -t $CI_REGISTRY_IMAGE:latest .
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
    - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
    - docker push $CI_REGISTRY_IMAGE:latest

trivy-scan:
  stage: docker
  image: aquasec/trivy:latest
  script:
    - trivy image --exit-code 1 --severity HIGH,CRITICAL $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA

deploy-staging:
  stage: deploy
  image: alpine:3.18
  script:
    - echo "部署到测试环境"
    # kubectl apply -f k8s/
  environment:
    name: staging
    url: https://staging.example.com
  only:
    - develop

deploy-production:
  stage: deploy
  image: alpine:3.18
  script:
    - echo "部署到生产环境"
    # kubectl apply -f k8s/
  environment:
    name: production
    url: https://example.com
  when: manual
  only:
    - main
```

### Jenkins Pipeline 完整配置

```groovy
pipeline {
    agent any

    tools {
        maven 'Maven 3.9'
        jdk 'JDK 21'
    }

    environment {
        MYSQL_DATABASE = 'testdb'
        MYSQL_ROOT_PASSWORD = 'test'
        DOCKER_IMAGE = "myapp:${BUILD_NUMBER}"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build') {
            steps {
                sh 'mvn -T 4 clean compile'
            }
        }

        stage('Static Analysis') {
            parallel {
                stage('Checkstyle') {
                    steps {
                        sh 'mvn checkstyle:check'
                    }
                }
                stage('SpotBugs') {
                    steps {
                        sh 'mvn spotbugs:check'
                    }
                }
            }
        }

        stage('Test') {
            steps {
                sh 'mvn -T 4 test'
            }
            post {
                always {
                    junit 'target/surefire-reports/TEST-*.xml'
                }
            }
        }

        stage('Coverage') {
            steps {
                sh 'mvn jacoco:report'
            }
            post {
                always {
                    jacoco coverageCriteria: [
                        [lineCoverage: 80.0, branchCoverage: 70.0]
                    ]
                }
            }
        }

        stage('Dependency Check') {
            steps {
                sh 'mvn org.owasp:dependency-check-maven:check'
            }
        }

        stage('Docker Build') {
            steps {
                script {
                    def customImage = docker.build("${DOCKER_IMAGE}")
                    customImage.inside { ->
                        sh 'java -version'
                    }
                }
            }
        }

        stage('Security Scan') {
            steps {
                sh "trivy image --exit-code 0 --severity HIGH,CRITICAL ${DOCKER_IMAGE}"
            }
        }

        stage('Deploy Staging') {
            when {
                branch 'develop'
            }
            steps {
                sh "echo 'Deploying to staging...'"
                // sh "kubectl apply -f k8s/staging/"
            }
        }

        stage('Deploy Production') {
            when {
                branch 'main'
            }
            steps {
                input message: '部署到生产环境?', ok: '部署'
                sh "echo 'Deploying to production...'"
                // sh "kubectl apply -f k8s/production/"
            }
        }
    }

    post {
        always {
            cleanWs()
        }
        success {
            emailext(
                subject: "构建成功: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: "构建 ${env.BUILD_URL} 成功完成。",
                to: "team@example.com"
            )
        }
        failure {
            emailext(
                subject: "构建失败: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: "构建 ${env.BUILD_URL} 失败。",
                to: "team@example.com"
            )
        }
    }
}
```

### Azure Pipelines 配置

```yaml
# azure-pipelines.yml

trigger:
  branches:
    include:
      - main
      - develop

pr:
  branches:
    include:
      - main

pool:
  vmImage: 'ubuntu-latest'

variables:
  MAVEN_CACHE_FOLDER: $(Pipeline.Workspace)/.m2/repository
  DOCKER_IMAGE: 'myapp:$(Build.BuildId)'

stages:
  - stage: Build
    jobs:
      - job: Build
        steps:
          - task: Maven@4
            inputs:
              mavenPomFile: 'pom.xml'
              goals: 'clean compile'
              options: '-T 4'
              javaHomeOption: 'JDKVersion'
              jdkVersionOption: '1.21'
              mavenVersionOption: 'Default'

  - stage: Test
    jobs:
      - job: Test
        steps:
          - task: Maven@4
            inputs:
              mavenPomFile: 'pom.xml'
              goals: 'test'
              options: '-T 4'
              testResultsFiles: '**/surefire-reports/TEST-*.xml'
              javaHomeOption: 'JDKVersion'
              jdkVersionOption: '1.21'

          - task: PublishCodeCoverageResults@1
            inputs:
              codeCoverageTool: 'JaCoCo'
              summaryFileLocation: 'target/site/jacoco/jacoco.xml'

  - stage: Security
    jobs:
      - job: Security
        steps:
          - task: Maven@4
            inputs:
              mavenPomFile: 'pom.xml'
              goals: 'org.owasp:dependency-check-maven:check'

  - stage: Docker
    jobs:
      - job: Docker
        steps:
          - task: Docker@2
            inputs:
              command: build
              dockerfile: Dockerfile
              tags: |
                $(DOCKER_IMAGE)
                myapp:latest

          - script: |
              docker run --rm -d -p 8080:8080 --name test-container $(DOCKER_IMAGE)
              sleep 10
              curl -f http://localhost:8080/actuator/health
              docker stop test-container

  - stage: Deploy
    jobs:
      - deployment: Staging
        environment: 'staging'
        strategy:
          runOnce:
            deploy:
              steps:
                - script: |
                    echo "Deploying to staging..."
```
