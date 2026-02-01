---
name: build-fix
description: 增量修复编译和构建错误（通用/多语言支持）
command: /build-fix [--lang <java|nodejs|python|go|rust] [--dry-run]
---

# 构建与修复

增量修复编译和构建错误，支持多种编程语言。

## 使用方法

```
/build-fix                    # 自动检测语言并修复
/build-fix --lang java        # 修复 Java 项目
/build-fix --lang nodejs      # 修复 Node.js 项目
/build-fix --lang python      # 修复 Python 项目
/build-fix --lang go          # 修复 Go 项目
/build-fix --dry-run          # 预览修复但不应用
```

## 与 /java-build 的区别

| 指令 | 用途 | 推荐场景 |
|------|------|----------|
| `/build-fix` | 通用构建修复 | 多语言项目、快速修复 |
| `/java-build` | Java 深度修复 | Spring Boot 项目、复杂依赖问题 |

## 修复流程

## 语言检测与构建命令

### 自动检测

```bash
# 按优先级检测构建文件
if [ -f "pom.xml" ] || [ -f "build.gradle" ]; then
    echo "java"
elif [ -f "package.json" ]; then
    echo "nodejs"
elif [ -f "requirements.txt" ] || [ -f "pyproject.toml" ]; then
    echo "python"
elif [ -f "go.mod" ]; then
    echo "go"
elif [ -f "Cargo.toml" ]; then
    echo "rust"
fi
```

### Java 构建命令

```bash
# Maven
mvn clean compile
mvn clean package

# Gradle
gradle build
gradle compileJava
```

### Node.js 构建命令

```bash
# npm
npm install
npm run build
npm test

# pnpm
pnpm install
pnpm build
pnpm test

# yarn
yarn install
yarn build
yarn test
```

### Python 构建命令

```bash
# pip
pip install -r requirements.txt
pip install -e .
python -m pytest

# poetry
poetry install
poetry build
poetry run pytest
```

### Go 构建命令

```bash
go mod download
go build ./...
go test ./...
```

### Rust 构建命令

```bash
cargo build
cargo test
cargo clippy
```

## 常见 Java 编译错误

| 错误 | 典型修复 |
|------|----------|
| `cannot find symbol` | 新增 import 或修正类名拼写 |
| `package X does not exist` | 新增 Maven 依赖或修正包名 |
| `incompatible types` | 类型转换或修正泛型声明 |
| `missing return statement` | 新增 return 语句 |
| `method X in class Y cannot be applied` | 修正方法参数类型或数量 |
| `cyclic inheritance` | 重构类继承关系 |
| `IOException, FileNotFoundException` | 新增 throws 声明或 try-catch |

## 构建命令

```bash
# Maven 构建（跨平台通用）
mvn clean compile
mvn clean package
mvn clean install

# Gradle 构建（跨平台通用）
gradle build
gradle compileJava
gradle bootJar

# 跳过测试构建（跨平台通用）
mvn clean package -DskipTests
```

为了安全，一次修复一个错误！

## 相关指令

- `/java-build` - **Java 深度修复**（推荐用于 Spring Boot + Maven 依赖问题）
- `/code-review` - 修复后审查代码
- `/verify` - 执行完整验证循环

## 使用建议

**使用 /build-fix 当：**
- 需要快速修复编译错误
- 项目使用多种语言
- 错误简单明确

**使用 /java-build 当：**
- Java/Spring Boot 项目
- 有复杂 Maven 依赖冲突
- 需要深度分析和增量修复
