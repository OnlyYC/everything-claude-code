---
name: setup-pm
description: 检测并配置项目构建工具（Maven/Gradle），自动设置环境
command: /setup-pm [--force] [--tool <maven|gradle>]
---

# 构建工具配置

检测并配置项目使用的构建工具，自动创建配置文件。

## 使用方法

```
/setup-pm                              # 自动检测并配置
/setup-pm --tool maven                 # 强制使用 Maven
/setup-pm --tool gradle                # 强制使用 Gradle
/setup-pm --force                      # 覆盖现有配置
```

## 执行步骤

### 第一步：检测构建工具

**PowerShell (Windows):**
```powershell
# 检测当前项目的构建工具
if (Test-Path "pom.xml") { Write-Output "Maven" }
elseif (Test-Path "build.gradle") { Write-Output "Gradle" }
elseif (Test-Path "build.gradle.kts") { Write-Output "Gradle (Kotlin)" }
else { Write-Output "Unknown" }

# 查找所有构建文件
Get-ChildItem -Filter "pom.xml" -Depth 0 -ErrorAction SilentlyContinue
Get-ChildItem -Filter "build.gradle*" -Depth 0 -ErrorAction SilentlyContinue
```

**Bash (macOS/Linux/Git Bash):**
```bash
# 检测当前项目的构建工具
if [ -f "pom.xml" ]; then
    echo "Maven"
elif [ -f "build.gradle" ]; then
    echo "Gradle"
elif [ -f "build.gradle.kts" ]; then
    echo "Gradle (Kotlin)"
else
    echo "Unknown"
fi

# 查找所有构建文件
find . -maxdepth 1 -name "pom.xml" -o -name "build.gradle" -o -name "build.gradle.kts"
```

### 第二步：创建配置文件

检测完成后，自动创建 `.claude/build-tool.json` 配置文件：

```json
{
  "buildTool": "maven",
  "detectedAt": "2025-02-01T10:30:00Z",
  "configFiles": ["pom.xml"],
  "javaVersion": "21",
  "springBootVersion": "3.2.0"
}
```

### 第三步：验证配置

**验证 Maven 配置：**
```bash
# 检查 Maven 版本
mvn --version

# 查看项目信息
mvn help:evaluate -Dexpression=project.version -q -DforceStdout

# 验证依赖树
mvn dependency:tree
```

**验证 Gradle 配置：**
```bash
# 检查 Gradle 版本
gradle --version
# 或使用 Gradle Wrapper
./gradlew --version

# 查看项目信息
gradle projects

# 验证依赖
gradle dependencies
```

## 检测优先级

系统按以下顺序决定使用哪个构建工具：

1. **环境变量**：`CLAUDE_BUILD_TOOL`
2. **项目配置**：`.claude/build-tool.json`
3. **项目文件**：是否存在 `pom.xml` 或 `build.gradle*`
4. **全局配置**：`~/.claude/build-tool.json`
5. **回退**：Maven（默认）

## 配置文件

### 项目配置
```json
// .claude/build-tool.json
{
  "buildTool": "maven"
}
```

### 全局配置
```json
// ~/.claude/build-tool.json
{
  "buildTool": "maven"
}
```

### 第四步：验证配置生效

```bash
# 检查配置文件是否存在
cat .claude/build-tool.json

# 输出当前构建工具
echo "当前构建工具: $(jq -r .buildTool .claude/build-tool.json)"
```

## 环境变量

## Maven 常用命令

| 命令 | 说明 |
|------|------|
| `mvn clean compile` | 清理并编译 |
| `mvn clean package` | 打包 |
| `mvn clean install` | 安装到本地仓库 |
| `mvn test` | 执行单元测试 |
| `mvn verify` | 执行集成测试 |
| `mvn spring-boot:run` | 启动应用 |

## Gradle 常用命令

| 命令 | 说明 |
|------|------|
| `gradle clean build` | 清理并构建 |
| `gradle test` | 执行测试 |
| `gradle bootRun` | 启动应用 |
| `./gradlew` | 使用 Gradle Wrapper |

## 相关指令

- `/build-fix` - 修复编译错误
- `/verify` - 执行完整验证循环
- `/update-docs` - 同步项目文档

## 配置验证命令

**完整验证流程：**

```bash
# 1. 检查配置文件
ls -la .claude/build-tool.json

# 2. 显示构建工具信息
echo "=== 构建工具配置 ==="
cat .claude/build-tool.json

# 3. 验证工具可用性
echo "=== 工具版本 ==="
mvn --version 2>/dev/null || gradle --version

# 4. 测试构建
echo "=== 测试构建 ==="
mvn clean compile -q || gradle clean build -q

echo "配置验证完成！"
```
