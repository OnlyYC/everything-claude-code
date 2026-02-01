---
name: update-docs
description: 从单一真相来源同步项目文档
command: /update-docs [--dry-run] [--check-only] [--output <目录>]
---

# 更新文档

从单一真相来源（pom.xml、application.yml）同步项目文档。

## 使用方法

```
/update-docs                    # 分析并更新文档
/update-docs --dry-run          # 预览变更但不写入
/update-docs --check-only       # 仅检查文档是否过时
/update-docs --output ./docs    # 指定输出目录
```

## 执行步骤

### 第一步：读取 pom.xml

**PowerShell (Windows):**
```powershell
# 读取项目基本信息
[xml]$pom = Get-Content "pom.xml"
$groupId = $pom.project.groupId
$artifactId = $pom.project.artifactId
$version = $pom.project.version
$name = $pom.project.name
$description = $pom.project.description

Write-Output "项目: $name ($artifactId v$version)"
Write-Output "描述: $description"

# 列出所有依赖
$pom.project.dependencies.dependency | ForEach-Object {
    Write-Output "$($_.groupId):$($_.artifactId):$($_.version)"
}
```

**Bash (macOS/Linux):**
```bash
# 使用 xmllint 或 mvn 提取信息
mvn help:evaluate -Dexpression=project.name -q -DforceStdout
mvn help:evaluate -Dexpression=project.description -q -DforceStdout
mvn help:evaluate -Dexpression=project.version -q -DforceStdout

# 列出依赖树
mvn dependency:tree
```

### 第二步：读取 application.yml

**Bash:**
```bash
# 提取所有配置项
grep -E "^[a-z]" application.yml | sort

# 提取带注释的配置
grep -B1 "^[a-z]" application.yml
```

**PowerShell:**
```powershell
# 读取并显示配置
Get-Content "application.yml" | Select-String -Pattern "^[a-z]"
```

### 第三步：检查 README.md 一致性

```bash
# 检查项目名称是否一致
PROJECT_NAME=$(mvn help:evaluate -Dexpression=project.name -q -DforceStdout)
if ! grep -q "$PROJECT_NAME" README.md; then
    echo "警告: README.md 中的项目名称与 pom.xml 不一致"
fi
```

### 第四步：生成 docs/CONTRIB.md

```bash
# 创建 docs 目录（如果不存在）
mkdir -p docs

# 从 pom.xml 提取信息生成贡献指南
cat > docs/CONTRIB.md << 'EOF'
# 贡献指南

## 项目信息
EOF

# 添加项目名称和版本
echo "- **项目**: $(mvn help:evaluate -Dexpression=project.name -q -DforceStdout)" >> docs/CONTRIB.md
echo "- **版本**: $(mvn help:evaluate -Dexpression=project.version -q -DforceStdout)" >> docs/CONTRIB.md

# 添加环境要求和快速开始（从模板）
cat >> docs/CONTRIB.md << 'TEMPLATETOKEN'

## 环境要求
- JDK 21+
- Maven 3.9+
- MySQL 8.0+

## 快速开始
```bash
git clone <repo>
cd <project>
mvn clean install
mvn spring-boot:run
```
TEMPLATETOKEN
```

### 第五步：生成 docs/RUNBOOK.md

```bash
cat > docs/RUNBOOK.md << 'EOF'
# 运维手册

## 部署流程
```bash
mvn clean package
java -jar target/app.jar --spring.profiles.active=prod
```

## 常见问题
EOF

# 从 application.yml 提取配置项并添加到常见问题
echo "### 配置项检查" >> docs/RUNBOOK.md
grep -E "^[a-z]" application.yml | head -10 >> docs/RUNBOOK.md
```

### 第六步：识别过时的文档

检查文档中的版本、依赖和配置是否与当前代码一致：

```bash
# 检查 README.md 中的版本是否过时
CURRENT_VERSION=$(mvn help:evaluate -Dexpression=project.version -q -DforceStdout)
if ! grep -q "$CURRENT_VERSION" README.md; then
    echo "警告: README.md 中的版本可能与 pom.xml 不一致"
fi

# 检查文档中提到的依赖版本
POM_DEPS=$(mvn dependency:list | grep -E "^\[INFO\]" | wc -l)
echo "当前依赖数量: $POM_DEPS"
```

报告格式：
```
文档新鲜度检查：
  README.md:     [最新/过时]
  docs/API.md:   [需要更新/已弃用]
  docs/DEPLOY.md: [最新/过时]
```

本指令支持以下环境：
- **Windows**: PowerShell 5.1+ 或 Git Bash
- **macOS/Linux**: Bash 4.0+

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
- Maven 3.9+ (或 Gradle 8.x+)
- MySQL 8.0+

## 快速开始

**Windows (PowerShell):**
```powershell
git clone <repo>
cd <project>
mvn clean install
mvn spring-boot:run
```

**macOS/Linux:**
```bash
git clone <repo>
cd <project>
mvn clean install
mvn spring-boot:run
```

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

**Windows (PowerShell):**
```powershell
mvn clean package
java -jar target\app.jar --spring.profiles.active=prod
```

**macOS/Linux:**
```bash
mvn clean package
java -jar target/app.jar --spring.profiles.active=prod
```

## 常见问题
### 应用启动失败
- 检查 MySQL 是否运行
- 检查端口是否被占用
- 检查配置文件是否正确
```

单一真相来源：pom.xml 和 application.yml

## 输出报告格式

```
====================================
文档更新报告
====================================

分析日期: $(date +%Y-%m-%d)

源文件状态:
  pom.xml:           $(stat -c %y pom.xml)
  application.yml:   $(stat -c %y application.yml)

目标文件状态:
  docs/CONTRIB.md:   $(stat -c %y docs/CONTRIB.md 2>/dev/null || echo '需要创建')
  docs/RUNBOOK.md:   $(stat -c %y docs/RUNBOOK.md 2>/dev/null || echo '需要创建')

变更摘要:
  - 新增依赖: N 个
  - 新增配置项: N 个
  - 过时文档: N 个
  - 需要更新的文档: docs/CONTRIB.md, docs/RUNBOOK.md
```

## 相关指令

- `/setup-pm` - 配置项目构建工具
- `/update-codemaps` - 分析代码库结构并更新架构文档

## 完整执行流程示例

```bash
# 完整的文档更新流程
echo "=== 开始文档更新 ==="

# 1. 检查源文件
echo "检查源文件..."
test -f pom.xml && echo "✓ pom.xml 存在" || echo "✗ pom.xml 不存在"
test -f application.yml && echo "✓ application.yml 存在" || echo "✗ application.yml 不存在"

# 2. 创建输出目录
mkdir -p docs
echo "✓ docs 目录已就绪"

# 3. 生成文档
echo "生成文档..."
# [执行上述生成命令]

# 4. 显示摘要
echo "=== 文档更新完成 ==="
ls -lh docs/
```
