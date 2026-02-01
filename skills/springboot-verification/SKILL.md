---
name: springboot-verification
description: Spring Boot 项目验证流程：构建验证、静态分析、测试覆盖率、安全扫描、差异审查、Docker 验证、CI/CD 集成。适配 Java 21 + Spring Boot 3 + MyBatis-Plus + MySQL 技术栈。
version: 1.1.0
tech_stack: [Java 21, Spring Boot 3, MyBatis-Plus, MySQL]
tools: Read, Write, Edit, Bash, Grep, Glob
related_skills:
  verification-loop: "通用验证循环"
  tdd-workflow: "TDD 方法论"
  java-testing: "Java 测试框架"
  security-review: "安全审查清单"
  eval-harness: "Eval 驱动开发框架"
---

# Spring Boot 验证流程

完整的 Spring Boot 项目验证系统，支持发版前或提交 PR 前的完整检查。

## 验证时机

- **提交 PR 前** - 确保代码质量
- **重大变更后** - 验证功能完整性
- **部署前** - 最终安全检查
- **定期执行** - 持续监控代码健康度

## 阶段 1：构建

### 基础构建

```bash
# 清理并编译（跳过测试）
mvn -T 4 clean compile

# 完整构建（包括测试）
mvn -T 4 clean verify

# 跳过测试的快速验证
mvn -T 4 clean verify -DskipTests
```

### 构建失败处理

如果构建失败，按以下顺序检查：

1. 检查依赖冲突 - `mvn dependency:tree`
2. 检查编译错误 - 修复 Java 语法
3. 检查资源文件 - 确认配置正确
4. 清理缓存 - `mvn clean` 后重试

### 依赖检查

```bash
# 查看依赖树
mvn dependency:tree

# 检查依赖冲突
mvn dependency:analyze

# 查找过时依赖
mvn versions:display-dependency-updates
```

## 阶段 2：静态分析

### SpotBugs（Bug 检测）

```bash
# 运行 SpotBugs 检查
mvn spotbugs:check

# 生成报告
mvn spotbugs:spotbugs
```

常见 SpotBugs 问题：

| 问题类型 | 说明 | 修复建议 |
|---------|------|---------|
| NP_NULL_ON_SOME_PATH | 空指针风险 | 添加 null 检查或使用 Optional |
| RCN_REDUNDANT_NULLCHECK_OF_NONNULL_VALUE | 多余的 null 检查 | 移除不必要的检查 |
| SE_BAD_FIELD | 非序列化字段 | 添加 transient 或实现 Serializable |
| DLS_DEAD_LOCAL_STORE | 死代码 | 移除未使用的变量 |

### Checkstyle（代码风格）

```bash
# 运行 Checkstyle 检查
mvn checkstyle:check

# 自动修复部分问题
mvn checkstyle:checkstyle
```

### PMD（代码质量）

```bash
# 运行 PMD 检查
mvn pmd:check

# 生成报告
mvn pmd:pmd
```

常见 PMD 问题：

| 问题类型 | 说明 | 修复建议 |
|---------|------|---------|
| EmptyIfStmt | 空的 if 语句 | 移除或添加逻辑 |
| UnusedImports | 未使用的导入 | 清理 import |
| EmptyControlStatement | 空控制语句 | 添加逻辑或移除 |
| SimplifyConditional | 简化条件 | 使用三元运算符 |

## 阶段 3：测试 + 覆盖率

### 运行测试

```bash
# 运行所有测试
mvn -T 4 test

# 运行指定测试类
mvn test -Dtest=UserServiceTest

# 运行指定测试方法
mvn test -Dtest=UserServiceTest#shouldCreateUser

# 跳过测试
mvn -DskipTests

# 并行测试（加速）
mvn -T 4 test -DforkCount=4
```

### 生成覆盖率报告

```bash
# 生成 JaCoCo 报告
mvn jacoco:report

# 检查覆盖率
mvn jacoco:check

# 查看报告
# 打开 target/site/jacoco/index.html
```

### 覆盖率目标

| 指标 | 最低要求 | 推荐值 |
|------|---------|-------|
| 指令覆盖率（Instruction） | 70% | 80%+ |
| 分支覆盖率（Branch） | 60% | 75%+ |
| 行覆盖率（Line） | 70% | 80%+ |
| 方法覆盖率（Method） | 80% | 90%+ |
| 类覆盖率（Class） | 80% | 95%+ |

### 测试报告检查清单

- [ ] 总测试数、通过/失败数正常
- [ ] 覆盖率百分比达标
- [ ] 无跳过的测试（除非有意为之）
- [ ] 无不稳定的测试
- [ ] 测试执行时间合理

## 阶段 4：安全扫描

### OWASP 依赖检查

```bash
# 运行 OWASP 依赖检查
mvn org.owasp:dependency-check-maven:check

# 生成报告
mvn org.owasp:dependency-check-maven:aggregate
```

### 漏洞处理优先级

| 严重级别 | CVSS 分数 | 处理要求 |
|---------|----------|---------|
| 严重（Critical） | 9.0-10.0 | 立即修复 |
| 高危（High） | 7.0-8.9 | 尽快修复 |
| 中危（Medium） | 4.0-6.9 | 计划修复 |
| 低危（Low） | 0.1-3.9 | 可选修复 |

### 密钥检测

```bash
# 使用 truffleHog 扫描密钥
trufflehog filesystem .

# 使用 git-secrets（需预配置）
git secrets --scan

# 使用 gitleaks
gitleaks detect
```

### 常见敏感信息

- 硬编码密码/API Key
- AWS/Google Cloud 访问密钥
- 数据库连接字符串
- OAuth Token
- 私钥文件

## 阶段 5：代码格式检查

### Spotless（代码格式化）

```bash
# 检查格式
mvn spotless:check

# 自动格式化
mvn spotless:apply

# 格式化配置（pom.xml）
```

```xml
<plugin>
    <groupId>com.diffplug.spotless</groupId>
    <artifactId>spotless-maven-plugin</artifactId>
    <version>2.41.1</version>
    <configuration>
        <java>
            <googleJavaFormat/>
            <removeUnusedImports/>
        </java>
    </configuration>
</plugin>
```

### 代码风格检查清单

- [ ] 一致的缩进（4 空格）
- [ ] 一致的括号风格
- [ ] 导入语句排序
- [ ] 无未使用的导入
- [ ] 一致的空行使用

## 阶段 6：差异审查

### 查看变更

```bash
# 查看变更统计
git diff --stat

# 查看详细差异
git diff

# 查看暂存区差异
git diff --staged

# 查看最近 N 次提交
git log -n 5 --oneline
```

### 代码审查清单

#### 功能性

- [ ] 实现符合需求
- [ ] 边界条件已处理
- [ ] 错误场景已覆盖

#### 代码质量

- [ ] 无调试日志（System.out.println）
- [ ] 无 TODO/FIXME（除非跟踪）
- [ ] 有意义的变量/方法命名
- [ ] 适当的注释

#### 安全性

- [ ] 无 SQL 注入风险
- [ ] 无 XSS 风险
- [ ] 输入验证完整
- [ ] 敏感数据已脱敏

#### 性能

- [ ] 无 N+1 查询问题
- [ ] 适当的缓存策略
- [ ] 无内存泄漏风险
- [ ] 数据库查询优化

#### 测试

- [ ] 单元测试覆盖新代码
- [ ] 测试用例有意义
- [ ] 无脆弱测试

### 提交信息规范

```
<type>(<scope>): <subject>

<body>

<footer>
```

类型（type）：
- feat: 新功能
- fix: 修复 Bug
- docs: 文档变更
- style: 代码格式
- refactor: 重构
- test: 测试相关
- chore: 构建/工具

示例：

```
feat(user): add email verification

- Implement email verification flow
- Add verification token service
- Update user registration endpoint

Closes #123
```

## 输出模板

```
验证报告
===================
构建：     [通过/失败]
静态分析： [通过/失败] (spotbugs/checkstyle/pmd)
测试：     [通过/失败] (X/Y 通过，Z% 覆盖率)
安全：     [通过/失败] (发现 N 个漏洞)
格式：     [通过/失败]
差异：     [X 个文件变更，+Y/-Z 行]

整体：     [就绪 / 未就绪]

待修复问题：
1. [问题描述] - 严重程度
2. [问题描述] - 严重程度

建议：
- [改进建议 1]
- [改进建议 2]
```

## 持续模式

在开发过程中保持快速反馈循环：

```bash
# 快速检查（每次保存后）
mvn -T 4 compile

# 中等检查（每 15-30 分钟）
mvn -T 4 test

# 完整检查（提交前）
mvn -T 4 clean verify
```

## 阶段 7：Docker 验证

### Dockerfile 检查

```bash
# 构建镜像
docker build -t myapp:latest .

# 检查镜像大小
docker images myapp:latest

# 运行容器验证
docker run --rm myapp:latest java -version

# 健康检查
docker run --rm -p 8080:8080 myapp:latest \
  curl -f http://localhost:8080/actuator/health
```

### Dockerfile 最佳实践

```dockerfile
# 多阶段构建 - 减小镜像大小
FROM maven:3.9-eclipse-temurin-21 AS builder
WORKDIR /app
COPY pom.xml .
COPY src ./src
RUN mvn -T 4 clean package -DskipTests

# 运行时镜像 - 使用轻量级基础镜像
FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
COPY --from=builder /app/target/*.jar app.jar
EXPOSE 8080
HEALTHCHECK --interval=30s --timeout=3s \
  CMD curl -f http://localhost:8080/actuator/health || exit 1
ENTRYPOINT ["java", "-jar", "app.jar"]
```

### Docker Compose 验证

```bash
# 启动完整环境
docker-compose up -d

# 等待服务就绪
docker-compose logs -f app

# 执行健康检查
curl http://localhost:8080/actuator/health

# 执行集成测试
docker-compose exec app mvn test

# 清理
docker-compose down -v
```

## CI/CD 集成

### GitHub Actions 示例

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

      - name: 运行静态分析
        run: |
          mvn -T 4 spotbugs:check
          mvn -T 4 checkstyle:check

      - name: 运行测试
        run: mvn -T 4 test

      - name: 生成覆盖率报告
        run: mvn jacoco:report

      - name: 检查覆盖率
        run: |
          COVERAGE=$(mvn jacoco:check | grep "Line Coverage" | awk '{print $4}' | tr -d '%')
          echo "覆盖率: $COVERAGE%"
          if (( $(echo "$COVERAGE < 80" | bc -l) )); then
            echo "覆盖率不足 80%"
            exit 1
          fi

      - name: OWASP 依赖检查
        run: mvn org.owasp:dependency-check-maven:check

      - name: 上传覆盖率报告
        uses: codecov/codecov-action@v3
        with:
          files: target/site/jacoco/jacoco.xml
```

### GitLab CI 示例

```yaml
# .gitlab-ci.yml

image: maven:3.9-eclipse-temurin-21

variables:
  MAVEN_OPTS: "-Dmaven.repo.local=$CI_PROJECT_DIR/.m2/repository"

stages:
  - build
  - test
  - verify
  - deploy

build:
  stage: build
  script:
    - mvn -T 4 clean compile
  artifacts:
    paths:
      - target/
    expire_in: 1 hour

test:
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

verify:
  stage: verify
  script:
    - mvn -T 4 spotbugs:check
    - mvn -T 4 checkstyle:check
    - mvn jacoco:check
  dependencies:
    - build
  only:
    - main
    - develop
```

### Jenkins Pipeline 示例

```groovy
pipeline {
    agent any

    stages {
        stage('Build') {
            steps {
                sh 'mvn -T 4 clean compile'
            }
        }
        stage('Test') {
            steps {
                sh 'mvn -T 4 test'
                junit 'target/surefire-reports/TEST-*.xml'
            }
        }
        stage('Coverage') {
            steps {
                sh 'mvn jacoco:report'
                jacoco coverageCriteria: [
                    [lineCoverage: 80.0, branchCoverage: 70.0]
                ]
            }
        }
        stage('Verify') {
            steps {
                sh 'mvn spotbugs:check'
                sh 'mvn checkstyle:check'
                sh 'mvn org.owasp:dependency-check-maven:check'
            }
        }
        stage('Docker') {
            steps {
                sh 'docker build -t myapp:${BUILD_NUMBER} .'
                sh 'docker run --rm myapp:${BUILD_NUMBER} curl -f http://localhost:8080/actuator/health'
            }
        }
    }
}
```

## 一键验证脚本

### Windows PowerShell

```powershell
# verify.ps1
# Spring Boot 项目一键验证脚本

Write-Host "=== Spring Boot 项目验证 ===" -ForegroundColor Cyan

$ErrorActionPreference = "Stop"

try {
    # 阶段 1：构建
    Write-Host "`n[1/7] 构建项目..." -ForegroundColor Yellow
    mvn clean compile -T 4

    # 阶段 2：静态分析
    Write-Host "`n[2/7] 静态分析..." -ForegroundColor Yellow
    mvn spotbugs:check
    mvn checkstyle:check

    # 阶段 3：测试
    Write-Host "`n[3/7] 运行测试..." -ForegroundColor Yellow
    mvn test -T 4

    # 阶段 4：覆盖率
    Write-Host "`n[4/7] 生成覆盖率报告..." -ForegroundColor Yellow
    mvn jacoco:report

    # 阶段 5：安全扫描
    Write-Host "`n[5/7] 安全扫描..." -ForegroundColor Yellow
    mvn org.owasp:dependency-check-maven:check

    # 阶段 6：差异检查
    Write-Host "`n[6/7] 差异检查..." -ForegroundColor Yellow
    git diff --stat

    Write-Host "`n[7/7] 验证完成！" -ForegroundColor Green
    Write-Host "`n所有检查通过，可以提交 PR。" -ForegroundColor Green
}
catch {
    Write-Host "`n`n验证失败：$_" -ForegroundColor Red
    exit 1
}
```

### macOS/Linux/WSL

```bash
#!/bin/bash
# verify.sh
# Spring Boot 项目一键验证脚本

set -e

echo "=== Spring Boot 项目验证 ==="

# 阶段 1：构建
echo "[1/7] 构建项目..."
mvn -T 4 clean compile

# 阶段 2：静态分析
echo "[2/7] 静态分析..."
mvn spotbugs:check
mvn checkstyle:check

# 阶段 3：测试
echo "[3/7] 运行测试..."
mvn -T 4 test

# 阶段 4：覆盖率
echo "[4/7] 生成覆盖率报告..."
mvn jacoco:report

# 阶段 5：安全扫描
echo "[5/7] 安全扫描..."
mvn org.owasp:dependency-check-maven:check

# 阶段 6：差异检查
echo "[6/7] 差异检查..."
git diff --stat

echo "[7/7] 验证完成！"
echo "所有检查通过，可以提交 PR。"
```

## 验证失败处理

### 常见问题

| 问题 | 可能原因 | 解决方案 |
|------|---------|---------|
| 编译失败 | 依赖冲突 | `mvn dependency:tree` 检查 |
| 测试失败 | 代码变更 | 运行单个测试定位问题 |
| 覆盖率不足 | 缺少测试 | 补充测试用例 |
| 静态分析失败 | 代码风格 | 自动修复或手动调整 |
| 安全漏洞 | 依赖版本 | 升级依赖或添加例外 |

### 回滚策略

```bash
# 查看提交历史
git log --oneline -n 10

# 回滚到指定提交
git reset --hard <commit-hash>

# 撤销最近一次提交（保留更改）
git reset --soft HEAD~1

# 撤销最近一次提交（丢弃更改）
git reset --hard HEAD~1
```

**记住**：快速反馈胜过后期意外。严格把关——在生产系统中将警告视为缺陷。

## 相关技能

- `verification-loop` - 通用验证循环
- `tdd-workflow` - 测试驱动开发
- `java-testing` - Java 测试指南
- `security-review` - 安全审查清单
- `eval-harness` - Eval 驱动开发
