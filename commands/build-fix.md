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

1. **检测项目类型**：自动识别构建工具和编程语言
2. **执行构建**：运行对应的构建命令获取错误信息
3. **分析错误**：按严重性和类型分组
4. **增量修复**：每次修复一个错误后重新验证
5. **生成报告**：汇总修复结果和剩余问题

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
